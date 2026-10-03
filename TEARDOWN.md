# Teardown Record — TradeCore Project 2

> **The AWS environment described in this repository was fully destroyed on 2026-10-03.**
> This repo is now a **template**: every Terraform module, workflow and document below
> is retained so anyone can study it or rebuild the stack from scratch.
> To rebuild, follow the [Deployment Guide](README.md#deployment-guide).
> To destroy it yourself, follow the runbook below.

| | |
|---|---|
| **Date of teardown** | 2026-10-03 |
| **AWS account** | `987292390922` |
| **Primary region** | `af-south-1` (secondary: `us-east-1`) |
| **Profile used** | `ENOFE` (SSO, AdministratorAccess) |
| **State backend** | `s3://tradecore-production-tfstate/tradecore/production/terraform.tfstate` (destroyed) |
| **Terraform** | v1.16.0 |
| **Result** | 81/81 targeted resources destroyed, 0 resources remaining in either region |

---

## 1. What was destroyed

### Terraform-managed (90 state entries → 81 deletions + 6 state-infra + 3 data sources)

| Module | Count | Resources |
|---|---|---|
| `module.networking` | 18 | VPC, 4 subnets, 3 route tables, IGW, route, flow log + role/policy, log group |
| `module.ecs` | 13 | cluster, service, task definition, 2 IAM roles, task-exec policy + attachment, SG + 5 SG rules, log group |
| `module.observability` | 12 | CloudTrail, KMS key + alias, 5 CloudWatch alarms, SNS topic + subscription, budget |
| `module.secrets_manager` | 10 | 5 secrets + 5 versions (`db-host`, `db-name`, `db-password`, `db-user`, `jwt-secret`) |
| `module.rds` | 6 | DB instance, subnet group, monitoring role + attachment, SG + rule |
| `module.logs` | 6 | Log bucket (+ sub-resources), `data.aws_caller_identity` |
| `module.s3` | 5 | App-data bucket + sub-resources |
| `module.alb` | 5 | ALB, HTTP/HTTPS listeners, target group, SG |
| `module.iam` | 4 | GitHub OIDC provider, Actions role, inline policy, caller identity |
| `module.cognito` | 3 | User pool, client, domain |
| `module.ecr` | 2 | Repository + lifecycle policy |
| `module.state` | 6 | State bucket, bucket policy/PAB/encryption/versioning, DynamoDB lock table |

### Outside Terraform (deleted manually)

| Resource | Why it wasn't in state |
|---|---|
| Amplify app `dpqtxdawh7h1c` | Console-managed; TF only tracked ID/domain as variables |
| SNS topic `tradeCore-budget-alert` | Untagged, created outside TF |
| Chatbot config `expadox-lab-project2-alert` + role `AWSChatbotRole-expadox-lab-project2-alert` + policy `AWS-Chatbot-NotificationsOnly-Policy-53156481…` + `AWSServiceRoleForAWSChatbot` | Chatbot setup predates the TF config |
| Log groups `/aws/rds/instance/…/postgresql`, `RDSOSMetrics`, `/aws/ecs/containerinsights/…/performance` | Created by RDS/ECS monitoring, never tracked |
| Log group `/aws/vpc/archvault-production` | Belonged to a different project in the same account |
| 9 orphaned ECS task-definition revisions (`tradecore-production:1…20`) | Only the current revision is in state; historical ones are never tracked |
| RDS final snapshot `tradecore-production-final-snapshot` + 9 automated snapshots | Snapshots are intentionally not tracked |
| 9,669 versions in the logs bucket, 295 in the state bucket | Versioned objects are invisible to `force_destroy = false` |
| 10 ECR images | `aws_ecr_repository` has no `force_destroy`; a non-empty repo fails deletion |
| Default VPCs in `af-south-1` (`vpc-0197d7dbcbe09e72a`) and `us-east-1` (`vpc-0d536c5c6b8ff2597`) | AWS creates these automatically; never part of the project |

---

## 2. Runbook (how the teardown was performed)

Order matters: **0 → 1 → 2 → 3 → 4 → 5**. Skipping Step 2 makes Step 3 fail on the database.

### Step 0 — Stop CI first
A concurrently running workflow takes the S3 lockfile and blocks the destroy.

```bash
gh workflow disable drift.yml   -R <owner>/tradecore-project2
gh workflow disable terraform.yml -R <owner>/tradecore-project2
```

### Step 1 — Capture before anything is destroyed
```bash
cd terraform
export TF_VAR_environment=production TF_VAR_container_image=placeholder \
       TF_VAR_db_name=tradecore TF_VAR_db_username=tradecoreDB \
       TF_VAR_db_password=placeholder TF_VAR_jwt_secret=placeholder \
       TF_VAR_github_org_id=<id> TF_VAR_github_repo_id=<id>
terraform state pull > state-backup.json      # local copy of remote state
terraform output -json > outputs.json
terraform state list
terraform plan -destroy -out=destroy.tfplan <12 × -target>   # review first
```

**Why `-target`?** `terraform/state/main.tf` sets `prevent_destroy = true` on the state
bucket, which aborts a plain `terraform destroy` **at plan time**, before it touches
anything. Targeting all modules *except* `module.state` sidesteps it without editing code.

```bash
terraform plan -destroy -out=destroy.tfplan \
  -target=module.networking -target=module.ecr -target=module.alb -target=module.ecs \
  -target=module.s3 -target=module.rds -target=module.secrets_manager \
  -target=module.cognito -target=module.acm -target=module.iam \
  -target=module.logs -target=module.observability
```

### Step 2 — Clear the two hard blockers (AWS CLI, no code edits)
```bash
# Database/main.tf hardcodes deletion_protection = true → destroy would fail
aws rds modify-db-instance --db-instance-identifier tradecore-production-rds \
  --no-deletion-protection --apply-immediately --region af-south-1

# ECR refuses to delete a non-empty repository
aws ecr describe-images --repository-name tradecore-api --region af-south-1 \
  --query 'imageDetails[].imageDigest' --output text \
| tr '\t' ' ' | xargs -n1 -I{} echo imageDigest={} \
| xargs aws ecr batch-delete-image --repository-name tradecore-api --region af-south-1 --image-ids
```

### Step 3 — Destroy, purge, repeat
```bash
terraform apply destroy.tfplan      # expect: BucketNotEmpty on tradecore-production-logs
```
`S3/main.tf` and `terraform/logs/main.tf` both set `force_destroy = false` **and** both
buckets have versioning enabled, so Terraform cannot delete them while any version exists.
Purge every version, then re-run until clean:

```bash
B=tradecore-production-logs
while :; do
  aws s3api list-object-versions --bucket "$B" --no-paginate --output json > vers.json
  python3 -c "
import json; d=json.load(open('vers.json'))
o=[{'Key':v['Key'],'VersionId':v['VersionId']} for v in d.get('Versions') or []]
o+=[{'Key':v['Key'],'VersionId':v['VersionId']} for v in d.get('DeleteMarkers') or []]
json.dump({'Objects':o}, open('del.json','w')); print(len(o))"   # → 0 breaks the loop
  aws s3api delete-objects --bucket "$B" --delete file://del.json
done
terraform destroy <same 12 -targets> -auto-approve    # → "Destroy complete!"
```

### Step 4 — Everything Terraform does not manage
```bash
# final snapshot (skip_final_snapshot = false creates it during Step 3)
aws rds delete-db-snapshot --db-snapshot-identifier tradecore-production-final-snapshot
# chatbot chain — order matters, deleting the SLR conflicts while config exists
aws chatbot delete-slack-channel-configuration --region us-east-2 \
  --chat-configuration-arn arn:aws:chatbot::987292390922:chat-configuration/slack-channel/expadox-lab-project2-alert
aws iam detach-role-policy --role-name AWSChatbotRole-expadox-lab-project2-alert --policy-arn <arn>
aws iam delete-role       --role-name AWSChatbotRole-expadox-lab-project2-alert
aws iam delete-policy     --policy-arn <chatbot-policy-arn>
aws iam delete-service-linked-role --role-name AWSServiceRoleForAWSChatbot
aws sns delete-topic --topic-arn arn:aws:sns:af-south-1:987292390922:tradeCore-budget-alert
aws logs delete-log-group --log-group-name <each leftover>
aws amplify delete-app --app-id dpqtxdawh7h1c --region us-east-1
# default VPC: subnets → detach+delete IGW → non-main route tables → delete-vpc
```

### Step 5 — State teardown (the last Terraform command you can run)
`outputs.tf` reads `module.state.bucket_name`, so no `terraform plan`/`output` may run after this.

```bash
terraform state rm module.state        # 6 instances removed
# purge tradecore-production-tfstate (versions + delete markers + .tflock), then:
aws s3api delete-bucket --bucket tradecore-production-tfstate --region af-south-1
aws dynamodb delete-table --table-name tradecore-production-tflock --region af-south-1
```

### Step 6 — Verify
```bash
aws resourcegroupstaggingapi get-resources --tag-filters Key=ManagedBy,Values=Terraform
# …note: this index is eventually consistent and lags by several minutes.
# Trust the direct APIs: ecs list-clusters, rds describe-db-instances, s3api list-buckets,
# ecr describe-repositories, cognito-idp list-user-pools, iam list-roles, ec2 describe-vpcs.
```

---

## 3. Gotchas encountered

| # | Gotcha | Symptom | Fix |
|---|---|---|---|
| 1 | `prevent_destroy` on the state bucket | `terraform destroy` fails **before planning anything** | Target every module except `module.state` |
| 2 | `deletion_protection = true` on RDS | Destroy fails at the DB instance | `modify-db-instance --no-deletion-protection` first |
| 3 | `force_destroy = false` + versioning | `409 BucketNotEmpty` | Purge all versions/delete-markers, re-run |
| 4 | ECR has no `force_destroy` attribute | `RepositoryNotEmptyException` | `batch-delete-image` first |
| 5 | `skip_final_snapshot = false` | A snapshot survives the destroy, unmanaged | Delete it deliberately (or keep it as a backup) |
| 6 | Only the **current** task definition is in state | Up to 20 registered revisions survive | `ecs deregister-task-definition` on each |
| 7 | Secrets use the provider's default 30-day recovery window | Secrets linger as `Scheduled for deletion` | Purge immediately with `delete-secret --force-delete-without-recovery` (used here — all 5 verified `ResourceNotFoundException` straight after) |
| 8 | KMS minimum deletion window is **7 days** | Key shows `PendingDeletion` | Nothing — that is the fastest AWS allows |
| 9 | The tagging API lags deletions by minutes | Sweep still reports destroyed resources | Verify with the direct service APIs |
| 10 | State bucket cannot be deleted by the run that uses it | Backend dies mid-run | `terraform state rm module.state` last, then delete manually |

---

## 4. Known incident: an account rename silently broke OIDC

**Symptom** (GitHub Actions, 2026-10-03):

```
Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

**Cause:** the GitHub account was renamed `IsiakaOladayo` → `Isiaka-Ismail`. GitHub builds the
OIDC `sub` claim from the **current** owner/repository name, while the IAM trust policy
(`terraform/iam/main.tf`, driven by `github_org` in `terraform/variables.tf`) still listed the
old name:

```
repo:IsiakaOladayo/tradecore-project2:*          ← stale owner name
repo:IsiakaOladayo@103737461/tradecore-project2@1355257673:*   ← IDs survive renames, names don't
```

**Diagnosis recipe for any "Not authorized to perform sts:AssumeRoleWithWebIdentity":**
1. `aws iam get-role --role-name <role> --query 'Role.AssumeRolePolicyDocument'` → read the `sub` list.
2. Compare each entry against the repo's current `owner/name` (`gh api repos/<owner>/<repo>`).
3. Rename, transfer or fork of a repository invalidates name-based subjects — prefer the
   `owner@repo_id/repo@id` form where GitHub supplies it, and update on every rename.

---

## 5. What intentionally survives

| Item | Reason |
|---|---|
| ECS cluster `tradecore-production` (`INACTIVE`) | ECS sets deleted clusters to `INACTIVE` and keeps the record; `list-clusters` shows nothing, only `describe-clusters` does |
| ECS service `tradecore-api-production` (`INACTIVE`) | Same tombstone behaviour |
| 20 ECS task-definition revisions (`tradecore-production:1…20`, `INACTIVE`) | Deregistration is the only operation AWS exposes — there is **no** task-definition delete API |
| AWS-managed KMS keys (RDS/ACM/SNS/EBS/Secrets Manager defaults) | AWS does not permit deleting them |
| Customer KMS key `922d7999-…` | `PendingDeletion`, physically destroyed 2026-10-10 (the 7-day minimum window) |
| Default VPCs in the other ~28 regions | Out of scope; only the two project regions were cleared |
| `AWSServiceRoleFor*` for Auto Scaling/ECS/ELB/Organizations/RDS/SSO/… | AWS service-linked roles for other services |

All of the above are inert and cost nothing (ECS `INACTIVE` records are free, the scheduled
key is already inaccessible). **Secrets were force-purged on teardown**, so none remain —
not even in a scheduled state.

**Everything else is gone from `af-south-1` and `us-east-1`** — verified against the direct
service APIs: `0` S3 buckets, ECR repos, ECS clusters/services/task definitions (active), RDS
instances/snapshots, Cognito pools, CloudTrail trails, alarms, SNS topics, budgets, log groups,
Amplify apps, VPCs, flow logs, volumes, EIPs, AMIs, Lambda functions, API Gateways, hosted
zones, CloudFormation stacks, IAM roles/policies/OIDC providers — plus `0` Terraform-tagged
resources in every other region.

> **Note on the tagging sweep:** `resourcegroupstaggingapi` kept reporting 21 entries in
> `af-south-1` for many minutes after deletion. Those are the ECS `INACTIVE` tombstones and
> the scheduled secrets above, plus a handful of lagging entries (Cognito pool, flow log) that
> the direct APIs confirm no longer exist. When verifying a teardown, trust the per-service
> APIs — the tagging index is eventually consistent.

---

## 6. Rebuilding

Follow [README → Deployment Guide](README.md#deployment-guide): bootstrap `module.state`,
migrate the backend, set the GitHub secrets, then `terraform plan` / `terraform apply`.
Estimated steady-state cost: **~$30–50/month** (see [Cost Model](README.md#cost-model)).

First step, before anything else: a local `terraform/.terraform/` still references the
destroyed state bucket, so clear it (`rm -rf .terraform && terraform init -backend=false`) —
the guide opens with that note.

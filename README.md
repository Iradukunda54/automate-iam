# Automating Creation of IAM Resources

CloudFormation template + Git sync setup that provisions a one-time console
password (Secrets Manager), an EC2 permissions group, an S3 permissions
group, and three IAM console users.

## What it creates

| Resource | Details |
|---|---|
| `iam-lab/onetime-password` (Secrets Manager) | Auto-generated 16-char password, shared by all three users |
| `ec2-list-create-group` (IAM Group) | `ec2:Describe*` + `ec2:RunInstances` |
| `s3-list-buckets-group` (IAM Group) | `s3:ListAllMyBuckets` + `s3:GetBucketLocation` |
| `ec2-user1` (IAM User) | Member of `ec2-list-create-group`. Can list and launch EC2 instances. |
| `ec2-user2` (IAM User) | Member of `ec2-list-create-group`, **plus** an inline `Deny` on `ec2:RunInstances`. Can list instances but cannot launch one — explicit Deny always wins over the group's Allow. |
| `s3-user` (IAM User) | Member of `s3-list-buckets-group`. Can list buckets, no EC2 access. |

All three users get AWS Console access via `LoginProfile`, with
`PasswordResetRequired: true` so they must set their own password on first
sign-in. The password itself is never stored in the template or in git —
it's pulled in at deploy time with a CloudFormation dynamic reference:

```yaml
Password: !Sub '{{resolve:secretsmanager:${OneTimePasswordSecret}:SecretString:password}}'
```

Retrieve the generated password after deployment:

```bash
aws secretsmanager get-secret-value --secret-id iam-lab/onetime-password --query SecretString --output text
```

## Repo layout

```
templates/
  iam-lab.yaml          CloudFormation template (the actual resources)
  iam-lab-deploy.yaml    Git sync stack deployment file
```

## Option A — Deploy directly with the AWS CLI (quick manual test)

```bash
aws cloudformation deploy \
  --stack-name iam-automation-lab \
  --template-file templates/iam-lab.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region eu-west-1
```

## Option B — CloudFormation Git sync (required for the lab's GitSync rubric)

Git sync links a CloudFormation stack directly to a branch in this repo:
every push to that branch is proposed as a change (via pull request) and,
once merged, CloudFormation deploys it automatically. No manual `deploy`
calls needed after the initial setup.

**One-time setup (must be done interactively in the AWS Console — it
involves authorizing AWS's GitHub App via OAuth, which can't be scripted):**

1. **Create the connection**: CloudFormation console → **Stacks** →
   **Create stack** → **With new resources (standard)**.
2. Under **Prerequisite - Prepare template**, choose **Sync from Git**.
3. **Stack name**: `iam-automation-lab`.
4. **Stack deployment file**: choose **I am providing my own file in my
   repository** → path `templates/iam-lab-deploy.yaml` (already committed
   in this repo).
5. **Template definition repository** → **Link a Git repository** →
   provider **GitHub** → **Connection**: click **add a new connection**,
   which opens the CodeConnections console. Authorize the AWS Connector for
   GitHub app and scope it to this repo only (least privilege — don't grant
   it access to every repo in your account).
6. Select this repo and the branch to monitor (e.g. `main`).
7. **Template file path**: `templates/iam-lab.yaml`.
8. **IAM role**: choose **New IAM role** so CloudFormation scopes the
   deployment role's permissions to exactly what this template needs,
   rather than reusing a broad existing role.
9. Leave **Enable comment on pull request** on, so template changes get a
   diff summary posted to the PR before anyone merges.
10. Submit. CloudFormation opens a pull request in this repo — review and
    merge it to create the stack for the first time.

**After setup**: any future edit to `templates/iam-lab.yaml` or
`templates/iam-lab-deploy.yaml`, committed to the monitored branch, opens a
PR with a change summary; merging it updates the live stack automatically.

### GitSync security notes
- The CodeConnections GitHub connection is scoped to this single repo, not
  your whole GitHub account.
- The auto-generated deployment IAM role is least-privilege (scoped to this
  stack), not a shared admin role.
- Pull-request review is the deployment gate — nothing reaches AWS without
  a merge to the monitored branch.
- The one-time password never appears in git history; only the Secrets
  Manager ARN/dynamic reference does.

## Task 2 — verify each user's access

This part is manual (needs a real browser session against the AWS Console)
and isn't something that can be scripted from here. For each of the three
users:

1. Sign in at the account's IAM sign-in URL
   (`https://<account-id>.signin.aws.amazon.com/console`) with the
   username and the shared one-time password from Secrets Manager. You'll
   be forced to set a new password immediately.
2. Go to the **S3** console and try to list buckets.
   - `s3-user`: succeeds → screenshot.
   - `ec2-user1` / `ec2-user2`: fails with an access-denied banner (no S3
     permissions at all) → screenshot.
3. Go to the **EC2** console, **Instances** page.
   - All three: listing/describing instances succeeds → screenshot.
   - Try **Launch instance**: `ec2-user1` succeeds; `ec2-user2` fails with
     an explicit `UnauthorizedOperation` on `ec2:RunInstances`; `s3-user`
     fails (no EC2 permissions at all) → screenshot each.

That gives 2 screenshots per user (6 total): one EC2 result, one S3
result, each showing success or the expected denial.

## Cleanup

```bash
aws cloudformation delete-stack --stack-name iam-automation-lab --region eu-west-1
```

(If the stack was created via Git sync, delete it from the CloudFormation
console's **Git sync** tab instead, so the sync configuration is torn down
too — otherwise it error the next time the repo is pushed to.)

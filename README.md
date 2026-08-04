# Automating Creation of IAM Resources

CloudFormation template, tracked in this repo and synced live to AWS via
CloudFormation Git sync, that provisions a one-time console password, an
S3 permissions group, an EC2 permissions group, and three IAM console
users.

## What it creates

| Resource | Details |
|---|---|
| `iam-lab/one-time-password` (Secrets Manager) | Auto-generated 20-char password, shared by all three users |
| `S3-List-Buckets-Policy` (Managed policy) | `s3:ListAllMyBuckets` + `s3:GetBucketLocation` |
| `EC2-List-And-Launch-Policy` (Managed policy) | `ec2:Describe*` (list) + `ec2:RunInstances`, `ec2:CreateTags` (create) |
| `S3-User-Group` (IAM Group) | Attached to `S3-List-Buckets-Policy` |
| `EC2-User-Group` (IAM Group) | Attached to `EC2-List-And-Launch-Policy` |
| `ec2-user1` (IAM User) | Member of `EC2-User-Group`. Can list and launch EC2 instances. |
| `ec2-user2` (IAM User) | Member of `EC2-User-Group`, **plus** an inline `Deny` on `ec2:RunInstances`. Can list instances but cannot launch one — explicit Deny always wins over the group's Allow. |
| `s3-user` (IAM User) | Member of `S3-User-Group`. Can list buckets, no EC2 access. |

All three users get AWS Console access via `LoginProfile`, with
`PasswordResetRequired: true` so they must set their own password on first
sign-in. The password itself is never stored in the template or in git —
it's pulled in at deploy time with a CloudFormation dynamic reference:

```yaml
Password: !Sub '{{resolve:secretsmanager:${OneTimePasswordSecret}:SecretString:password}}'
```

Retrieve the generated password:

```bash
aws secretsmanager get-secret-value --secret-id iam-lab/one-time-password --query SecretString --output text
```

## Repo layout

```
templates/
  iam-lab.yaml           CloudFormation template (the actual resources)
  iam-lab-deploy.yaml    Git sync stack deployment file
```

## Live stack

This template is already deployed as stack `iam-resources-lab` in account
`447558491229` (`eu-west-1`), `UPDATE_COMPLETE`. The file in this repo is
an exact mirror of what's live — this section (and the Git sync setup
below) is what turns that pre-existing stack into one CloudFormation
manages via Git going forward, without recreating or touching any of its
resources.

To deploy this template as a brand-new stack elsewhere:

```bash
aws cloudformation deploy \
  --stack-name iam-resources-lab \
  --template-file templates/iam-lab.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region eu-west-1
```

## CloudFormation Git sync setup

Git sync links a CloudFormation stack directly to a branch in this repo:
every push to that branch is proposed as a change (via pull request) and,
once merged, CloudFormation deploys it automatically — no manual `deploy`
calls needed afterward.

Git sync supports attaching to a stack that **already exists** as long as
it's in one of a set of stable states (`CREATE_COMPLETE`,
`UPDATE_COMPLETE`, `UPDATE_ROLLBACK_COMPLETE`, `IMPORT_COMPLETE`,
`IMPORT_ROLLBACK_COMPLETE`) — `iam-resources-lab` is `UPDATE_COMPLETE`, so
it qualifies. This means the running IAM users/groups/secret aren't
recreated; Git sync just takes over how future changes are deployed.

**One-time setup (interactive — it involves authorizing AWS's GitHub App
via OAuth, which can't be scripted from here):**

1. CloudFormation console → **Stacks** → open `iam-resources-lab`.
2. Open the **Git sync** tab → choose to configure/edit Git sync for this
   stack.
3. **Template definition repository** → **Link a Git repository** →
   provider **GitHub** → **Connection**: click **add a new connection**,
   which opens the CodeConnections console. Authorize the AWS Connector
   for GitHub app and scope it to the `Iradukunda54/automate-iam` repo
   only (least privilege — don't grant it access to every repo in the
   account).
4. Select this repo (`Iradukunda54/automate-iam`) and the branch to
   monitor (`master`).
5. **Template file path**: `templates/iam-lab.yaml`.
6. **Stack deployment file**: choose **I am providing my own file in my
   repository** → path `templates/iam-lab-deploy.yaml` (already committed
   here).
7. **IAM role**: choose **New IAM role** so CloudFormation scopes the
   deployment role's permissions to exactly what this template needs,
   rather than reusing a broad existing role.
8. Leave **Enable comment on pull request** on, so future template edits
   get a diff summary posted to the PR before anyone merges.
9. Submit. CloudFormation opens a pull request in this repo reconciling
   the stack with the repo state — review and merge it to activate Git
   sync.

**After setup**: any future edit to `templates/iam-lab.yaml` or
`templates/iam-lab-deploy.yaml`, committed to `master`, opens a PR with a
change summary; merging it updates the live stack automatically.

### GitSync security notes
- The CodeConnections GitHub connection is scoped to this single repo, not
  the whole GitHub account.
- The auto-generated deployment IAM role is least-privilege (scoped to
  this stack), not a shared admin role.
- Pull-request review is the deployment gate — nothing reaches AWS without
  a merge to `master`.
- The one-time password never appears in git history; only the Secrets
  Manager ARN/dynamic reference does.

## Task 2 — verify each user's access

This part is manual (needs a real browser session against the AWS
Console) and isn't something that can be scripted from here. For each of
the three users:

1. Sign in at the account's IAM sign-in URL
   (`https://447558491229.signin.aws.amazon.com/console`) with the
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
aws cloudformation delete-stack --stack-name iam-resources-lab --region eu-west-1
```

(If Git sync is active, disconnect it from the stack's **Git sync** tab
first, then delete the stack — otherwise Git sync errors on the next push
to the monitored branch.)

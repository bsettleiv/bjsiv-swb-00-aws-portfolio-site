# CLAUDE.md

Context for AI-assisted work on this repo. Read this before making changes.

## What this repo is
Personal portfolio site for Mr. Settle (bsettle.net). Static site, deployed to AWS via GitHub Actions.

## Current status
- Placeholder site is live at https://www.bsettle.net and deploys end to end.
- Deploy pipeline (GitHub Actions + OIDC) is working.
- Next: codify the AWS infrastructure in Terraform (platform repo first, then this repo). See `docs/decisions.md`.
- Site code (Astro) has not been started. Do not write site code unless asked.

## Architecture
- Visitors → Cloudflare DNS (DNS only for `www`) → CloudFront distribution → S3 static website endpoint.
- Apex `bsettle.net` → Cloudflare proxied CNAME + Single Redirect (301) to `https://www.bsettle.net`.
- ACM certificate in us-east-1 covering `bsettle.net` and `www.bsettle.net`, DNS-validated in Cloudflare.
- S3 bucket is public via bucket policy. Known tradeoff vs. private bucket + Origin Access Control.

## Resource names
Convention: `bjsiv-swb-00-<category>-<type>-<project>`, with a `Project` tag on every resource set to the project name.

| Resource | Name / ID |
|---|---|
| S3 bucket | `bjsiv-swb-00-sto-s3-portfolio` |
| CloudFront distribution | ID `EAUPIMCGL44O9`, description `bjsiv-swb-00-net-cdn-portfolio` |
| IAM role (GitHub deploy) | `bjsiv-swb-00-iam-role-gha-deploy` |
| IAM permissions policy | `bjsiv-swb-00-iam-policy-portfolio-gha-deploy` |
| ACM certificate | bsettle.net + www.bsettle.net |

Known gap: the IAM role name lacks `portfolio`. Left as-is; only fix if recreating the role.

## Deploy pipeline
- Workflow: `.github/workflows/deploy.yml`.
- Triggers: push to `main`, and manual dispatch.
- Steps: check out → assume the IAM role via OIDC → `aws s3 sync ./site s3://bjsiv-swb-00-sto-s3-portfolio --delete` → CloudFront invalidation on `/*` for distribution `EAUPIMCGL44O9`.
- Site files live in `site/`. Only that folder is uploaded.

### Gotchas we already hit
- **Use the distribution ID, not the CloudFront domain** (`dXXXX.cloudfront.net`), in `DISTRIBUTION_ID`.
- **Role ARN must be the role's ARN** (`role/...`), copied from the role's summary page. Not the policy's ARN.
- **GitHub OIDC `sub` uses numeric IDs now.** The trust policy must match exactly:
  `repo:bsettleiv@278570317/bjsiv-swb-00-aws-portfolio-site@1423718810:ref:refs/heads/main`
  If the repo is ever deleted and recreated, the IDs change and the trust policy must be updated.
- **Keep the Cloudflare ACM validation CNAMEs in place** for auto-renewal (~13 months). Don't delete them.
- **Keep Cloudflare `www` DNS-only (grey cloud).** Proxying it adds a second cache layer and complicates debugging.

## Working rules for AI changes
- Never push directly to `main`. Create a branch and open a PR.
- Never modify AWS resources directly. Infra changes go through Terraform (once it's in place) and the pipeline.
- Changes under `.github/` and `terraform/` need owner review (see `CODEOWNERS`).
- Don't commit secrets, credentials, or tokens. Account and distribution IDs are fine.
- When a decision changes, update `docs/decisions.md` in the same PR.

## Upcoming
- Platform repo `bjsiv-swb-00-aws-platform`: state bucket (S3, native locking, key per project), GitHub OIDC provider, all IAM roles/policies, shared modules and reusable workflows.
- This repo: Terraform for the S3 bucket, bucket policy, CloudFront distribution, and ACM cert, imported rather than recreated. No IAM here.
- Astro site: not started. Design is dark, fluid, and cohesive, with no bright or crowded palettes.
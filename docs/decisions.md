# Decisions

A running log of technical decisions for the portfolio site and why they were made. Add an entry in the same PR as the change it describes.

---

## 2026-10: Hosting on S3 + CloudFront, not Amplify or Cloudflare Pages
**Decision:** Host the static site on S3 behind CloudFront.
**Why:** The project is meant to showcase cloud and infrastructure skills, so the AWS setup is part of the portfolio. Amplify would be simpler but hides the pieces. Cloudflare Pages is simpler still but isn't AWS.
**Tradeoff:** More setup and cost than Amplify, but still small (pennies per month at portfolio traffic).

## 2026-10: Public S3 bucket via website endpoint
**Decision:** Use the S3 static website endpoint as the CloudFront origin, with a public read bucket policy.
**Why:** The website endpoint serves `index.html` for folder paths (e.g., `/about/`) without extra config. Astro generates those paths.
**Tradeoff:** The bucket is publicly readable and CloudFront talks to it over HTTP. The recommended alternative is a private bucket with Origin Access Control. Be ready to explain this tradeoff.

## 2026-10: Cloudflare for DNS, apex redirect via Single Redirect
**Decision:** DNS stays on Cloudflare. `www` is the primary host, and the apex redirects to it with a 301.
**Why:** The domain was already on Cloudflare. A Single Redirect handles the apex without any AWS resources.

## 2026-10: ACM certificate via DNS validation in Cloudflare
**Decision:** Request the certificate in us-east-1 and validate using CNAME records in Cloudflare.
**Why:** CloudFront requires certificates in us-east-1. DNS validation supports automatic renewal, as long as the validation records stay in place.

## 2026-10: GitHub Actions deploys with OIDC, not stored AWS keys
**Decision:** The deploy workflow assumes an IAM role through GitHub's OIDC provider. No long-lived access keys are stored in GitHub.
**Why:** Short-lived credentials scoped to one repo and branch. Nothing to rotate or leak.
**Note:** GitHub's `sub` claim uses numeric owner and repo IDs. The trust policy must match that format, or the role won't assume.

## 2026-10: Console first, then Terraform
**Decision:** Build the infrastructure in the console first to understand each resource, then codify it in Terraform.
**Why:** Learning each piece before automating it. Terraform rebuild becomes a demonstration of the code, not a guess.

## 2026-10: AI-assisted changes go through PRs with a gated deploy
**Decision:** Claude Code on the web makes changes on a branch and opens a PR. Branch protection on `main` requires review before merge. Merging triggers the deploy. The AI has no AWS credentials.
**Why:** Keeps a reviewable history and an approval gate. Credentials stay scoped to the pipeline.

## 2026-10: Deploy role and policy naming
**Decision:** Role `bjsiv-swb-00-iam-role-gha-deploy`, policy `bjsiv-swb-00-iam-policy-portfolio-gha-deploy`.
**Why:** Follows the naming convention, with one exception. The role name omits `portfolio`. Fixing it means recreating the role, which wasn't worth the churn.
## 2026-10: Shared platform repo plus per-project Terraform
**Decision:** Account-wide infrastructure lives in a separate repo, `bjsiv-swb-00-aws-platform`. Each project repo keeps its own resources in a `terraform/` folder.
**Why:** More projects are planned. Some resources exist once per account and are shared by every project: the GitHub OIDC provider and the Terraform state bucket. Keeping those in one place avoids duplicates and keeps project repos small and repeatable.
**Split:**
- Platform repo: state bucket, GitHub OIDC provider, IAM roles and policies for every project, shared modules, reusable GitHub Actions workflows.
- Project repo: the project's own resources (for this repo: S3 bucket, bucket policy, CloudFront distribution, ACM certificate). It references role and provider ARNs, but never creates them.

## 2026-10: IAM roles are owned by the platform repo
**Decision:** Project repos never create or change IAM. The platform repo defines each project's deploy role and Terraform role.
**Why:** A repo that can write its own IAM can give itself any permission. Centralizing IAM keeps each project limited to what the platform grants it.
**Roles per project:** a deploy role (S3 sync and CloudFront invalidation only) and a separate Terraform role. The Terraform role is scoped to the project's resources and only trusts a protected GitHub `environment` that requires approval, not just `main`.

## 2026-10: Single AWS account, isolation by naming and tags
**Decision:** All projects share one AWS account. IAM policies are scoped to each project's resources by name (`bjsiv-swb-00-*-<project>`) and by the `Project` tag.
**Why:** Simple and cheap at personal-project scale.
**Tradeoff:** Isolation is weaker than separate accounts under AWS Organizations, and billing isn't separated by account. Revisit if a project needs hard isolation. Moving to multiple accounts later means rebuilding the platform layer.

## 2026-10: Remote Terraform state in S3 with native locking
**Decision:** One S3 state bucket, created by the platform repo, with one key per project (e.g., `portfolio/terraform.tfstate`). Locking uses S3's native lockfile (`use_lockfile = true`), with no DynamoDB table.
**Why:** CI needs shared state. One bucket keeps it in one place, and a key per project keeps each project's state separate.
**Bootstrap:** The platform repo creates the state bucket using local state, then migrates its own state into the bucket.

## 2026-10: Extract shared modules on the second project, not the first
**Decision:** The portfolio's Terraform starts as plain resources. A shared `static-site` module (S3 + CloudFront + ACM) gets pulled out once a second project needs it, and projects pin it by git tag.
**Why:** A module designed around one project almost always ends up with the wrong inputs. The second project shows what actually varies.

## 2026-10: Import existing resources, don't recreate them
**Decision:** Bring the console-built resources under Terraform with `import` blocks.
**Why:** The live site stays up, and the distribution ID and bucket name don't change.

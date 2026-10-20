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
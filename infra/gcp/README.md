# GCP Bootstrap Inputs

Use this file to pin environment-specific values before provisioning.

## Required values

- `PROJECT_ID`: Google Cloud project ID for the demo
- `PROJECT_NAME`: display name for the project
- `BILLING_ACCOUNT`: billing account ID to link to the project
- `REGION`: GKE region, for example `us-central1`
- `CLUSTER_NAME`: for example `gitops-promoter-demo`
- `BASE_DOMAIN`: `gitops-promoter.dev`

## DNS model

- DNS stays with your DNS provider.
- Create/maintain A records in your DNS provider for:
  - `demo.gitops-promoter.dev`
  - `promoter-webhook.gitops-promoter.dev`
  - `grafana.gitops-promoter.dev`
- Point them at the ingress load balancer IP after bootstrap.

## IAM and API prerequisites

Enable APIs at minimum:

- Cloud Resource Manager API
- Cloud Billing API
- Kubernetes Engine API
- Compute Engine API
- IAM API
- Service Account Credentials API

## Bootstrap checks

Before first `terraform apply`, verify:

1. `gcloud auth list` shows your account.
2. `gcloud beta billing accounts list` returns the target billing account.
3. Your account can create projects and attach billing.
4. `gitops-promoter.dev` DNS records can be edited in your DNS provider.

## Notes

- Terraform files live in `infra/gcp/terraform`.
- Start with regional GKE for higher availability, then downscale later if needed.

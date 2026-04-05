# GitOps Promoter Demo on GCP

This repository bootstraps a public demo environment for GitOps Promoter on Google Cloud.

It uses:

- **GKE** for Kubernetes
- **Terraform** for cluster/network provisioning
- **Argo CD** with an App-of-Apps pattern
- **Single-source Argo CD Applications** only
- **Umbrella Helm charts** stored in this repository
- **Your DNS provider** for public DNS
- **cert-manager** + **ingress-nginx** for HTTPS
- **Sealed Secrets** for secret delivery

This README explains how to set the environment up from scratch in your own GCP account.

## What this repository bootstraps

The initial bootstrap commit is focused on cluster foundations:

- Argo CD
- cert-manager
- ingress-nginx
- Sealed Secrets
- GitOps Promoter
- base namespaces and RBAC
- an ACME `ClusterIssuer`

Later commits can add:

- GitHub App credentials
- GitOps Promoter `ScmProvider`, `GitRepository`, and `PromotionStrategy` resources
- demo workload repository wiring
- webhook secrets
- monitoring and dashboards

## Repository layout

- `apps/`: Argo CD `Application` objects
- `charts/`: umbrella charts and Helm values
- `manifests/`: raw Kubernetes manifests applied by Argo CD
- `promoter-config/`: GitOps Promoter CRs
- `infra/gcp/terraform/`: Terraform for GCP networking and GKE
- `infra/gcp/check-prereqs.sh`: local environment check script
- `docs/`: architecture notes

## Conventions

- Argo CD applications use **single-source** `spec.source`, not multi-source apps.
- Helm deployments come from **in-repo umbrella charts**.
- Environment-specific values should live in ignored local files such as `terraform.tfvars`, not in committed source.

## Prerequisites

Install these tools before starting:

- `git`
- `gcloud`
- `kubectl`
- `terraform`
- `helm`

You also need:

- a GCP account with billing enabled
- a GCP project for the demo
- a domain you can manage in your DNS provider
- a GitHub repository to host this repo

## 1. Clone the repository

```bash
git clone https://github.com/<your-github-owner>/gitops-promoter-demo.git
cd gitops-promoter-demo
```

## 2. Customize repository and domain references

Before provisioning, replace the placeholders and personal org references in the repo.

At minimum, update:

- `apps/root-app.yaml`
- `apps/argocd.yaml`
- `apps/cert-manager.yaml`
- `apps/ingress-nginx.yaml`
- `apps/sealed-secrets.yaml`
- `apps/demo-config.yaml`
- `apps/gitops-promoter.yaml`
- `charts/argocd/values.yaml`
- `charts/gitops-promoter/values.yaml`
- `promoter-config/git-repository.yaml`
- `promoter-config/scm-provider.yaml`

Things you will almost certainly change:

- GitHub owner/repository URLs
- Argo CD RBAC group mapping
- public hostnames
- GitOps Promoter GitHub owner/repo references
- secret names once you introduce real credentials

If you are using your own domain instead of `gitops-promoter.dev`, update:

- `demo.<your-domain>`
- `promoter-webhook.<your-domain>`
- `grafana.<your-domain>`

## 3. Authenticate to Google Cloud

Log into GCP and set up Application Default Credentials for Terraform.

```bash
gcloud auth login
gcloud auth application-default login
gcloud config set project <your-project-id>
gcloud auth application-default set-quota-project <your-project-id>
```

## 4. Prepare Terraform variables

Copy the example file and edit it for your environment.

```bash
cp infra/gcp/terraform/terraform.tfvars.example infra/gcp/terraform/terraform.tfvars
```

Typical fields to edit:

- `create_project`
- `project_id`
- `project_name`
- `billing_account`
- `region`
- `cluster_name`
- `node_machine_type`
- `node_disk_size_gb`
- `node_count_min`
- `node_count_max`

If the GCP project already exists, use:

```hcl
create_project = false
```

If you want Terraform to create the project, use:

```hcl
create_project = true
```

## 5. Provision the GKE cluster

```bash
cd infra/gcp/terraform
terraform init
terraform plan -out tfplan
terraform apply tfplan
cd ../../..
```

Terraform creates:

- required GCP APIs
- a custom VPC
- a subnet with pod/service secondary ranges
- a regional GKE cluster
- a managed node pool with autoscaling

## 6. Fetch kubeconfig

```bash
gcloud container clusters get-credentials <cluster-name> \
  --region <region> \
  --project <project-id>
```

Verify cluster access:

```bash
kubectl get nodes
```

If your GKE auth plugin is missing, install `gke-gcloud-auth-plugin` and retry.

## 7. Bootstrap Argo CD once

Install Argo CD directly once, then let it self-manage from this repository.

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

The second apply uses server-side apply to avoid CRD annotation size issues on fresh installs.

## 8. Hand off to GitOps

Apply the App-of-Apps root application:

```bash
kubectl apply -f apps/root-app.yaml
```

That causes Argo CD to begin reconciling the applications under `apps/`.

## 9. Access the Argo CD UI

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 --decode && echo
```

Start a local port-forward:

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Then open:

- `https://localhost:8080`

Login with:

- username: `admin`
- password: value from `argocd-initial-admin-secret`

## 10. Create DNS records

After `ingress-nginx` is running, get its public IP:

```bash
kubectl -n ingress-nginx get svc
```

Create A records in your DNS provider for:

- `demo.<your-domain>`
- `promoter-webhook.<your-domain>`
- `grafana.<your-domain>`

Point them at the ingress controller load balancer IP.

## 11. Verify bootstrap components

Check the Argo CD applications:

```bash
kubectl -n argocd get applications.argoproj.io
```

Check pods:

```bash
kubectl get pods -A
```

You should see these namespaces/components coming up:

- `argocd`
- `cert-manager`
- `ingress-nginx`
- `kube-system` / Sealed Secrets
- `gitops-promoter`

## 12. Add GitOps Promoter credentials and config

The bootstrap commit intentionally stops short of creating working GitHub App credentials.

Before promotions can work end-to-end, you still need to:

1. Create a GitHub App for GitOps Promoter.
2. Create the demo config repository.
3. Initialize the environment branches.
4. Seal GitHub App credentials into Kubernetes.
5. Commit the `promoter-config/` resources and any sealed secrets.
6. Re-enable webhook secret configuration if desired.

Relevant files:

- `apps/promoter-config.yaml`
- `promoter-config/scm-provider.yaml`
- `promoter-config/git-repository.yaml`
- `promoter-config/promotion-strategy.yaml`
- `promoter-config/commit-statuses/*`

## 13. Suggested first commits

A clean sequence is:

1. **Bootstrap commit**
   - `apps/`
   - `charts/`
   - `manifests/`
2. **Promoter config + secrets commit**
   - `apps/promoter-config.yaml`
   - `promoter-config/`
   - sealed secret manifests
3. **Monitoring and demo workloads commit**

This keeps the cluster bring-up simple and avoids introducing broken credentials too early.

## Version pins currently used

Bootstrap charts are currently pinned to:

- Argo CD `9.4.17`
- cert-manager `1.20.1`
- GitOps Promoter `0.5.1`
- ingress-nginx `4.15.1`
- sealed-secrets `2.18.4`

Update the dependency versions in the umbrella chart `Chart.yaml` files when you upgrade.

## Notes

- `infra/gcp/terraform/terraform.tfvars` is intentionally ignored and should stay local.
- `.terraform.lock.hcl` is safe to commit and helps keep provider resolution reproducible.
- The root app must point at a repository/branch that already contains the committed bootstrap manifests.
- If Argo CD shows `Unknown` sync status right after bootstrap, give it a moment to refresh after the first push.

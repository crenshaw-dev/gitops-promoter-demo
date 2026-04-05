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
- `charts/`: umbrella charts and Helm values (each chart may include `Chart.lock` from `helm dependency build`)
- `manifests/`: raw Kubernetes manifests applied by Argo CD
- `promoter-config/`: GitOps Promoter CRs
- `infra/gcp/terraform/`: Terraform for GCP networking and GKE
- `infra/gcp/check-prereqs.sh`: local environment check script
- `infra/gcp/get-ingress-lb-ip.sh`: print ingress-nginx load balancer IP for DNS A records
- `docs/`: architecture notes

## Conventions

- Argo CD applications use **single-source** `spec.source`, not multi-source apps.
- Helm deployments come from **in-repo umbrella charts**.
- Environment-specific values should live in ignored local files such as `terraform.tfvars`, not in committed source.
- **GitOps first:** change the cluster by committing to this repository and letting Argo CD sync. Avoid `kubectl apply`, `kubectl patch`, or ad-hoc edits to workloads except in a real break-glass situation (for example, Argo CD cannot reconcile and you need a one-time repair).
- **After any break-glass change:** update the matching manifests or Helm values here and push **before** you consider the incident closed, so the next sync does not fight the cluster or reintroduce the failure.
- **Bootstrap exception:** the very first Argo CD install still uses a one-time `helm template` | `kubectl apply` from `charts/argocd` (see [§7](#7-bootstrap-argo-cd-once)); everything after that should flow from Git.

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

## 6. Fetch kubeconfig and verify `kubectl`

Use the same Google identity that has access to the GKE cluster (the account you use in the Cloud Console).

### 6.1 Sign in with the Google Cloud CLI

If you are not already logged in, or your tokens expired:

```bash
gcloud auth login
```

If you use several Google accounts on one machine, sign in to the one that should touch this cluster (replace the address with yours):

```bash
gcloud auth login <your-google-account>
```

Confirm the account you intend to use:

```bash
gcloud auth list
gcloud config set account <your-google-account>
```

### 6.2 Point `gcloud` at the right project

```bash
gcloud config set project <project-id>
```

To discover cluster name and location (regional clusters use `--region`; zonal clusters use `--zone`):

```bash
gcloud container clusters list --project <project-id>
```

### 6.3 Merge cluster credentials into kubeconfig

For a **regional** cluster:

```bash
gcloud container clusters get-credentials <cluster-name> \
  --region <region> \
  --project <project-id>
```

For a **zonal** cluster, use `--zone <zone>` instead of `--region`.

This updates your kubeconfig and sets the current context to that cluster.

### 6.4 Application Default Credentials (optional but recommended)

If `kubectl` or other tools warn that the quota project on Application Default Credentials does not match your GCP project, align it:

```bash
gcloud auth application-default set-quota-project <project-id>
```

(You may have already run `gcloud auth application-default login` in **section 3** for Terraform; the quota project can still be set separately.)

### 6.5 Verify access

```bash
kubectl config current-context
kubectl get nodes
```

On first use, GKE may install a matching `kubectl` client version automatically. If authentication fails, install the plugin and retry:

```bash
gcloud components install gke-gcloud-auth-plugin
```

## 7. Bootstrap Argo CD once

Install Argo CD once from **this repository’s Helm chart** (same version and values as the `argocd` Application), then hand off to GitOps. Server-side apply avoids CRD `last-applied-configuration` size limits and matches how large manifests are applied safely.

From the repository root, with `helm` and `kubectl` configured for the cluster:

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
cd charts/argocd
helm dependency build
helm template argocd . -f values.yaml --namespace argocd \
  | kubectl apply --server-side --force-conflicts --field-manager=argocd-bootstrap -f -
cd ../..
```

The release name **`argocd`** must match the Argo CD Helm release the `argocd` Application expects so labels and selectors stay consistent.

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

Start a local port-forward to the Service’s **HTTP** port. With TLS terminated at the Ingress, the Argo CD API server runs **plain HTTP** on the pod; forwarding to the Service port named **`https` (443)** still reaches that HTTP listener, so a browser at `https://localhost:…` will try TLS and the connection will fail. Use the **`http`** port (80) and open **`http://`**, not `https://`.

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:http
# equivalent: ... 8080:80
```

Then open:

- `http://localhost:8080`

Login with:

- username: `admin`
- password: value from `argocd-initial-admin-secret`

## 10. Create DNS records

After `ingress-nginx` is running, read the load balancer **IPv4** address (what your A records must target):

```bash
./infra/gcp/get-ingress-lb-ip.sh
```

Equivalent one-liner:

```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'
```

If the script errors, the Service may still show `<pending>` under `EXTERNAL-IP`; wait and retry. Use `kubectl -n ingress-nginx get svc` for full status.

This repository assumes DNS for your domain is **not** in Google Cloud DNS (no managed zone is created by Terraform here). Create A records in **your DNS provider** (registrar, Cloudflare, Route 53, and so on):

- `demo.<your-domain>`
- `promoter-webhook.<your-domain>`
- `grafana.<your-domain>`

Point each hostname at the ingress IP from the script. TTL around 300 seconds is reasonable while validating TLS.

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

**Ingress admission webhook:** the ingress-nginx chart is configured so **cert-manager injects the CA** into `ValidatingWebhookConfiguration/ingress-nginx-admission` (`controller.admissionWebhooks.certManager.enabled`). After sync, `kubectl get validatingwebhookconfiguration ingress-nginx-admission -o jsonpath='{.webhooks[0].clientConfig.caBundle}' | wc -c` should print a **non-zero** length; if it stays empty, new `Ingress` objects can fail API validation.

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

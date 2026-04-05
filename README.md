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

- Argo CD (embedded Dex + GitHub connector in Helm values; OAuth credentials via Sealed Secret)
- cert-manager
- ingress-nginx
- Sealed Secrets
- GitOps Promoter
- base namespaces and RBAC
- an ACME `ClusterIssuer`

Later commits can add:

- **`promoter-config/secrets/github-app-credentials.sealed.yaml`** (GitHub App PEM) and edits to **`promoter-config/scm-provider.yaml`** / **`git-repository.yaml`** (**§12**)
- GitOps Promoter `ScmProvider`, `GitRepository`, and `PromotionStrategy` resources (already in repo; need real IDs and secret)
- a sealed **repository-write** credential under **`manifests/argocd-repo-hydrator/`** so the hydrator can push to this repo (**§8**; for a **public** repo you can skip a separate pull secret), plus **`env/dev`**, **`env/e2e`**, **`env/prd`** and **`env/dev-next`**, **`env/e2e-next`**, **`env/prd-next`** so the hydrator and GitOps Promoter can run
- **`manifests/argocd-github-webhook/argocd-github-webhook.sealed.yaml`** for the Argo CD Git webhook HMAC when using push-to-sync (see **§8**)
- **`manifests/argocd-dex-github/argocd-dex-github.sealed.yaml`** for GitHub OAuth credentials used by embedded Dex (see **§9**)
- monitoring and dashboards

## Repository layout

- `apps/`: Argo CD `Application` objects
- `demo-apps/guestbook/`: minimal in-tree Helm chart (Deployment + Service, **`gcr.io/google-samples/gb-frontend:v5`**). Guestbook **`Application`**s use [**source hydration**](https://argo-cd.readthedocs.io/en/stable/user-guide/source-hydrator/): Argo reads **`demo-apps/guestbook`** on **`HEAD`**, writes rendered YAML under **`hydrated/guestbook-{dev,e2e,prd}`** on each env’s **`env/<env>-next`** branch, and syncs from **`env/<env>`** after GitOps Promoter merges **`env/<env>-next` → `env/<env>`**. Per-env replica counts use **`demo-apps/guestbook/env/{dev,e2e,prd}/values.yaml`**
- `charts/`: umbrella charts and Helm values (each chart may include `Chart.lock` from `helm dependency build`)
- `manifests/`: raw Kubernetes manifests applied by Argo CD (top-level YAML is synced by **`demo-config`** into `gitops-promoter`; **`manifests/argocd-github-webhook/`**, **`manifests/argocd-dex-github/`**, and **`manifests/argocd-repo-hydrator/`** (Sealed **repository-write** Secret for hydrator push when this repo is public) are synced into **`argocd`** by their Applications)
- `promoter-config/`: GitOps Promoter CRs and (under **`promoter-config/secrets/`**) sealed **`Secret`** manifests applied with the same **`promoter-config`** Application into **`gitops-promoter`**. One **`ArgoCDCommitStatus`** (**`commit-statuses/argocd-commit-status.yaml`**) selects all guestbook **`Application`**s via **`app.kubernetes.io/name: guestbook`**; the controller maps each app to an environment from **`spec.sourceHydrator.syncSource.targetBranch`** (hydrated guestbook apps) and emits **`argocd-health`** commit statuses ([docs](https://gitops-promoter.readthedocs.io/en/latest/commit-status-controllers/argocd/)). **`PromotionStrategy`** uses strategy-level **`activeCommitStatuses`** (**`argocd-health`**, **`timer`**) so ordering across envs also uses **`promoter-previous-environment`** ([gating](https://gitops-promoter.readthedocs.io/en/latest/gating-promotions/)).
- `infra/gcp/terraform/`: Terraform for GCP networking and GKE
- `infra/gcp/check-prereqs.sh`: local environment check script
- `infra/gcp/get-ingress-lb-ip.sh`: print ingress-nginx load balancer IP for DNS A records
- `docs/`: architecture notes

## Conventions

- Argo CD applications use **single-source** `spec.source`, not multi-source apps, **except** the guestbook env **`Application`**s, which use **`spec.sourceHydrator`** so GitOps Promoter can gate on hydrated branches ([source hydrator](https://argo-cd.readthedocs.io/en/stable/user-guide/source-hydrator/)).
- Every `Application` under `apps/` uses **automated** sync with **`prune: true`** and **`selfHeal: true`** (including `root-app`).
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
- `kubeseal` (same major line as the in-cluster Sealed Secrets controller) when you create sealed manifests locally

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
- `apps/guestbook-dev.yaml`, `apps/guestbook-e2e.yaml`, `apps/guestbook-prd.yaml`
- `apps/argocd-repo-hydrator.yaml`
- `apps/argocd-github-webhook.yaml`
- `apps/argocd-dex-github.yaml`
- `apps/gitops-promoter.yaml`
- `charts/argocd/values.yaml`
- `charts/gitops-promoter/values.yaml`
- `promoter-config/git-repository.yaml`
- `promoter-config/scm-provider.yaml`

Things you will almost certainly change:

- GitHub owner/repository URLs
- Argo CD RBAC group mapping and Dex **`orgs`** / **`teams`** in **`charts/argocd/values.yaml`** (must match `org:team` in **`policy.csv`**)
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

### Git webhooks (sync soon after you push)

By default Argo CD **polls** Git about every **three minutes**. To refresh as soon as GitHub (or another provider) receives your push, expose the Argo CD API over **HTTPS** and register a **repository webhook** that calls Argo CD’s **`/api/webhook`** endpoint. See the upstream [Git webhook configuration](https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/) guide for full detail and other SCMs (GitLab, Bitbucket, Azure DevOps, Gogs).

**Prerequisites**

- DNS and TLS for your Argo CD hostname work (this demo: **`https://demo.<your-domain>`**).
- GitHub (or your host) can reach that URL from the public internet.

**1. Choose a shared secret**

Generate a random string (example):

```bash
openssl rand -base64 32
```

You will use the same value in GitHub and in the cluster.

**2. Add the webhook in GitHub**

In the GitHub repository that Argo CD reads (your fork of this demo repo, or any repo backing an Application):

1. **Settings** → **Webhooks** → **Add webhook**
2. **Payload URL:** `https://demo.<your-domain>/api/webhook` (use your real Argo CD host; path must be **`/api/webhook`**)
3. **Content type:** **`application/json`** (required; the default form encoding is not supported)
4. **Secret:** paste the value from step 1
5. **Events:** enable **Just the push event** (or restrict to pushes only)

Save the webhook. GitHub may show a failed delivery until the sealed manifest in the next section is synced.

**3. Seal the shared secret and commit it (Sealed Secrets only)**

Helm in this repo sets **`webhook.github.secret`** on **`argocd-secret`** to the indirection string **`$argocd-github-webhook:githubWebhookSecret`**. Argo CD resolves that at runtime from a normal **`Secret`** named **`argocd-github-webhook`** in **`argocd`**, as in the upstream [webhook “Alternative”](https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/#alternative) docs. That **`Secret`** must have label **`app.kubernetes.io/part-of: argocd`** and data key **`githubWebhookSecret`** holding the same value you configured in GitHub.

**Where to put the sealed manifest (this is the only supported path in this repo)**

1. From the **repository root**, run **`kubeseal`** (replace the placeholder secret). The Bitnami chart in **`charts/sealed-secrets`** exposes the controller as Service **`sealed-secrets`** in **`kube-system`**, not the `kubeseal` default **`sealed-secrets-controller`**, so the controller flags are required.

```bash
kubectl create secret generic argocd-github-webhook -n argocd \
  --from-literal=githubWebhookSecret='PASTE_THE_SAME_SECRET_AS_GITHUB' \
  --dry-run=client -o yaml \
  | kubectl label --local --dry-run=client -f - app.kubernetes.io/part-of=argocd -o yaml \
  | kubeseal \
      --controller-name sealed-secrets \
      --controller-namespace kube-system \
      -o yaml -n argocd \
      -w manifests/argocd-github-webhook/argocd-github-webhook.sealed.yaml
```

2. Commit **`manifests/argocd-github-webhook/argocd-github-webhook.sealed.yaml`** and push to the branch **`root-app`** tracks.

The **`Application/argocd-github-webhook`** in **`apps/argocd-github-webhook.yaml`** (picked up by the app-of-apps) syncs **`manifests/argocd-github-webhook/`** into **`argocd`** with **automated** sync and **self-heal**, so the decoded **`Secret`** appears without **`kubectl apply`**. Until you add that file, the app simply manages an empty directory; after you push the sealed manifest, the next sync creates the **`Secret`** and GitHub deliveries can verify.

If you omit the webhook secret entirely, hooks can still trigger a refresh, but Argo CD cannot verify the sender; for a **public** URL, configuring the secret is **strongly recommended** (see the upstream docs).

**4. Verify**

Push a trivial commit to the tracked branch. In the Argo CD UI, the Application should move to **Refreshing** quickly instead of waiting for the poll interval. In GitHub, open the webhook’s **Recent Deliveries** and confirm **`200`** responses.

### Source hydrator credentials (push for in-tree guestbook)

The guestbook **`Application`**s use Argo CD’s [**source hydrator**](https://argo-cd.readthedocs.io/en/stable/user-guide/source-hydrator/): Argo renders the in-tree chart under **`demo-apps/guestbook`**, **pushes** rendered manifests to **`env/<env>-next`** branches in **this** repository, and **syncs** the cluster from **`env/<env>`**. That matches GitOps Promoter’s hard-coded **`*-next`** convention ([getting started](https://gitops-promoter.readthedocs.io/en/latest/getting-started/#promotion-strategy)).

Keeping the chart **in-tree** avoids a second GitHub repo: the hydrator’s dry source and hydrated output both live in **`gitops-promoter-demo`** (Argo does not support hydrating into a different repo yet — [issue #22719](https://github.com/argoproj/argo-cd/issues/22719)).

This demo enables the hydrator in **`charts/argocd/values.yaml`** (**`commitServer.enabled: true`** and **`configs.params.hydrator.enabled: "true"`**). After the next Helm sync of Argo CD, the application controller and API server pick up **`argocd-cmd-params-cm`**.

If this repository is **public**, Argo CD can **clone** the dry chart **without** a **`repository`** credential; you still need a **`repository-write`** **`Secret`** so the hydrator can **push**. Use a [**GitHub App**](https://argo-cd.readthedocs.io/en/stable/user-guide/source-hydrator/#using-the-source-hydrator) with **Contents** read/write—typically the **same** installation as GitOps Promoter’s **`ScmProvider`** (**§12**). The data keys are the usual Argo CD repository fields (**`type`**, **`url`**, **`githubAppID`**, **`githubAppPrivateKey`**, optional **`githubAppInstallationID`**).

If the repository is **private**, add a second **`Secret`** with **`argocd.argoproj.io/secret-type: repository`** for the same URL (see the upstream hydrator doc’s pull/push pair).

From the **repository root**, seal the **write** secret into **`manifests/argocd-repo-hydrator/`** (same **`kubeseal`** controller flags as **§8** Git webhook / **§9** Dex). Replace the App ID, optional installation ID, PEM path, and repo URL.

```bash
kubectl create secret generic argocd-repo-gitops-promoter-write -n argocd \
  --from-literal=type=git \
  --from-literal=url='https://github.com/<your-github-owner>/gitops-promoter-demo' \
  --from-literal=githubAppID='YOUR_GITHUB_APP_ID' \
  --from-file=githubAppPrivateKey=/path/to/your-github-app.private-key.pem \
  --dry-run=client -o yaml \
  | kubectl label --local --dry-run=client -f - argocd.argoproj.io/secret-type=repository-write -o yaml \
  | kubectl label --local --dry-run=client -f - app.kubernetes.io/part-of=argocd -o yaml \
  | kubeseal \
      --controller-name sealed-secrets \
      --controller-namespace kube-system \
      -o yaml -n argocd \
      -w manifests/argocd-repo-hydrator/argocd-repo-gitops-promoter-write.sealed.yaml
```

Commit that sealed file. The **`Application/argocd-repo-hydrator`** (**`apps/argocd-repo-hydrator.yaml`**, sync wave **2**) applies it before the guestbook apps (wave **5**). Until it exists, hydration cannot push and the guestbook applications will not reach a healthy sync.

**Repository settings:** ensure GitHub does **not** auto-delete **`*-next`** branches when PRs merge (Promoter relies on them). Prefer disabling **Automatically delete head branches** or add branch protection for a pattern such as **`env/**`** ([Promoter note](https://gitops-promoter.readthedocs.io/en/latest/getting-started/#github-app-configuration)).

**Not the GitOps Promoter webhook**

This section is only for **Argo CD’s** Git notification endpoint (`/api/webhook` on the Argo CD host). **GitOps Promoter** uses a separate hostname (this demo: **`promoter-webhook.<your-domain>`**) and its own ingress—do not point the Argo CD Git webhook at that URL.

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

### GitHub login (Dex)

Argo CD ships with [Dex](https://dexidp.io/). This repository wires a [GitHub OAuth2 connector](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/#configuring-github-oauth2) in **`charts/argocd/values.yaml`** under **`argo-cd.configs.cm.dex.config`**. OAuth **client ID** and **client secret** are **not** in Git: they live in **`Secret/argocd-dex-github`** (keys **`clientId`** and **`clientSecret`**) with label **`app.kubernetes.io/part-of: argocd`**, delivered like other secrets via Sealed Secrets.

**1. Register a GitHub OAuth App**

In GitHub: **Settings** → **Developer settings** → **OAuth Apps** → **New OAuth application**.

- **Application name:** any label you like (for example `Argo CD demo`)
- **Homepage URL:** your public Argo CD URL (this demo: **`https://demo.<your-domain>`**)
- **Authorization callback URL:** **`https://demo.<your-domain>/api/dex/callback`** (must match the **`url`** in **`argo-cd.configs.cm`** plus **`/api/dex/callback`**)

Under the app, create a **client secret**.

**Org access (required for org-based login):** GitHub will not let Dex read org or team membership until the OAuth app is allowed for that org. For this demo, **`argoproj-labs`** org owners must approve the app under **Organization `argoproj-labs`** → **Settings** → **Third-party access** (or users complete the org grant when authorizing). Without this, Dex logs **`application not authorized to read org data`** and the UI shows **login failed**. See Dex’s [GitHub connector caveats](https://dexidp.io/docs/connectors/github/).

**2. Seal credentials into the repo (exact path)**

From the **repository root** (same **`kubeseal`** controller flags as the Git webhook secret in **§8**):

```bash
kubectl create secret generic argocd-dex-github -n argocd \
  --from-literal=clientId='YOUR_GITHUB_OAUTH_CLIENT_ID' \
  --from-literal=clientSecret='YOUR_GITHUB_OAUTH_CLIENT_SECRET' \
  --dry-run=client -o yaml \
  | kubectl label --local --dry-run=client -f - app.kubernetes.io/part-of=argocd -o yaml \
  | kubeseal \
      --controller-name sealed-secrets \
      --controller-namespace kube-system \
      -o yaml -n argocd \
      -w manifests/argocd-dex-github/argocd-dex-github.sealed.yaml
```

Commit **`manifests/argocd-dex-github/argocd-dex-github.sealed.yaml`** and push. The **`Application/argocd-dex-github`** in **`apps/argocd-dex-github.yaml`** syncs it into **`argocd`** on **sync wave 1**, before the **`argocd`** Application on **wave 2**, so Dex can read the **`Secret`** when **`argocd-cm`** is applied.

Until that **`Secret`** exists, Dex may log errors about missing client credentials; **`admin`** login (above) still works. After a successful sync, use **Log in via GitHub** on the Argo CD sign-in page.

**3. Align org, team claims, and RBAC**

- This demo’s **`dex.config`** matches the shape used in the upstream Argo CD project demo: org **`argoproj-labs`**, team **`gitops-promoter-approvers`**. Only members of that team can finish OAuth; others fail at Dex (often as **login failed**). The OAuth app must be **approved for `argoproj-labs`** so Dex can read org/team data.
- Dex’s default **`teamNameField`** (**`name`**) controls how **`org:team`** appears in **`groups`**. The **`g, …`** line in **`argo-cd.configs.rbac.policy.csv`** must match that claim (often **`argoproj-labs:gitops-promoter-approvers`** when GitHub’s team name matches the slug). If you get **readonly** after login, adjust **`policy.csv`** to the actual group string.

If you fork the repo, update the **`orgs`** / **`teams`** entries, the **`g, org:team, role:admin`** line, and your GitHub OAuth app’s callback URL for your real hostname.

**4. If you still see “login failed”**

1. Confirm the OAuth app’s **Authorization callback URL** is exactly **`https://<same-host-as-argocd-cm-url>/api/dex/callback`** (no trailing slash; **`https`** if users hit the UI over TLS).
2. Confirm **`Secret/argocd-dex-github`** exists in **`argocd`** and keys are **`clientId`** / **`clientSecret`** (camelCase), and the label **`app.kubernetes.io/part-of: argocd`** is present.
3. Read Dex and API server logs after a failed attempt:

```bash
kubectl -n argocd logs deploy/argocd-dex-server --tail=80
kubectl -n argocd logs deploy/argocd-server --tail=80
```

Look for GitHub OAuth errors, **`application not authorized to read org data`**, or unresolved **`$…`** placeholders in config (secret indirection broken).

**5. Optional: disable the local `admin` user**

After GitHub login works, you can turn off the built-in admin account per the [Argo CD FAQ](https://argo-cd.readthedocs.io/en/stable/faq/#how-to-disable-admin-user) (`admin.enabled` in **`argo-cd.configs.cm`**).

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

## 12. GitOps Promoter: GitHub App, sealed credentials, and Git repository

The bootstrap install includes the controller (**`apps/gitops-promoter.yaml`**, wave **3**) and the **`promoter-config`** Application (wave **4**), which syncs **`promoter-config/`** recursively into **`gitops-promoter`**. Until you add a real GitHub App and secret, **`ScmProvider`** and friends will not be able to talk to GitHub.

Official reference: [GitOps Promoter getting started](https://gitops-promoter.readthedocs.io/en/latest/getting-started/) (permissions, `Secret` shape, `ScmProvider` / `GitRepository`).

### 12.1 GitHub App

1. [Create a GitHub App](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app) (org or user).
2. Permissions (from upstream docs): **Checks** read/write, **Contents** read/write, **Pull requests** read/write.
3. **Webhook (recommended):** set **Webhook URL** to **`https://promoter-webhook.<your-domain>/`** (same host as **`charts/gitops-promoter/values.yaml`** → **`webhookReceiver.ingress.hostname`**). Use the payload format GitHub defaults to unless the Promoter docs specify otherwise.
4. **Webhook “secret” on GitHub:** GitHub lets you set a signing secret for repository webhooks, but the GitOps Promoter **webhook receiver does not verify** **`X-Hub-Signature-256`** (or similar) today — see [`internal/webhookreceiver/server.go`](https://github.com/argoproj-labs/gitops-promoter/blob/main/internal/webhookreceiver/server.go) in **v0.25.1** / **main**. Setting a secret in GitHub does not harden this endpoint until upstream adds verification. Mitigations: keep the URL non-obvious, restrict at ingress/network (allowlist GitHub IPs, internal-only URL, or a front proxy that validates signatures), and treat the receiver as **unauthenticated trigger** surface (it only enqueues reconcile when a matching **`ChangeTransferPolicy`** exists).
5. Generate and download a **private key** (`.pem`).
6. Note the app’s **App ID** (numeric). **Install** the **GitHub App** on the user or organization that owns the repositories GitOps Promoter will use via the API (see below). Note **Installation ID** if you want to pin it (optional on `ScmProvider`; see **`promoter-config/scm-provider.yaml`**).

**Which repositories must the installation include?**

- **Required:** every GitHub repository the Promoter mutates via the API — at minimum, whatever you name in **`promoter-config/git-repository.yaml`** (this demo: **`owner/gitops-promoter-demo`** — the same repo as bootstrap). Install the GitHub App on that repo (or use **All repositories** in a sandbox).
- **Argo CD** clones this repo (anonymous if **public**); the **hydrator push** **`Secret`** (**§8**) is still required to write hydrated commits. The **GitHub App** on **`ScmProvider`** is what Promoter uses for API access—often the same installation as the hydrator **write** secret.
- When you click **Install**, choose **Only select repositories** and include the repo from **`git-repository.yaml`** and any other repos your **`PromotionStrategy`** targets. **All repositories** is simpler for sandboxes but broader than necessary.

### 12.2 Repository and branches

1. Use the GitHub repo named in **`promoter-config/git-repository.yaml`** (this demo: **`crenshaw-dev/gitops-promoter-demo`** — change **`owner`** / **`name`** if you fork under a different path).
2. **`promotion-strategy.yaml`** expects merge targets **`env/dev`**, **`env/e2e`**, and **`env/prd`**. Your hydrator / promotion flow must match the [branch conventions](https://gitops-promoter.readthedocs.io/en/latest/getting-started/#promotion-strategy) the project documents (including **`env/<env>-next`** hydration branches). Align branch names here with what you actually create in Git.

### 12.3 Seal the GitHub App private key (same Application as promoter CRs)

The **`ScmProvider`** references **`secretRef.name: github-app-credentials`**. The data key must be **`githubAppPrivateKey`** (PEM string), per the [getting started](https://gitops-promoter.readthedocs.io/en/latest/getting-started/#github-app-configuration) `Secret` example.

From the **repository root**, with the PEM file on disk (adjust paths):

```bash
kubectl create secret generic github-app-credentials -n gitops-promoter \
  --from-file=githubAppPrivateKey=/path/to/your-github-app.private-key.pem \
  --dry-run=client -o yaml \
  | kubeseal \
      --controller-name sealed-secrets \
      --controller-namespace kube-system \
      -o yaml -n gitops-promoter \
      -w promoter-config/secrets/github-app-credentials.sealed.yaml
```

Commit **`promoter-config/secrets/github-app-credentials.sealed.yaml`**. The **`promoter-config`** Application applies it to **`gitops-promoter`** alongside **`ScmProvider`**, **`GitRepository`**, and **`PromotionStrategy`** (no separate Argo `Application`).

### 12.4 Wire non-secret IDs in Git

Edit **`promoter-config/scm-provider.yaml`**: set **`spec.github.appID`** to your numeric App ID. Uncomment and set **`installationID`** only if you need to pin a single installation.

Edit **`promoter-config/git-repository.yaml`** if **`owner`** / **`name`** differ from the GitHub repository that holds **`demo-apps/guestbook`** and the **`env/...`** promotion branches (normally your **`gitops-promoter-demo`** fork).

Push; after sync, **`kubectl -n gitops-promoter get sealedsecret,secret,scmprovider`** should show the decoded **`Secret`** and a healthy **`ScmProvider`**.

### 12.5 Argo CD UI and other secrets

- Argo CD integrations (extension, links, commit-status keys): **`charts/argocd/values.yaml`** and [GitOps Promoter Argo CD integrations](https://gitops-promoter.readthedocs.io/en/latest/argocd-integrations/). If a **`PromotionStrategy`** is not top-level in an Application’s resource tree, set **`promoter.argoproj.io/has-promotionstrategy: "true"`** on that Application so the extension tab appears.
- Argo CD Git webhook HMAC: **§8** → **`manifests/argocd-github-webhook/`**.
- Argo CD Dex GitHub OAuth: **§9** → **`manifests/argocd-dex-github/`**.

### 12.6 Checklist (other prerequisites)

1. Create GitHub App, **install** it with access to the repo in **`git-repository.yaml`** (see **§12.1**); configure webhook host to **`promoter-webhook.<your-domain>`** when ingress is ready.
2. Ensure branches in that repo match **`promotion-strategy.yaml`** (and your hydrator).
3. Seal and commit **`promoter-config/secrets/github-app-credentials.sealed.yaml`**.
4. Set **`appID`** (and optionally **`installationID`**) in **`promoter-config/scm-provider.yaml`**; fix **`git-repository.yaml`** owner/name.
5. Optional: **§8** / **§9** sealed files for Argo CD.

## 13. Suggested first commits

A clean sequence is:

1. **Bootstrap commit**
   - `apps/` (includes **`apps/argocd-repo-hydrator.yaml`**, **`apps/argocd-github-webhook.yaml`**, and **`apps/argocd-dex-github.yaml`**; sealed credential files can land in follow-up commits)
   - `charts/`
   - `manifests/` (including **`manifests/argocd-repo-hydrator/`** once the hydrator **repository-write** secret is sealed)
2. **Promoter config + secrets commit**
   - `apps/promoter-config.yaml`
   - `promoter-config/` (including **`promoter-config/secrets/github-app-credentials.sealed.yaml`** when the GitHub App exists)
   - other sealed secrets as needed (**`manifests/argocd-github-webhook/…`**, **`manifests/argocd-dex-github/…`**)
3. **Demo workloads commit**
   - `demo-apps/guestbook/` and **`apps/guestbook-dev.yaml`**, **`apps/guestbook-e2e.yaml`**, **`apps/guestbook-prd.yaml`**
4. **Monitoring commit** (optional)

This keeps the cluster bring-up simple and avoids introducing broken credentials too early.

## Version pins currently used

Bootstrap charts are currently pinned to:

- Argo CD `9.4.17`
- cert-manager `1.20.1`
- GitOps Promoter `0.5.1` (Helm chart; Argo CD UI extension bundle **`v0.25.1`** per [GitOps Promoter Argo CD integrations](https://gitops-promoter.readthedocs.io/en/latest/argocd-integrations/))
- ingress-nginx `4.15.1`
- sealed-secrets `2.18.4`

Update the dependency versions in the umbrella chart `Chart.yaml` files when you upgrade.

## Notes

- `infra/gcp/terraform/terraform.tfvars` is intentionally ignored and should stay local.
- `.terraform.lock.hcl` is safe to commit and helps keep provider resolution reproducible.
- The root app must point at a repository/branch that already contains the committed bootstrap manifests.
- If Argo CD shows `Unknown` sync status right after bootstrap, give it a moment to refresh after the first push.

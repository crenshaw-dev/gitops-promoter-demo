# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/crenshaw-dev/gitops-promoter-demo
# cd into the cloned directory
git checkout 3b64947279fe29563e0768962e0f2e4ed9c40840
helm template . --name-template guestbook-prd --namespace guestbook-prd --values ./demo-apps/guestbook/env/prd/values.yaml --include-crds
```

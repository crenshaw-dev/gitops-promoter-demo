# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/crenshaw-dev/gitops-promoter-demo
# cd into the cloned directory
git checkout de00fe5ce7d4c0dfcabc51bc594d673120d6ea37
helm template . --name-template guestbook-e2e --namespace guestbook-e2e --values ./demo-apps/guestbook/env/e2e/values.yaml --include-crds
```

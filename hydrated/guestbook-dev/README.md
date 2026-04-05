# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/crenshaw-dev/gitops-promoter-demo
# cd into the cloned directory
git checkout da3f12346652ea8d40d9736e54e814d435cc5fef
helm template . --name-template guestbook-dev --namespace guestbook-dev --values ./demo-apps/guestbook/env/dev/values.yaml --include-crds
```

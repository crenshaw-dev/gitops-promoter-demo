# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/crenshaw-dev/gitops-promoter-demo
# cd into the cloned directory
git checkout 58a149da8ed7246cece2529b1c7f56919b24cebe
helm template . --name-template guestbook-e2e --namespace guestbook-e2e --values ./demo-apps/guestbook/env/e2e/values.yaml --include-crds
```

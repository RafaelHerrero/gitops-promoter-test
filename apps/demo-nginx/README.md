# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/RafaelHerrero/gitops-promoter-test
# cd into the cloned directory
git checkout 92d4eb70b527a06afa33c00d17ef049e49d3cacc
helm template . --name-template demo-nginx-development --namespace demo-nginx-dev --include-crds
```

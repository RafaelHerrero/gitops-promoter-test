# Manifest Hydration

To hydrate the manifests in this repository, run the following commands:

```shell
git clone https://github.com/RafaelHerrero/gitops-promoter-test
# cd into the cloned directory
git checkout c97b18521ce827ce8e0758b2f34d93f8454a46d0
helm template . --name-template demo-nginx-staging --namespace demo-nginx-staging --include-crds
```

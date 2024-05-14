# loculus_deployments

We have a persistent instance running on AWS at https://productiontest.pathoplexus.org/ . 
It connects to a managed database running on Amazon RDS. 
It is configured according to https://github.com/pathoplexus/loculus_deployments/tree/main/deployments/pathoplexus and changes to that repo will change it, via ArgoCD sync at https://argocd.pathoplexus.org

## Config files

### config.json

`config.json` defines the branch and commit sha to use. Update it to update the deployment.

Example:

```json
{
  "branch" : "main",
  "target" : "dccf86fa44482c6a497d11a8857acb231392b42a",
  "number" : 9999,
  "head_short_sha_7": "dccf86f",
"host" : "productiontest.pathoplexus.org",
    "previewName" : "productiontest"
}
```

### values.yaml

The `values.yaml` file is applied _over_ the default `values.yaml`. Example:

```yaml
runDevelopmentMainDatabase: false
bannerMessage: This is an environment for penetration testing
runDevelopmentKeycloakDatabase: false
runDevelopmentMainDatabase: false
defaultOrganisms:
  mpox:
    ingest:
      configFile:
        subsample_fraction: 1
```

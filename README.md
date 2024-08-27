# Gitops repo for Pathoplexus' Loculus deployments

This repo contains gitops data that is read by Argo CD.

It currently defines 3 deployments:

- production
- staging
- main

All 3 deployments are persistent and connect to managed databases running on Amazon RDS.

The configuration is done in https://github.com/pathoplexus/loculus_deployments/tree/main/deploy

Changes to this repo will change deployments via ArgoCD sync at https://argocd.pathoplexus.org

## Config files

### config.json

`config.json` defines the branch and commit sha to use. Update it to update the deployment.

Example:

```json
{
  "branch": "main",
  "target": "dccf86fa44482c6a497d11a8857acb231392b42a",
  "number": 9999,
  "head_short_sha_7": "dccf86f",
  "host": "productiontest.pathoplexus.org",
  "previewName": "productiontest"
}
```

### values.yaml

The `values.yaml` file is applied _over_ the default `values.yaml`. Example:

```yaml
runDevelopmentMainDatabase: false
bannerMessage: This is an environment for penetration testing
runDevelopmentKeycloakDatabase: false
defaultOrganisms:
  mpox:
    ingest:
      configFile:
        subsample_fraction: 1
```

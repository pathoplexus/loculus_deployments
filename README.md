# Gitops repo for Pathoplexus' Loculus deployments

This repo contains gitops data that is read by Argo CD.

It currently defines 4 deployments:

- production (https://pathoplexus.org)
- demo (currently https://demo.pathoplexus.org)
- staging (currently https://staging.pathoplexus.org)
- main (currently https://preview-main.pathoplexus.org)

Production, demo and staging deployments are persistent, connected to managed databases running on Amazon RDS.

Main uses an ephemeral database that is set up fresh on every sync.

The configuration is done in https://github.com/pathoplexus/loculus_deployments/tree/main/deploy

Changes to this repo will change deployments via ArgoCD sync at https://argocd.pathoplexus.org

## Config files

### config.json

`config.json` defines the branch and commit sha to use. Update it to update the deployment.

Example:

```json
{
  "branch": "production",
  "special_environment": "production",
  "host": "pathoplexus.org",
  "head_sha": "9130a5840e1ac7ffe198795ba663fdd545b390c3"
}
```

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

## Staging rollout procedure

<details>
<summary>ssh config to use `ssh bastion`</summary>
```
Host bastion
  HostName XXX.XXX.XXX.XXX
  User ec2-user
  IdentityFile YOUR_PATH_TO_KEY.pem
```
</details>

To rollout to staging, we want to first make staging be identical with prod. Thus, we clone the prod db to staging, then restart the backend on the cloned db:

```sh
ssh bastion "cd pathoplexus/scripts/db-clone/ && ./clone-prod-to-staging.sh"
kubectl rollout restart deployment/loculus-backend -n staging
```

We then make sure that staging really is identical to prod by running the integrity script from `pathoplexus/pathoplexus`:

```sh
cd ~/code/pathoplexus/data-integrity-tests/regression-testing
micromamba activate pp-integrity
snakemake -F
cat  results/*.diff
```

The result is expected to be just empty lines.

We can then rollout a new commit to staging, via PR in this repo:

```sh
gh workflow run bump-staging.yml -R pathoplexus/loculus_deployments
```

Merge it, then check again (after a few minutes) that integrity/regression tests look good (rerun the snipped from above).

Once that's done, you can (after reviewing everything looks healthy and testing new features) deploy to production through PR triggered by:

```sh
gh workflow run promote-staging-to-production.yml -R pathoplexus/loculus_deployments
```

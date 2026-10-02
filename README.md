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

Staging and production are namespaces of one EKS cluster. Pass its context on every `kubectl` call (`--context <eks-context>`, e.g. `ppx-new`): your default context may be a different cluster.

Before merging any PR opened by the workflows below, check its diff: it should change only `head_sha` in `deploy/<env>/config.json`, to the SHA you mean.

```sh
gh pr diff <PR_NUMBER> -R pathoplexus/loculus_deployments
```

### Set staging to prod commit (only required if staging is not on prod commit)

To rollout to staging, we want to first make staging be identical with prod. 

When staging is not on the same commit as prod, first change the commit pointed to by prod to staging. You can use a workflow to create the PR for this:

```sh
gh workflow run set-staging-to-be-same-as-current-production.yaml -R pathoplexus/loculus_deployments
```

To get the PR's number run:

```sh
gh pr list -R pathoplexus/loculus_deployments
```

then merge with:

```sh
gh pr merge -R pathoplexus/loculus_deployments --admin --squash <PR_NUMBER>
```

Run the clone straight after merging, and treat staging as broken until it is done. Staging now runs the old prod code on a database the newer release has already changed. If that release added a Flyway migration, the old backend runs on a newer schema and can fail to read rows written in the new format. If it added preprocessing pipeline versions, the silo-importers loop with `No lineage definition URL configured for pipeline version N`, while the old SILO pods keep serving the previous data. Don't read staging logs or run the regression until the clone and backend restart are done.

### Clone prod db to staging

Thus, we clone the prod db to staging, then restart the backend on the cloned db:

```sh
ssh bastion "cd pathoplexus/scripts/db-clone/ && ./clone-prod-to-staging.sh"
kubectl --context <eks-context> rollout restart deployment/loculus-backend -n staging
```

Note the UTC time the clone finished in the bump PR: it is what staging-only and prod-only rows are later judged against.

If a surgery runs on prod after the clone, clone again before validating: staging otherwise lacks its changes.

### Check staging is identical to prod

Wait for SILO to import the cloned data first. A SILO pod only becomes Ready after its first successful import, and the old pod keeps serving until then, so a regression run too early compares stale data. Staging is ready when each organism's LAPIS count equals prod's and it serves a single `pipelineVersion`:

```sh
curl -s "https://lapis-staging.pathoplexus.org/<organism>/sample/aggregated?fields=pipelineVersion"
curl -s "https://lapis.pathoplexus.org/<organism>/sample/aggregated?fields=pipelineVersion"
```

We then make sure that staging really is identical to prod by running the integrity script from `pathoplexus/pathoplexus`:

```sh
cd ~/code/pathoplexus/data-integrity-tests/regression-testing
micromamba activate pp-integrity
mv results results-$(date -u +%Y%m%dT%H%MZ)   # keep the previous run's diffs; a full run writes ~38 GB
snakemake -F
ls results/*.diff | wc -l                       # 2 per organism, 30 for the 15 in the suite
find results -name '*.diff' -size +1c           # prints nothing when staging and prod are identical
```

Identical means every sequence diff is 0 bytes and every metadata diff is 1 byte (a blank line). Check the files themselves: the workflow always exits 0, and a target that failed to build leaves no file, which `cat results/*.diff` would show as "no differences". Run it only inside `pp-integrity`, which pins GNU coreutils: Ubuntu's uutils `sort` deadlocks on mpox's long lines.

The suite lists its organisms explicitly, so a newly added organism is not covered. Check it separately (LAPIS count, search page, news page). To compare a subset, for example organisms that have finished reprocessing, pass their diff files as explicit targets.

### Run DB surgeries on staging

Every hand-run change to the prod database goes through [pathoplexus/db-surgery](https://github.com/pathoplexus/db-surgery): one surgery per PR, with the header from its `TEMPLATE.sql`. Run surgeries on staging after the clone and before the bump, so the validation sees them:

1. Each surgery is one transaction that starts with the database guard from `TEMPLATE.sql` (`current_database()` against `expected_db`) and asserts its expected row counts.
2. Dry run on staging ending in `ROLLBACK`; record the counts in the PR. Then run it for real on staging.
3. Commit the file with the guard set to staging.

The guard only compares database names. In pgAdmin, also check which server you are connected to.

### Point staging at main's commit

We can then rollout a new commit to staging, via PR in this repo:

```sh
gh workflow run bump-staging.yml -R pathoplexus/loculus_deployments
```

Trigger it only after staging is on the prod SHA and the clone is done: the workflow writes the changelog and compare link from staging's SHA at the moment it runs, so an earlier run lists only part of what the bump deploys.

### Wait for reprocessing

If the new version adds preprocessing pipeline versions, every affected organism reprocesses (all 16 took about 2 h 20 min on 2026-10-01). An organism is done when its LAPIS serves only the new `pipelineVersion`.

The backend switches an organism to the new version only when every entry that is error-free on the current version is also error-free on the new one. A single entry that newly errors holds the organism on the old version, and nothing reports it. If an organism has not switched after its new pipeline has gone idle, look for entries with errors on the new version only.

Expected during the rollout, and not a problem:

- every preprocessing pod restarts once when the old backend pod goes away, and a new organism's preprocessing pod restarts until the new backend is up;
- old and new website pods serve side by side for a few minutes, so content that exists only in the new version (a new organism's card, a new news page) appears and disappears per request;
- silo-importer cycles that fail with `the column 'X' is not contained in the object` or `IncompleteRead` and succeed on the next cycle;
- after every Argo CD sync, a PostSync hook starts one ingest per organism (`loculus-ingest-bootstrap-<organism>-<ts>` jobs). Check those before starting an ingest by hand; until a new organism's first ingest, its homepage card shows `-1 sequences`.

`No lineage definition URL configured for pipeline version N` does not fix itself.

### Check staging doesn't differ from prod (or only as expected)

When reprocessing has finished, rerun the regression (above) and explain every difference in the bump PR. For each row that exists on only one side, compare its submitted and released times with the clone and bump times:

- submitted on prod after the clone: prod-only, expected;
- ingested on both sides after the clone: staging runs its own ingest, so the same INSDC record gets a different accession on each side, expected;
- submitted before the clone but released on staging only after the bump: the new code accepts entries the old code rejected. Prod will release them too. Name the cause and get it accepted before promoting.

Then complete the rest of the bump PR's post-merge checklist, including submit/revise/revoke. The staging website is behind basic auth (credentials with the team); a port-forward of the website service works for quick page checks.

### Promote to production

Once that's done, you can (after reviewing everything looks healthy and testing new features) deploy to production through PR triggered by:

```sh
gh workflow run promote-staging-to-production.yml -R pathoplexus/loculus_deployments
```

Link the bump PR that deployed this exact SHA, with its post-merge checklist completed.

Before merging, run the staging surgeries on prod:

1. Re-check each surgery's asserted counts on prod, read-only, on the day: rows ingested or submitted since the staging run can change them.
2. Note the UTC time, for point-in-time recovery if the surgery has to be undone.
3. Switch the guard to prod locally, dry run with `ROLLBACK`, then run it.
4. Fill in the actual counts and the run details in the db-surgery PR, switch the guard back to staging, and merge.

### After promoting

Watch the rollout and, once reprocessing has finished, repeat the staging checks against prod. Record the results in the promote PR.

- Argo CD app `loculus-pp-production` is Synced/Healthy with a recent finish time:
  `kubectl --context <eks-context> -n argocd get applications.argoproj.io loculus-pp-production -o custom-columns='SYNC:.status.sync.status,HEALTH:.status.health.status,FINISHED:.status.operationState.finishedAt'`
- Every organism ends up serving only its new `pipelineVersion` on `https://lapis.pathoplexus.org`.
- No silo-importer logs `No lineage definition URL`, no SILO pod restarts or OOMs, no new backend ERROR lines.
- A new organism has data in LAPIS and on its website pages.

Rolling prod back to an older SHA after the new pipeline versions are current makes the silo-importers loop with `No lineage definition URL configured for pipeline version N`. Add the new versions to the older values' `lineageSystemDefinitions` first. Rolling back across a Flyway migration needs a database restore.

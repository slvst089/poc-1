### _Issue 01 - Fix Helm 'UPGRADE FAILED: has no deployed releases'_
###### Published Time: 2026-03-30


#### [The Error: "has no deployed releases"](https://devopsil.com/articles/2026-03-30-helm-upgrade-failed-no-deployed-releases-fix#the-error-has-no-deployed-releases)

You run `helm upgrade --install` and get:

```
Error: UPGRADE FAILED: "my-app" has no deployed releases
```

This is confusing because you're using the `--install` flag, which should handle first-time installs. But [Helm](https://devopsil.com/tools/helm "Helm") still refuses.
<br><br>

#### [Root Cause](https://devopsil.com/articles/2026-03-30-helm-upgrade-failed-no-deployed-releases-fix#root-cause)

This happens when a previous `helm install` (or `helm upgrade --install`) failed partway through. The release exists in Helm's history with a `failed` status, but no revision was ever successfully `deployed`. Helm sees an existing release (so it tries `upgrade`), but the upgrade logic requires at least one `deployed` revision to diff against. Since none exists, it errors out.

```
helm history my-app -n your-namespace
```

```
REVISION  STATUS  CHART          DESCRIPTION
1         failed  my-app-1.0.0   Install complete (failed)
```

That single `failed` revision is the culprit.
<br><br>

#### [Step-by-Step Fix](https://devopsil.com/articles/2026-03-30-helm-upgrade-failed-no-deployed-releases-fix#step-by-step-fix)

##### [Option A: Uninstall and Reinstall (Safest)](https://devopsil.com/articles/2026-03-30-helm-upgrade-failed-no-deployed-releases-fix#option-a-uninstall-and-reinstall-safest)

If the application was never successfully running:

```
# Remove the failed release
helm uninstall my-app -n your-namespace

# Install fresh
helm install my-app ./my-chart -n your-namespace -f values.yaml
```

Or in one step if you prefer `upgrade --install`:

```
helm uninstall my-app -n your-namespace 2>/dev/null; \
helm upgrade --install my-app ./my-chart -n your-namespace -f values.yaml
```

##### [Option B: Force Replace the Failed Release](https://devopsil.com/articles/2026-03-30-helm-upgrade-failed-no-deployed-releases-fix#option-b-force-replace-the-failed-release)

If you want to keep using `upgrade --install` in a CI/CD pipeline without two steps:

```
helm upgrade --install my-app ./my-chart \
  -n your-namespace \
  -f values.yaml \
  --force \
  --reset-values
```

Note: `--force` replaces resources using delete/recreate, which causes brief downtime.

##### [Option C: Delete the Failed Release Secret Manually](https://devopsil.com/articles/2026-03-30-helm-upgrade-failed-no-deployed-releases-fix#option-c-delete-the-failed-release-secret-manually)

If `helm uninstall` itself fails:

```
# List all release secrets
kubectl get secrets -n your-namespace -l "owner=helm,name=my-app"

# Delete them all
kubectl delete secrets -n your-namespace -l "owner=helm,name=my-app"

# Now install fresh
helm upgrade --install my-app ./my-chart -n your-namespace -f values.yaml
```

##### [Option D: Fix in CI/CD Pipelines](https://devopsil.com/articles/2026-03-30-helm-upgrade-failed-no-deployed-releases-fix#option-d-fix-in-cicd-pipelines)

For automated pipelines, wrap the logic to handle this case:

```
#!/bin/bash
set -e

RELEASE="my-app"
NAMESPACE="production"
CHART="./my-chart"

# Check if release exists and has only failed revisions
STATUS=$(helm status "$RELEASE" -n "$NAMESPACE" -o json 2>/dev/null | jq -r '.info.status' || echo "not-found")

if [ "$STATUS" = "failed" ]; then
  echo "Found failed release, uninstalling first..."
  helm uninstall "$RELEASE" -n "$NAMESPACE" --wait
fi

helm upgrade --install "$RELEASE" "$CHART" \
  -n "$NAMESPACE" \
  -f values.yaml \
  --wait \
  --timeout 5m0s
```
<br><br>
##### [Verify the Fix](https://devopsil.com/articles/2026-03-30-helm-upgrade-failed-no-deployed-releases-fix#verify-the-fix)

```
helm list -n your-namespace
```

You should see:

```
NAME     NAMESPACE   REVISION  STATUS    CHART
my-app   production  1         deployed  my-app-1.0.0
```

Also check that the actual workloads are running:

```
kubectl get pods -n your-namespace -l app.kubernetes.io/instance=my-app
```
<br><br>
#### [Prevention Tips](https://devopsil.com/articles/2026-03-30-helm-upgrade-failed-no-deployed-releases-fix#prevention-tips)

*   **Always use `--atomic`** in CI/CD: `helm upgrade --install --atomic --timeout 5m0s`. This auto-rolls back on failure, leaving the release in a clean state instead of `failed`.
*   **Use `--wait`** so Helm waits for pods to be ready and properly records success or failure.
*   **Fix the root cause of the initial failure** (bad image tag, missing secrets, resource limits) before retrying. Blindly rerunning the pipeline hits the same error.
*   **Add the uninstall-on-failed pattern** to your CI/CD scripts as shown in Option D above.
*   **Test chart changes locally** with `helm template` and `helm lint` before pushing to CI.

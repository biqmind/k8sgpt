# Fork image: disable ConfigMap usage findings

The `Build ConfigMap-usage-disabled image` workflow applies
`disable-configmap-usage.patch` in its checkout, tests the patched analyzer, and
publishes a multi-platform image using the upstream `container/Dockerfile`.
The patch is not committed back to the source files, keeping upstream syncs
separate from the customization. Empty-data and size checks, including existing
upstream exemptions, retain their previous behavior. The patch deliberately
leaves Pod discovery code alone to minimize the change maintained across syncs.

## Install

Commit these files to the fork's `main` branch:

- `.github/workflows/build-patched-image.yaml`
- `patches/disable-configmap-usage.patch`
- `patches/README.md`

Do not also commit the patched Go source: the workflow applies it at build time.
Enable Actions in the fork. Organization policies must allow the referenced
actions and the workflow's `packages: write` permission. No separate registry
secret is required; publishing uses the job's `GITHUB_TOKEN`.

The inherited `Build container` workflow still targets `ghcr.io/k8sgpt-ai/k8sgpt`
and an upstream-only secret. Disable that workflow in the fork's Actions UI if
you only need the patched image; this new workflow does not change it.

## Run and deploy

A normal GitHub **Sync fork** update to `main` triggers the workflow. You can also
select it in Actions and choose **Run workflow** on `main` (including when there
are no new upstream changes). The workflow does not itself sync the fork.

An automated sync performed using another workflow's `GITHUB_TOKEN` does not
trigger a subsequent `push` workflow. In that case explicitly dispatch this
workflow after the sync, or use a suitably scoped GitHub App/PAT for the sync.
Do not force-reset the fork to upstream: that would remove these fork-only files.
Use the normal merge-based Sync fork flow instead.

Published tags:

```text
ghcr.io/biqmind/k8sgpt:no-configmap-usage
ghcr.io/biqmind/k8sgpt:no-configmap-usage-<full-source-commit-sha>
```

Both `linux/amd64` and `linux/arm64` are built. Use the image digest recorded in
the workflow summary for reproducible deployments. Updating a rolling tag alone
does not restart existing workloads; update/reconcile the analyzer deployment.
New GHCR packages may be private: either make the package public or configure
image pull credentials on the cluster. If a package already exists, grant this
repository Actions access in that package's settings.

If upstream changes the patched code/tests incompatibly, patch application or
regression tests fail before publishing. Review and update the patch rather
than silently dropping the customization. Build arguments follow the current
Dockerfile's `VERSION`, `COMMIT`, and `DATE` names.

# Payhawk's Supabase chart

This fork exists to give `charts/supabase` something the upstream repository does not: a **version
that can be pinned**. Upstream publishes the chart only as files in a git repository — there is no
Helm repository index, so `helm upgrade --repo … --version …` has nothing to resolve, and the only
way to consume it is to copy the directory.

Branches here:

| branch | what |
|---|---|
| `main` | upstream, untouched — kept so `git diff` against it stays meaningful |
| `payhawk/supabase-0.7.2` | upstream tag `supabase-0.7.2` plus the three patches below |
| `charts` | packaged `.tgz` artifacts, one per released version |

The consumer installs the packaged tarball by URL and verifies its SHA-256 before handing it to
Helm, so a force-push to this repository cannot change what a cluster runs.

## Base

| | |
|---|---|
| upstream | [supabase-community/supabase-kubernetes](https://github.com/supabase-community/supabase-kubernetes) |
| tag | `supabase-0.7.2` |
| commit | `d04a3133af93cc12af000b15f88c22507be5354f` |
| chart version | `0.7.2-payhawk.1` |

The community *Operator* in the same repository is `v1alpha1`/alpha and is deliberately not used.

## Patches

Everything expressible as a values override belongs in the consumer's `values.yaml`. These three are
not — they are template-level, so they live here.

Two are in `templates/storage/deployment.yaml`, both because upstream assumes the only S3 backend is
its own bundled minio:

1. `STORAGE_BACKEND` was hardcoded to `minio.enabled ? "s3" : "file"`. It now honours a top-level
   `storageBackend` value. Setting it through `environment.storage` instead produces a duplicate env
   key, which server-side apply rejects outright.
2. `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` were always read from the *minio* secret. They are
   the credentials storage-api uses against the object store, so with minio off they now come from
   the `s3` secret. Symptom of the bug: uploads fail `SignatureDoesNotMatch` while listing buckets
   (database-only) keeps returning 200.

The third is in `files/kong/kong.yml`:

3. The `dashboard` service — Studio on a catch-all `/` route, with the `basic-auth` plugin — was
   **deleted**, which is upstream's own advice when Studio is not served through Kong
   (supabase/supabase discussion #43852). A catch-all there answers every path Kong does not
   recognise with `WWW-Authenticate: Basic`, which a browser turns into a username/password dialog.
   It also makes one shared password the only thing between a request and full `service_role`
   access. Serve Studio on its own hostname, gated by your own front door. A request Kong cannot
   route now gets a plain 404.

## Releasing a new version

```sh
git checkout -B payhawk/supabase-<upstream-version> <upstream-tag>
# re-apply the three patches, read the diff against the tag
$EDITOR charts/supabase/Chart.yaml            # version: <upstream-version>-payhawk.<n>
git commit -am "chart: supabase <upstream-version>-payhawk.<n>"
git push origin payhawk/supabase-<upstream-version>

helm package charts/supabase -d /tmp
git checkout charts
cp /tmp/supabase-<version>.tgz .
shasum -a 256 supabase-<version>.tgz | tee -a SHA256SUMS
git commit -am "charts: supabase <version>" && git push origin charts
```

Then bump the version **and the digest** wherever the chart is consumed. Never overwrite a `.tgz`
that has already been published — cut a new `-payhawk.<n>` instead, so a pinned digest stays valid.

# Packaged charts

Install by URL and verify the digest in `SHA256SUMS` before handing the file to Helm.

Never overwrite a published `.tgz` — cut a new `-payhawk.<n>` instead, so a pinned digest stays valid.

Source branches and patch rationale: `payhawk/supabase-*` and its `PAYHAWK.md`.

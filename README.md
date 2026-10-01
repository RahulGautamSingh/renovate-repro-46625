# Minimal reproduction: postUpgradeTasks commits stale content

Reproduction for [renovatebot/renovate#46625](https://github.com/renovatebot/renovate/issues/46625), fixed in [renovatebot/renovate#46626](https://github.com/renovatebot/renovate/pull/46626).

Based on [dlm6693/renovate-repro-42729](https://github.com/dlm6693/renovate-repro-42729), from [discussion #42729](https://github.com/renovatebot/renovate/discussions/42729).

## Current behavior

`post-upgrade.sh` appends a marker line to `lib/go.sum` and `app/go.sum`.
Renovate logs `Post-upgrade file saved` for both, but the branch it pushes has the marker only in `lib/go.sum`.

## Expected behavior

Both `go.sum` files on the pushed branch end with `// renovate-post-upgrade-marker`.

## Why

With `gomodTidyAll`, the `updateArtifacts()` call for `lib` also returns its dependent `app/go.sum`.
`app` updates `github.com/google/uuid` directly, so its own call returns `app/go.sum` too.
`updatedArtifacts` ends up with two entries for `app/go.sum` and one for `lib/go.sum`.
The post-upgrade step refreshes only the first entry, and the commit writes the stale second entry over it.

## How to run

`postUpgradeTasks` needs self-hosted Renovate with the command allowed:

```bash
RENOVATE_ALLOWED_COMMANDS='["^\\./post-upgrade\\.sh$"]'
```

Then check the pushed branch:

```bash
git show renovate/github.com-google-uuid-1.x:lib/go.sum | tail -1
git show renovate/github.com-google-uuid-1.x:app/go.sum | tail -1
```

## Verified

Run with Renovate from source, `binarySource=global`, Go 1.26.1.

| Renovate | `updatedArtifacts` | `lib/go.sum` | `app/go.sum` |
| --- | --- | --- | --- |
| `main` (8f2ffc0e90) | `["app/go.sum", "lib/go.sum", "app/go.mod", "app/go.sum"]` | marker present | **marker missing** |
| with [#46626](https://github.com/renovatebot/renovate/pull/46626) (03dd814a16) | `["app/go.sum", "lib/go.sum", "app/go.mod"]` | marker present | marker present |

Both runs logged `Post-upgrade file saved` twice. The branches Renovate pushed are kept:

- [`evidence/unpatched-main`](../../tree/evidence/unpatched-main)
- [`evidence/with-fix`](../../tree/evidence/with-fix)

The only difference between them is the line the bug dropped:

```console
$ git diff origin/evidence/unpatched-main origin/evidence/with-fix
--- a/app/go.sum
+++ b/app/go.sum
@@ -1,2 +1,3 @@
 github.com/google/uuid v1.6.0 h1:NIvaJDMOsjHA8n1jAhLSgzrAzy1Hgr+hNrb57e+94F0=
 github.com/google/uuid v1.6.0/go.mod h1:TIyPZe4MgqvfeYDBFedMoGGpEw/LqOeaOT+nhxU+yHo=
+// renovate-post-upgrade-marker
```

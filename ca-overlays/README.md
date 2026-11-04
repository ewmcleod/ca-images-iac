# Custom Assembly Overlays

Source of truth for **overlay-based Custom Assembly**, reconciled by
`.github/workflows/custom-assembly-overlay.yml` (push to `main` + manual). Each
`<file>.yaml` becomes an org-level overlay named `<file>`.

It uses the `CHAINGUARD_ORG` and `CHAINGUARD_IDENTITY` repository variables.
There is no business-unit (`BU`) routing for overlays: an overlay is shared,
and where it applies is declared explicitly in its `bindings:`.

## How overlays differ from `ca-images-iac/`

| | `ca-images-iac/` (build apply) | `ca-overlays/` (overlays) |
| --- | --- | --- |
| Unit | one config per repo | one reusable overlay, bound to many repos |
| Targets | the file name / `<name>-<BU>` variant | declared in `bindings:` |
| Tag scope | all tags | `all-tags`, `tag-variant: dev`, or `exact-tags` |
| Build | one build, workflow waits for it | Chainguard rebuilds every matching tag; workflow waits for all of them |

> **A repo can't use both.** Don't bind an overlay to a repo that the
> `custom-assembly-build*.yml` workflows customize; `attach` will fail.
> Overlay rebuilds only run for orgs enrolled in overlay-based Custom Assembly.

## File format

One `<name>.yaml` (or `.yml`) per overlay directly in this directory
(subdirectories such as `examples/` are ignored):

```yaml
contents:
  packages: [gcsfuse, fuse3]
environment:
  FOO: bar
annotations:
  "com.acme.business-unit": "sws-wai"   # optional, passed through as-is

bindings:
  - repo: python
    exact-tags: ["3.13", "3.13-dev"]     # quote tags!
  - repo: flink
    tag-variant: dev
  - all-repos: true                       # every repo in the org, incl. future ones
    all-tags: true
```

Everything except `bindings:` is passed to `chainctl images overlays create/update -f`.
Each binding needs exactly one target (`repo` or `all-repos: true`) and exactly
one selector (`all-tags: true`, `tag-variant: dev`, or `exact-tags`).

## What a run does

For each file:

1. **Overlay** — creates it, or runs `update` when the content digest changes.
   The digest covers the config and all `certs/` files and is saved as the
   `dev.ca-images-iac.overlay-digest.<name>` annotation, so unchanged overlays
   are skipped and no rebuilds are triggered.
2. **Bindings** — `attach` new targets, `update-binding` changed selectors, and
   `detach` bindings of this overlay that the file no longer declares (set
   `PRUNE_BINDINGS` to anything other than `"true"` in `custom-assembly-overlay.yml`
   to only warn instead).
3. **Rebuild check** — one job after all overlays have reconciled. For every
   repo where an overlay changed (step 1 or 2), it waits for the rebuilds and
   fails if any tag stream's latest rebuild failed (e.g. a templated package
   that doesn't exist for one Python version). Rebuilds are matched by each
   overlay's digest annotation, which is copied into every rebuilt image, so
   unrelated builds of the same repo are ignored.

   It is checked **per repo**, not per overlay, because several overlays can
   share a repo (e.g. an `all-tags` overlay plus a `tag-variant: dev` one):
   Chainguard runs each overlay change as its own rebuild wave, one after
   another per repo, so the second overlay's rebuilds can start many minutes
   after its change. The check waits until every changed overlay on the repo
   has produced a rebuild (or the repo has been idle for
   `REBUILD_FIRST_REPORT_SECONDS`), and the summary table lists which changed
   overlays each image's build carried. Tunables (`REBUILD_*`) are in the
   workflow's `env:`. `all-repos` bindings are not checked.

Not handled: deleting the overlay when its file is removed. Detach its bindings
and run `chainctl images overlays delete <name>` by hand.

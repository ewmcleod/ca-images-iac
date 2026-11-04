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
| Build | workflow waits for build reports | Chainguard rebuilds matching tags asynchronously |

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
   `dev.ca-images-iac.overlay-digest` annotation, so unchanged overlays are
   skipped and no rebuilds are triggered.
2. **Bindings** — `attach` new targets, `update-binding` changed selectors, and
   `detach` bindings of this overlay that the file no longer declares (set
   `PRUNE_BINDINGS` to anything other than `"true"` in `custom-assembly-overlay.yml`
   to only warn instead).

Not handled: deleting the overlay when its file is removed. Detach its bindings
and run `chainctl images overlays delete <name>` by hand.

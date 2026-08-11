# ca-images-iac

Infrastructure-as-code for building customized [Chainguard](https://www.chainguard.dev/)
container images with [Custom Assembly](https://edu.chainguard.dev/chainguard/chainguard-images/features/ca-docs/custom-assembly-chainctl/).

GitHub Actions workflows discover the image configs in this repo, apply each one
(plus any shared CA certificates), and build the customized images.

## Contents

| Path | What it is |
| --- | --- |
| [`.github/workflows/custom-assembly-build-bu.yml`](.github/workflows/custom-assembly-build-bu.yml) | **Per–business-unit variants** (default, push-triggered). |
| [`.github/workflows/custom-assembly-build.yml`](.github/workflows/custom-assembly-build.yml) | **In-place customization** (manual `workflow_dispatch`). |
| [`ca-images-iac/`](ca-images-iac/) | One `<name>.yaml` Custom Assembly config per image. |
| [`ca-images-iac/README.md`](ca-images-iac/README.md) | **Full usage guide** — setup, layout, the business-unit annotation, troubleshooting. |
| [`certs/`](certs/) | Optional org-wide CA certificates injected into every build. |

## Workflows

Both read the same flat configs under `ca-images-iac/` (`<name>.yaml`, where
`<name>` is the Chainguard base image) and inject any CA certs from `certs/`.
They differ in **what they produce**:

### `custom-assembly-build.yml` — customize in place

Rebuilds each base image **in place** — the source and the built repo are both
`<name>`.

- **⚠️ Overwrites the existing image.** The customization is applied directly to
  `<name>`, replacing the current image rather than producing a separate copy.
- **Trigger:** manual only (**Actions → Custom Assembly Build → Run workflow**),
  so it only runs when you deliberately intend to overwrite.
- **Waiting on save-as:** building to a separate repo is intentionally omitted
  for now, pending Custom Assembly support for a declared `save-as` target in the
  config YAML. Once that's available, this can produce a copy instead of
  overwriting in place.

### `custom-assembly-build-bu.yml` — per–business-unit variants

Builds a **variant** repo `<name>-<BU>` from each `<name>` base, where `BU` is a
required repository variable. The base image is never modified. Requires each
config to declare a `*business-unit` annotation equal to `BU`.

- **Trigger:** automatic on push to `ca-images-iac/**` / `certs/**`, plus manual.
- **Use when:** you want per-business-unit copies without touching the bases.

> Keep only one of the two on `push` at a time — running both automatically
> would build the same configs two different ways on every commit.

## Quick start

1. Configure the required repository variables (**Settings → Secrets and
   variables → Actions → Variables**): `CHAINGUARD_IDENTITY`, `BU`, and
   `CHAINGUARD_ORG`.
2. Add an image config at `ca-images-iac/<name>.yaml` including a
   `business-unit` annotation whose value equals `BU`.
3. Commit and push — the workflow builds `<name>-<BU>` from the `<name>` base.

See [`ca-images-iac/README.md`](ca-images-iac/README.md) for the complete guide.

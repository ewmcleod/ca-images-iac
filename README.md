# ca-images-iac

Infrastructure-as-code for building customized [Chainguard](https://www.chainguard.dev/)
container images with [Custom Assembly](https://edu.chainguard.dev/chainguard/chainguard-images/features/ca-docs/custom-assembly-chainctl/).

A GitHub Actions workflow discovers the image configs in this repo, applies each
one (plus any shared CA certificates), and builds a per–business-unit variant of
every image.

## Contents

| Path | What it is |
| --- | --- |
| [`.github/workflows/custom-assembly-bu.yml`](.github/workflows/custom-assembly-bu.yml) | The build pipeline. |
| [`ca-images-iac/`](ca-images-iac/) | One `<name>.yaml` Custom Assembly config per image. |
| [`ca-images-iac/README.md`](ca-images-iac/README.md) | **Full usage guide** — setup, layout, the business-unit annotation, troubleshooting. |
| [`certs/`](certs/) | Optional org-wide CA certificates injected into every build. |

## Quick start

1. Configure the required repository variables (**Settings → Secrets and
   variables → Actions → Variables**): `CHAINGUARD_IDENTITY`, `BU`, and
   `CHAINGUARD_ORG`.
2. Add an image config at `ca-images-iac/<name>.yaml` including a
   `business-unit` annotation whose value equals `BU`.
3. Commit and push — the workflow builds `<name>-<BU>` from the `<name>` base.

See [`ca-images-iac/README.md`](ca-images-iac/README.md) for the complete guide.

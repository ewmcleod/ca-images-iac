# Custom Assembly Build — BU pipeline

This directory is the source of truth for the
[`.github/workflows/custom-assembly-bu.yml`](../.github/workflows/custom-assembly-bu.yml)
workflow (**"Custom Assembly Build - BU"**). The workflow customizes Chainguard
images with [Custom Assembly](https://edu.chainguard.dev/chainguard/chainguard-images/features/ca-docs/custom-assembly-chainctl/)
and builds a per–business-unit **variant** of each one.

For every image config it:

1. **Discovers** the image and computes its variant repo name.
2. **Verifies** the config declares a business-unit annotation matching the `BU`
   variable.
3. **Applies** the config plus any global CA certificates, then **waits** for the
   resulting build to finish.

Each image builds in its own isolated job (`fail-fast: false`), so a bad or
failing image only fails its own job — the others still build.

---

## One-time setup

The workflow reads three **repository variables** (Settings → Secrets and
variables → Actions → **Variables**):

| Variable               | Required | Purpose                                                                                     |
| ---------------------- | -------- | ------------------------------------------------------------------------------------------- |
| `CHAINGUARD_ORG`       | yes\*    | Your Chainguard organization (parent), e.g. `example.com`.                                  |
| `CHAINGUARD_IDENTITY`  | yes\*    | The assumable OIDC identity `chainctl` authenticates as. See setup guide below.             |
| `BU`                   | **yes**  | Business-unit tag. Drives the variant repo name and must match each config's annotation.    |

\* If `CHAINGUARD_ORG` or `CHAINGUARD_IDENTITY` is missing, the workflow **skips
gracefully** (green, no build). If `BU` is missing, it **fails**.

Configure the assumable identity following Chainguard's guide:
<https://edu.chainguard.dev/chainguard/administration/iam-organizations/assumable-ids/identity-examples/github-actions-identity/>

---

## Directory layout

```
ca-images-iac/
  <image>.yaml          # Custom Assembly config, one per image (.yaml or .yml)
  python.yaml
  nginx.yaml
certs/                  # global CA certs injected into EVERY build (repo root)
```

Rules:

- Each image is a single config file `<image>.yaml` (or `.yml`) directly under
  `ca-images-iac/`. The filename (minus extension) **is** the Chainguard base
  image to customize (the *source*).
- Having **both** `<image>.yaml` and `<image>.yml` is an error and that image is
  skipped.

### Base vs. variant naming

| | |
| --- | --- |
| **Source (base)** | the filename without extension, e.g. `python` |
| **Target (variant repo)** | `<name>-<BU>`, e.g. `python-payments` when `BU=payments` |

The base image is **never** modified. On a variant's first build the workflow
creates `<image>-<BU>` from the base (via `--save-as`); on later runs it updates
that variant in place.

---

## The business-unit annotation (required)

Every image config **must** declare a business-unit annotation whose value
equals the `BU` repository variable. This keeps the value version-controlled
alongside the image and guards against building into the wrong BU.

- The annotation key may use **any** reverse-DNS prefix — the check matches any
  key whose final segment is exactly `business-unit`:
  - ✅ `com.acme.business-unit`
  - ✅ `com.companyname.business-unit`
  - ✅ `business-unit`
  - ❌ `com.acme.sub-business-unit` (not matched)
  - ❌ `business-unit-owner` (not matched)
- The value must **exactly equal** `BU`.
- A **missing**, **mismatched**, or **conflicting** (two matching keys with
  different values) annotation fails that image's build job.

---

## Adding a new image

1. Create the config file:

   ```
   ca-images-iac/redis.yaml
   ```

2. Write the Custom Assembly config (see schema below), including the
   business-unit annotation matching your `BU`:

   ```yaml
   contents:
     packages:
       - jq
       - curl
   annotations:
     "com.acme.business-unit": "payments"   # must equal the BU repo variable
   ```

3. Commit and push. The push triggers the workflow, which builds
   `redis-payments` from the `redis` base.

---

## Config schema

Configs are pure [apko](https://edu.chainguard.dev/chainguard/chainguard-images/features/ca-docs/custom-assembly-chainctl/)
overlays passed verbatim to `chainctl images repos build apply -f`. Only these
top-level keys are valid:

| Key            | Purpose                                                            |
| -------------- | ----------------------------------------------------------------- |
| `contents`     | `packages` to add (must exist in Chainguard's repo), runtime repos |
| `environment`  | environment variables (the `CHAINGUARD_` prefix is reserved)      |
| `annotations`  | OCI annotations (the `dev.chainguard` / `org.opencontainers` prefixes are reserved) |
| `accounts`     | users/groups and `run-as`                                         |
| `certificates` | custom CA certs (can also be injected globally — see below)       |

> **Reserved:** `org.opencontainers.*` and `dev.chainguard.*` annotations and
> `CHAINGUARD_*` environment variables cannot be set.

---

## Certificates

Any files under the repo-root `certs/` directory matching `*.pem`, `*.crt`,
`*.cer`, or `*.ca` are injected into **every** image build as
`--with-certificates`. These are meant for org-wide internal/private CAs and are
applied uniformly to all images — not a per-image setting.

> Custom certificates are a Chainguard **Beta** feature that requires enrollment;
> contact your Customer Success team to enable it.

---

## Running the workflow

- **Automatically:** any push that touches `ca-images-iac/**`, `certs/**`, or the
  workflow file itself.
- **Manually:** Actions → **Custom Assembly Build - BU** → **Run workflow**
  (`workflow_dispatch`).

Each build waits up to **10 minutes** for the triggered Chainguard build to
complete and fails if it does not succeed. If a config produces no changes, the
build is skipped ("No changes detected").

---

## Troubleshooting

| Symptom (in the job log)                                              | Cause / fix                                                                      |
| --------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Run is green but nothing built                                        | `CHAINGUARD_ORG` / `CHAINGUARD_IDENTITY` not set → graceful skip. Set them.      |
| `BU repo variable is not set`                                         | Set the `BU` repository variable.                                                |
| `has no '*business-unit' annotation`                                  | Add a `*business-unit` annotation to the config, value = `BU`.                   |
| `business-unit='x' but repo variable BU='y' — they must match`        | Align the annotation value with the `BU` variable.                               |
| `has conflicting business-unit annotations`                           | Keep a single `*business-unit` annotation.                                       |
| `has both <name>.yaml and <name>.yml`                                 | Keep only one config file per image.                                             |
| `Unable to assume the identity ...`                                   | The OIDC identity is misconfigured — recheck the assumable-identity setup.       |
| `timed out after 10m waiting for build(s)`                            | The Chainguard build is slow or stuck; re-run or investigate in the console.     |

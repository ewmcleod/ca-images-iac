# Custom CA certificates

Drop your organization's internal/private CA certificates in this directory and
they will be injected into **every** image build.

- Supported file extensions: `.pem`, `.crt`, `.cer`, `.ca`
- These are added to each build via `chainctl ... --with-certificates=<file>`.
- They apply uniformly to all images — this is not a per-image setting.
- Commit only **public CA certificates** here, never private keys.

> Custom certificates are a Chainguard **Beta** feature that requires enrollment;
> contact your Customer Success team to enable it.

This file is a placeholder so the empty `certs/` directory is tracked in git.

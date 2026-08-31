# Python License Check Action

Check Python project dependencies for license compliance using [licensecheck](https://github.com/FHPythonUtils/LicenseCheck).

This action:
1. Finds all `pyproject.toml` and `requirements.txt` files in your project
2. Runs `licensecheck` to validate dependency licenses
3. Fails if any dependencies have incompatible or unknown licenses

It works in repositories that have no top-level `pyproject.toml` — for example, a repo that only contains `requirements.txt` files.

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `uv-version` | uv version to use | No | `7.1.3` |
| `licensecheck-version` | licensecheck version to use. A bare version resolves from PyPI; a `git+...` value is installed from source | No | `2026.0.8` |
| `skip-dependencies` | Space-separated list of dependencies to skip | No | `wrapt` (BSD-2-Clause, see [issue](https://github.com/GrahamDumpleton/wrapt/issues/298)) |
| `ignore-licenses` | Space-separated list of license types to ignore | No | `MPL` |
| `allowed-license-references` | Space-separated list of exact `LicenseRef-*` identifiers to accept | No | `""` |
| `license-overrides` | Multiline TOML entries and comments assigning reviewed licenses to exact package versions | No | `""` |
| `requirements-paths` | Paths to search for requirements files | No | `.` |
| `allow-no-packages` | Treat LicenseCheck's `NO_PACKAGES` result as success | No | `false` |
| `app-id` | GitHub App ID for accessing private repos | No | `""` |
| `app-private-key` | GitHub App private key for accessing private repos | No | `""` |
| `repository-owner` | Repository owner for GitHub App token | No | Current repo owner |

## Usage

### Basic usage

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v6

  - name: Check licenses
    uses: CVector-Energy/license-check-python@main
```

### Skip specific dependencies

Add the `[tool.licensecheck]` section to your pyproject.toml to ignore specific dependencies. For example:

```
[tool.licensecheck]
ignore_packages = ["wrapt"]  # BSD-2-Clause license. See https://github.com/GrahamDumpleton/wrapt/issues/298
```

### Ignore specific license types

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v6

  - name: Check licenses
    uses: CVector-Energy/license-check-python@main
    with:
      ignore-licenses: "MPL BSD-3-Clause"
```

### Allow exact custom license references

Unlike `ignore-licenses`, this input compares the raw custom license identifier and does not allow the generic proprietary-license category.

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v6

  - name: Check licenses
    uses: CVector-Energy/license-check-python@main
    with:
      allowed-license-references: "LicenseRef-NVIDIA-Proprietary"
```

### Override defective package metadata

Overrides remain subject to LicenseCheck's normal license policy. Keys must use exact versions, and comments can document how each license was verified.

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v6

  - name: Check licenses
    uses: CVector-Energy/license-check-python@main
    with:
      allowed-license-references: "LicenseRef-NVIDIA-Proprietary"
      license-overrides: |
        # BSD-3-Clause; verified from the license text in the 3.2.0 wheel.
        "us==3.2.0" = "BSD-3-Clause"
        # CUDA Toolkit EULA; reviewed for this exact artifact.
        "cuda-toolkit==13.0.3.0" = "LicenseRef-NVIDIA-Proprietary"
```

The action writes these entries verbatim to a temporary `licensecheck.toml` and removes it when the check finishes. It fails instead of overwriting a repository-owned `licensecheck.toml`; repositories using that file should keep their overrides there.

### Check specific directories

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v6

  - name: Check licenses
    uses: CVector-Energy/license-check-python@main
    with:
      requirements-paths: "./backend ./frontend"
```

### Allow projects with no dependencies

LicenseCheck normally returns exit code 3 when it finds requirement files but no dependency packages. Data-only or standard-library-only projects can accept that result explicitly:

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v6

  - name: Check licenses
    uses: CVector-Energy/license-check-python@main
    with:
      allow-no-packages: true
```

### With private dependencies

If your project depends on private packages, you can create a GitHub App and pass its credentials to the action:

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v6

  - name: Check licenses
    uses: CVector-Energy/license-check-python@main
    with:
      app-id: ${{ vars.APP_ID }}
      app-private-key: ${{ secrets.APP_PRIVATE_KEY }}
```

Your GitHub App should have the repository permission to read contents.

The action will automatically configure git authentication for private repos if the credentials are provided.

## Setting up a GitHub App for Private Repos

If your project depends on private packages from your organization, you'll need to create a GitHub App to grant the action access to those repositories.

### 1. Create a GitHub App

1. Go to your organization settings: `https://github.com/organizations/YOUR_ORG/settings/apps`
2. Click "New GitHub App"
3. Fill in the basic information:
   - **Name**: Something like "License Check" or "CI Private Repo Access"
   - **Homepage URL**: Your organization's homepage or repository URL
   - **Webhook**: Uncheck "Active" (not needed for this use case)
4. Under **Repository permissions**, set:
   - **Contents**: Read-only
5. Under **Where can this GitHub App be installed?**:
   - Select "Only on this account"
6. Click "Create GitHub App"

### 2. Generate a private key

1. After creating the app, scroll down to "Private keys"
2. Click "Generate a private key"
3. Save the downloaded `.pem` file securely

### 3. Install the app in your organization

1. Go to the app's settings page
2. Click "Install App" in the left sidebar
3. Select your organization
4. Choose either:
   - **All repositories** (if you want access to all private repos)
   - **Only select repositories** (choose specific repos)
5. Click "Install"

### 4. Configure GitHub Actions secrets and variables

In each repository that needs to use the license check action:

1. Go to repository Settings → Secrets and variables → Actions
2. Add a new **variable**:
   - **Name**: `APP_ID`
   - **Value**: Your GitHub App ID (found on the app's settings page)
3. Add a new **secret**:
   - **Name**: `APP_PRIVATE_KEY`
   - **Value**: The contents of the `.pem` file you downloaded

You can also configure these at the organization level to make them available to all repositories.

## License

MIT - See [LICENSE](LICENSE) for details.

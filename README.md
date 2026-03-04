# build-spk

Reusable GitHub Actions for Sandstorm SPK packaging.

## Entry points

- `mnutt/build-spk@<ref>`: end-to-end flow (build + optional preview publish + optional PR comment).
- `mnutt/build-spk/actions/build@<ref>`: build-only primitive.
- `mnutt/build-spk/actions/publish-preview-spk@<ref>`: preview release upload/pruning.
- `mnutt/build-spk/actions/comment-pr-loader@<ref>`: PR loader comment helper.
- `mnutt/build-spk/.github/workflows/spk-build-reusable.yml@<ref>`: reusable workflow wrapper.

Use a tag or commit SHA for `<ref>` in production.

## Primary usage (`mnutt/build-spk@<ref>`)

This is the default entry point. It:

1. Builds the SPK.
2. Optionally publishes preview release assets.
3. Optionally comments the loader link on the PR.

### Inputs

Required:

- `sandstorm_keyring_b64`
- `spk_output_path`

Core optional:

- `lima_spk_bin`
- `pre_pack_command` (defaults to standard `.sandstorm/build.sh` in VM)
- `vm_timeout` (default `10m`, passed to `lima-spk vm up --timeout`)
- `pack_mode` (`release` or `test`)
- `artifact_name`

Publish/comment optional:

- `github_token` (falls back to `github.token` if omitted)
- `publish_preview_release` (`"true"`/`"false"`)
- `release_tag`
- `release_name`
- `release_body`
- `preview_max_age_days`
- `preview_max_assets`
- `comment_pr_loader` (`"true"`/`"false"`)
- `loader_base_url`
- `issue_number`
- `loader_comment_marker`

### Outputs

- `package_id`
- `spk_http_url` (when publish step runs)
- `loader_url` (when comment step runs)

## Example (end-to-end action)

```yaml
name: Build SPK

on:
  workflow_dispatch:
  pull_request_target:
    types: [opened, synchronize, reopened]

jobs:
  spk:
    runs-on: ubuntu-24.04
    permissions:
      contents: write
      issues: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4

      # App/runtime steps stay in caller repo:
      - uses: actions/setup-node@v4
        with:
          node-version: "24"
          cache: yarn
      - run: corepack enable
      - run: yarn install --immutable
      - run: yarn build

      - uses: mnutt/build-spk@main
        with:
          sandstorm_keyring_b64: ${{ secrets.SANDSTORM_TEST_APP_KEY_B64 }}
          pack_mode: test
          spk_output_path: build/my-app-${{ github.sha }}.spk
          artifact_name: my-app-test-spk-${{ github.sha }}
          publish_preview_release: "true"
          comment_pr_loader: "true"
```

## Build-only usage (`actions/build`)

If you only want packaging and will wire publish/comment yourself:

```yaml
- id: build
  uses: mnutt/build-spk/actions/build@main
  with:
    sandstorm_keyring_b64: ${{ secrets.SANDSTORM_APP_KEY_B64 }}
    pack_mode: release
    spk_output_path: build/my-app-${{ github.sha }}.spk
```

## Reusable workflow

Use this if you prefer a `workflow_call` interface:

```yaml
jobs:
  build-test-spk:
    uses: mnutt/build-spk/.github/workflows/spk-build-reusable.yml@main
    with:
      environment_name: test-spk
      checkout_pr_head_for_pr_target: true
      pack_mode: test
      spk_output_path: build/my-app-test-${{ github.sha }}.spk
      artifact_name: my-app-test-spk-${{ github.sha }}
      publish_preview_release: true
      comment_pr_loader: true
    secrets:
      sandstorm_keyring_b64: ${{ secrets.SANDSTORM_TEST_APP_KEY_B64 }}
      github_token: ${{ github.token }}
```

## Notes

- This repo does not install Node.js or build your app code; do that in the caller workflow.
- If `lima_spk_bin` is missing, the action bootstraps `/tmp/vagrant-spk/lima-spk`.

## Setting up GitHub secrets

> [!WARNING]
> App private keys, especially your production/release key, must remain strictly secret.
> Do not allow untrusted users to run workflows that can access these secrets (for example via PR-triggered workflows with elevated permissions).

Run this from your app repo root:

```bash
# 1) Ensure test app id/key exists (runs once)
lima-spk pack --dev --set-version 0.0.0-dev /tmp/test.spk

# 2) Export private key as base64
TEST_APP_ID="$(cat .sandstorm/sandstorm-test-app-id)"
TEST_APP_KEY_B64="$(lima-spk getkey "$TEST_APP_ID" | base64 | tr -d '\n')"

# 3) Save to GitHub Actions secret
gh secret set SANDSTORM_TEST_APP_KEY_B64 --body "$TEST_APP_KEY_B64"
```

## Setting up production app key secret

For production/release keys, there is no `.sandstorm/sandstorm-test-app-id` helper file.
Retrieve your app ID from `.sandstorm/sandstorm-pkgdef.capnp`, then run:

```bash
# 1) Set app id manually from .sandstorm/sandstorm-pkgdef.capnp
APP_ID="<your-app-id-from-sandstorm-pkgdef.capnp>"

# 2) Export private key as base64
APP_KEY_B64="$(lima-spk getkey "$APP_ID" | base64 | tr -d '\n')"

# 3) Save to GitHub Actions secret
gh secret set SANDSTORM_APP_KEY_B64 --body "$APP_KEY_B64"
```

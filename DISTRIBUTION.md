# Distribution Guide

## Cutting a release

The release workflow (`.github/workflows/release.yml`) triggers automatically whenever a tag matching `v*` is pushed (e.g. `v1.0.0`). The usual flow:

1. Bump the version in three places so they stay in sync:
   - `package.json` → `"version"`
   - `src-tauri/Cargo.toml` → `[package] version`
   - `src-tauri/tauri.conf.json` → `"version"`
2. Commit and push that change.
3. Create and push a matching tag:

```bash
git tag v1.0.0
git push origin v1.0.0
```

Pushing the tag triggers four parallel jobs (Windows x64, Windows ARM64, macOS Intel, macOS Apple Silicon), each of which builds, bundles, and uploads its installer to a GitHub Release for that tag (created automatically on first upload). A typical full run takes 10–20 minutes depending on GitHub's runner queue.

You can also run the workflow without pushing a tag via **Actions → release → Run workflow** — this builds every platform and attaches the results as downloadable workflow artifacts instead of touching any GitHub Release, which is useful for a dry run.

## Unsigned builds — what users will see

Out of the box, this workflow produces **unsigned** installers. That's a deliberate choice for a project with no code-signing certificate yet — it keeps the workflow buildable without any paid certificate. The practical effect:

- **Windows:** SmartScreen will show an "Unknown publisher" warning on first run. Users click "More info" then "Run anyway".
- **macOS:** Gatekeeper will refuse to open the app with a "damaged" or "unidentified developer" message until the user either right-clicks and chooses Open, or runs:
  ```bash
  xattr -cr "/Applications/Image Converter Pro.app"
  ```

Neither of these affect functionality — they're purely a first-run trust prompt. If you want to remove them, see signing setup below.

## Optional: code signing

The workflow as shipped does **not** set any signing environment variables, so builds are unsigned (see above). To enable signing, add the secrets below in your repo's **Settings → Secrets and variables → Actions**, then add a matching `env:` block to the **"Build Tauri app"** step in `.github/workflows/release.yml` (the step that runs `tauri-apps/tauri-action@v0`).

### Windows

You'll need an OV or EV code-signing certificate (from DigiCert, SSL.com, etc.) or an Azure Trusted Signing / Key Vault subscription. The simplest path for a `.pfx`-based certificate:

```yaml
env:
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  TAURI_SIGNING_PRIVATE_KEY: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY }}
  TAURI_SIGNING_PRIVATE_KEY_PASSWORD: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY_PASSWORD }}
```

`TAURI_SIGNING_PRIVATE_KEY`/`_PASSWORD` are for Tauri's own updater signature (separate from Authenticode signing of the `.exe` itself). For Authenticode signing of the installer, follow Tauri's Windows signing guide at `v2.tauri.app/distribute/sign/windows` — the exact setup (local `.pfx` import step vs. Azure Key Vault vs. a `signCommand` calling a tool like `relic`) depends on which certificate provider you use, since Windows certificate signing isn't a single environment-variable knob the way macOS notarization is.

### macOS

You'll need an Apple Developer Program membership ($99/year) to get a "Developer ID Application" certificate, plus an app-specific password for notarization. Add these repository secrets:

| Secret | What it is |
|---|---|
| `APPLE_CERTIFICATE` | Base64-encoded `.p12` certificate export |
| `APPLE_CERTIFICATE_PASSWORD` | Password used when exporting the `.p12` |
| `APPLE_SIGNING_IDENTITY` | e.g. `Developer ID Application: Your Name (TEAMID)` |
| `APPLE_ID` | Your Apple ID email |
| `APPLE_PASSWORD` | An app-specific password generated for that Apple ID |
| `APPLE_TEAM_ID` | Your 10-character Apple Developer Team ID |

Then add a matching `env:` block referencing each secret to the same build step. Full details: Tauri's macOS signing guide at `v2.tauri.app/distribute/sign/macos`.

## Verifying a release build locally before publishing

Since signing/notarization can only be fully tested in CI (or on a machine with the actual certificates installed), it's worth doing an unsigned dry run first:

```bash
npm run tauri build -- --target aarch64-apple-darwin
```

(Substitute your platform's target triple.) Confirm the app launches, converts a test batch correctly, and that the installer in `src-tauri/target/<target>/release/bundle/` installs cleanly before cutting a real release.

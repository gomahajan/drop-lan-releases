# Local Drop release runbook

Use this process when publishing Drop updates without GitHub Actions. It reproduces the useful work performed by the macOS and Windows workflows while using the two development computers as the runners.

The Mac chat is the release coordinator. It prepares the exact release source, builds and publishes macOS first, and then asks the Windows chat to complete the same version. The Windows chat must read this file before beginning.

## Safety rules

- Never commit, print, or transfer the Tauri updater private key or its password.
- Never publish an unsigned updater artifact.
- Never force-push either repository.
- Never replace another platform's entries in `latest.json`.
- Both platforms must build the same source commit and version.
- Do not push a `v*` tag in local-release mode. The app repository currently uses those tags to start GitHub Actions.
- Do not run backend tests as part of packaging. The build and packaging commands perform the required compile checks.
- Pull the releases repository immediately before preparing and immediately before pushing a platform update.

## Repository roles

- `gomahajan/drop-lan`: application source and packaging scripts.
- `gomahajan/drop-lan-releases`: signed downloadable artifacts and the updater feed in `latest.json`.

The updater accepts only artifacts signed by the existing Tauri updater key. A normal installer without its `.sig` file is not a valid in-app update.

## Part A: Mac chat starts a release

### 1. Prepare an exact release commit

Start from the application source intended for release. Confirm all desired changes are committed and pushed to `main`, and that unrelated working-tree changes are not included.

Choose the next version. For a normal bug-fix update, increment the patch version, such as `0.6.22` to `0.6.23`.

Set the identical version in all four files:

- `package.json`
- `src-tauri/tauri.conf.json`
- `src-tauri/Cargo.toml`
- the `drop` package entry in `src-tauri/Cargo.lock`

Commit the version change as `Release Drop X.Y.Z` and push `main`. Record the full commit SHA.

Create an immutable coordination branch pointing at that commit. This branch does not trigger the current Actions workflows:

```bash
git branch local-release/vX.Y.Z RELEASE_COMMIT_SHA
git push origin local-release/vX.Y.Z
```

If that branch already exists, verify it points to the intended commit. Do not move an existing local-release branch to different source.

### 2. Build the signed universal macOS packages

Check out the exact coordination branch and verify the version in all four files. Install locked dependencies, then run the existing packager:

```bash
git fetch origin
git switch --detach origin/local-release/vX.Y.Z
pnpm install --frozen-lockfile
./scripts/package-macos-universal.sh
```

The packager reads the updater key from the current environment or, by default, from:

```text
~/.tauri/drop-updater/drop-updater.key
~/.tauri/drop-updater/password.txt
```

It must produce these files in `artifacts/`:

```text
Drop-macos-universal.dmg
Drop-macos-universal.zip
Drop-macos-universal.app.tar.gz
Drop-macos-universal.app.tar.gz.sig
```

Record SHA-256 hashes for the artifacts. Confirm the `.sig` file exists and is nonempty.

### 3. Publish the macOS half

Use a fresh or clean checkout of `gomahajan/drop-lan-releases` and update it first:

```bash
git pull --ff-only
```

From the exact application-source checkout, prepare the release repository:

```bash
GITHUB_REF_NAME=vX.Y.Z node scripts/prepare-release.mjs \
  artifacts \
  PATH_TO_DROP_LAN_RELEASES \
  macos
```

Inspect the generated changes. `latest.json` must have version `X.Y.Z`, and it must contain both:

```text
darwin-aarch64
darwin-x86_64
```

If `latest.json` already contained a Windows entry for the same version, it must still be present. Immediately before committing, fetch `origin/main` and confirm the checkout's `HEAD` still equals `origin/main`. If the remote changed, use a fresh clean checkout of the updated release repository and rerun `prepare-release.mjs`; do not overwrite or force-push.

Commit and push:

```bash
git add latest.json downloads/vX.Y.Z
git commit -m "Publish Drop vX.Y.Z for macOS"
git push origin main
```

Verify the public feed and macOS archive URLs after the push:

```text
https://raw.githubusercontent.com/gomahajan/drop-lan-releases/main/latest.json
https://raw.githubusercontent.com/gomahajan/drop-lan-releases/main/downloads/vX.Y.Z/Drop-macos-universal.app.tar.gz
```

Do not wait for Windows before reporting that the macOS update is available.

### 4. Hand off to Windows

Tell the user:

```text
The macOS update is published. On the Windows computer, tell Codex:
"Update the Windows app by pulling gomahajan/drop-lan-releases and following LOCAL_RELEASE.md, Part B."
```

## Part B: Windows chat completes a release

When the user asks to update the Windows app, first clone or pull `gomahajan/drop-lan-releases` and read this entire file. Do not infer the build version from the app source's moving `main` branch.

### 1. Discover and verify the pending version

Read `latest.json` in the updated releases repository. Its version is the pending Windows version. It must already contain `darwin-aarch64` and `darwin-x86_64`, and it must not already contain a different Windows release for that version.

For version `X.Y.Z`, fetch the application source and check out the exact coordination branch:

```powershell
git fetch origin
git switch --detach origin/local-release/vX.Y.Z
git rev-parse HEAD
```

Verify that `package.json`, `src-tauri/tauri.conf.json`, `src-tauri/Cargo.toml`, and the `drop` entry in `src-tauri/Cargo.lock` all say `X.Y.Z`. Stop if the branch or any version is missing or inconsistent.

### 2. Build the signed Windows installer

Use the updater signing key already stored on the Windows computer. Set it only in the current PowerShell process if it is not already configured. Replace the placeholders with the actual local secret-file paths:

```powershell
$env:TAURI_SIGNING_PRIVATE_KEY = Get-Content "PATH_TO_DROP_UPDATER_KEY" -Raw
$env:TAURI_SIGNING_PRIVATE_KEY_PASSWORD = (Get-Content "PATH_TO_PASSWORD_FILE" -Raw).Trim()
pnpm install --frozen-lockfile
pnpm tauri build --bundles nsis
```

If the updater key is unavailable, stop and report that. Do not publish an unsigned installer.

Collect the exact updater filenames:

```powershell
$installer = Get-ChildItem "src-tauri/target/release/bundle/nsis/*-setup.exe" | Select-Object -First 1
if (-not $installer) { throw "Windows installer was not produced" }
$artifactDir = Join-Path $env:TEMP "drop-vX.Y.Z-windows-release"
New-Item -ItemType Directory -Force -Path $artifactDir | Out-Null
Copy-Item $installer.FullName (Join-Path $artifactDir "Drop-windows-x64-setup.exe")
Copy-Item "$($installer.FullName).sig" (Join-Path $artifactDir "Drop-windows-x64-setup.exe.sig")
if (-not (Test-Path (Join-Path $artifactDir "Drop-windows-x64-setup.exe.sig"))) { throw "Updater signature is missing" }
Get-FileHash (Join-Path $artifactDir "Drop-windows-x64-setup.exe") -Algorithm SHA256
```

### 3. Publish the Windows half

Use a separate, clean checkout of `gomahajan/drop-lan-releases`:

```powershell
git pull --ff-only
```

Reconfirm that `latest.json` says `X.Y.Z` and contains both macOS entries. From the exact application-source checkout, run:

```powershell
$env:GITHUB_REF_NAME = "vX.Y.Z"
node scripts/prepare-release.mjs $artifactDir PATH_TO_DROP_LAN_RELEASES windows
```

Validate the complete updater feed:

```powershell
$feed = Get-Content "PATH_TO_DROP_LAN_RELEASES/latest.json" -Raw | ConvertFrom-Json
if ($feed.version -ne "X.Y.Z") { throw "Wrong updater version" }
if (-not $feed.platforms.'darwin-aarch64') { throw "Missing Apple Silicon update" }
if (-not $feed.platforms.'darwin-x86_64') { throw "Missing Intel Mac update" }
if (-not $feed.platforms.'windows-x86_64') { throw "Missing Windows update" }
```

Immediately before committing, fetch `origin/main` and confirm the checkout's `HEAD` still equals `origin/main`. If the remote changed, use a fresh clean checkout of the updated release repository and rerun `prepare-release.mjs`. Never force-push.

```powershell
git add latest.json downloads/vX.Y.Z
git commit -m "Publish Drop vX.Y.Z for Windows"
git push origin main
```

Verify that the public `latest.json` contains all three platform entries and that the Windows installer URL responds. Report the pushed release-repository commit and installer SHA-256.

Clear the key material from the current shell when finished:

```powershell
Remove-Item Env:TAURI_SIGNING_PRIVATE_KEY -ErrorAction SilentlyContinue
Remove-Item Env:TAURI_SIGNING_PRIVATE_KEY_PASSWORD -ErrorAction SilentlyContinue
```

## Recovery rules

- If a build fails, fix the source on `main`, choose a new patch version, and create a new `local-release/vX.Y.Z` branch. Do not silently move an already published version to new source.
- If a releases-repository push is rejected, pull normally, inspect the new remote changes, rerun `prepare-release.mjs`, revalidate every platform entry, and push a new commit. Never force-push.
- If only macOS is published, Windows installations remain on their previous version until Part B finishes; their local data is unaffected.
- If only Windows is published, macOS installations remain on their previous version until Part A finishes.
- A failed publish must not delete the previous version's files. `latest.json` is the only pointer selecting the newest update.
- User library data and application updater files are separate. Publishing or correcting an application update must never modify the Drop Library repository.

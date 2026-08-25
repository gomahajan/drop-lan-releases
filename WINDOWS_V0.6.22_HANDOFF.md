# Windows v0.6.22 local release handoff

GitHub Actions has exhausted its included minutes. Build and publish the existing Drop v0.6.22 Windows release locally. Do not trigger GitHub Actions, modify application source, change the version, or create another tag.

## 1. Check out the exact release source

In the `gomahajan/drop-lan` repository:

```powershell
git status --short
git fetch --tags origin
git switch --detach v0.6.22
git rev-parse HEAD
git merge-base --is-ancestor 5b0d8cb HEAD
```

The expected `HEAD` is:

```text
4adc97f2ee6c954a241d7c5430934ed222caa8ff
```

The final command must exit successfully. Commit `5b0d8cb` is the Windows taskbar-icon fix.

## 2. Build the signed NSIS installer

Use the updater signing key already configured on the Windows computer. Never commit or print the private key. If the key is not available, stop and report that instead of publishing an unsigned installer.

```powershell
pnpm install --frozen-lockfile
pnpm tauri build --bundles nsis
```

If the signing key is stored in files but is not already exported, set it only for the current PowerShell process. Adjust the paths to the actual local key location:

```powershell
$env:TAURI_SIGNING_PRIVATE_KEY = Get-Content "PATH_TO_DROP_UPDATER_KEY" -Raw
$env:TAURI_SIGNING_PRIVATE_KEY_PASSWORD = (Get-Content "PATH_TO_PASSWORD_FILE" -Raw).Trim()
pnpm tauri build --bundles nsis
```

## 3. Collect the updater files

From the `drop-lan` repository:

```powershell
$installer = Get-ChildItem "src-tauri/target/release/bundle/nsis/*-setup.exe" | Select-Object -First 1
if (-not $installer) { throw "Windows installer was not produced" }
$artifactDir = Join-Path $env:TEMP "drop-v0.6.22-windows-release"
New-Item -ItemType Directory -Force -Path $artifactDir | Out-Null
Copy-Item $installer.FullName (Join-Path $artifactDir "Drop-windows-x64-setup.exe")
Copy-Item "$($installer.FullName).sig" (Join-Path $artifactDir "Drop-windows-x64-setup.exe.sig")
Get-FileHash (Join-Path $artifactDir "Drop-windows-x64-setup.exe") -Algorithm SHA256
```

Both the installer and its `.sig` file must exist.

## 4. Add Windows to the existing updater feed

Clone `gomahajan/drop-lan-releases` into a separate directory, or update an existing clean clone. The macOS v0.6.22 release was published in commit `ee437b1`.

```powershell
git clone https://github.com/gomahajan/drop-lan-releases.git PATH_TO_RELEASE_REPO
git -C PATH_TO_RELEASE_REPO pull --ff-only
```

Before continuing, `PATH_TO_RELEASE_REPO/latest.json` must say `0.6.22` and contain both `darwin-aarch64` and `darwin-x86_64`. If it does not, pull again and stop if it is still missing.

From the `drop-lan` v0.6.22 checkout:

```powershell
$env:GITHUB_REF_NAME = "v0.6.22"
node scripts/prepare-release.mjs $artifactDir PATH_TO_RELEASE_REPO windows
```

Verify that `latest.json` still has both macOS platform entries and now also has `windows-x86_64`:

```powershell
$feed = Get-Content "PATH_TO_RELEASE_REPO/latest.json" -Raw | ConvertFrom-Json
if ($feed.version -ne "0.6.22") { throw "Wrong updater version" }
if (-not $feed.platforms.'darwin-aarch64') { throw "Missing Apple Silicon update" }
if (-not $feed.platforms.'darwin-x86_64') { throw "Missing Intel Mac update" }
if (-not $feed.platforms.'windows-x86_64') { throw "Missing Windows update" }
```

## 5. Commit and publish

```powershell
git -C PATH_TO_RELEASE_REPO add latest.json downloads/v0.6.22
git -C PATH_TO_RELEASE_REPO commit -m "Publish Drop v0.6.22 for Windows"
git -C PATH_TO_RELEASE_REPO push origin main
```

Report the pushed commit and the installer SHA-256. Do not run or rerun any GitHub Actions workflow.

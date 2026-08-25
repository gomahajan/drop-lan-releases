# Windows v0.6.22 publish handoff

GitHub Actions has exhausted its included minutes, but its Windows build job completed successfully and uploaded the signed v0.6.22 artifact. Download and publish that artifact. Do not rebuild the app, request or copy the updater private key, trigger GitHub Actions, modify application source, change the version, or create another tag.

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

## 2. Download the existing signed Windows artifact

The successful Windows workflow run is `32893059675`, and its artifact is `drop-windows-x64`. It is still available in GitHub and already contains the installer and updater signature.

```powershell
$artifactDir = Join-Path $env:TEMP "drop-v0.6.22-windows-release"
New-Item -ItemType Directory -Force -Path $artifactDir | Out-Null
gh run download 32893059675 --repo gomahajan/drop-lan --name drop-windows-x64 --dir $artifactDir
```

Verify both exact files and record the installer hash:

```powershell
if (-not (Test-Path (Join-Path $artifactDir "Drop-windows-x64-setup.exe"))) { throw "Signed installer is missing" }
if (-not (Test-Path (Join-Path $artifactDir "Drop-windows-x64-setup.exe.sig"))) { throw "Updater signature is missing" }
Get-FileHash (Join-Path $artifactDir "Drop-windows-x64-setup.exe") -Algorithm SHA256
```

## 3. Add Windows to the existing updater feed

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

## 4. Commit and publish

```powershell
git -C PATH_TO_RELEASE_REPO add latest.json downloads/v0.6.22
git -C PATH_TO_RELEASE_REPO commit -m "Publish Drop v0.6.22 for Windows"
git -C PATH_TO_RELEASE_REPO push origin main
```

Report the pushed commit and the installer SHA-256. Do not run or rerun any GitHub Actions workflow.

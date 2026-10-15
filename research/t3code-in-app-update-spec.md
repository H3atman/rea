# T3 Code in-app update: findings and an openpiso Admin adaptation spec

Date: 2026-10-13
Target studied: `T3-Code-0.0.46-nightly.20261013.3042-x86_64.AppImage` (164,762,524 bytes) and `github.com/pingdotgg/t3code`
Target project: `/home/rgsv/Projects/personal/openpiso` (Admin app = Kiosk window + web-admin)

Labels: **OBSERVED** means packaged text or published files were inspected. **DERIVED** means it follows from the observed code. **INFERRED** means analyst judgement. **UNKNOWN** means it was not verified. REA ran no updates and did not execute the app.

## 1. Conclusion

- T3 Code uses **electron-updater 6.8.9** with the **GitHub provider**. GitHub Releases serves as the update server: no backend, no CDN.
- Channels are git tags plus per-channel YAML manifests (`nightly-linux.yml`, `nightly.yml`, …) attached to each release. Nightly builds are GitHub _prereleases_. Stable builds use `make_latest=true`.
- The client checks 15 s after startup and then every 4 min. Download and install are **explicit, user-driven steps**, because `autoDownload` and `autoInstallOnAppQuit` are both `false`. The client verifies each download with the manifest's SHA-512.
- For openpiso Admin (Tauri v2, Windows, NSIS) the equivalent is **`tauri-plugin-updater`**, with a static `latest.json` on GitHub Releases. Tauri uses a **minisign** signature on every artifact (stronger than T3's checksum-only Linux path) and a **GitHub Actions** release workflow that copies T3's cron-gated nightly and its tag-triggered stable release.
- Main caveat: this is openpiso's **first internet egress dependency**. Make it opt-in and admin-initiated, and keep a fully offline update path (sideload the installer) for the LAN-only and diskless deployments.

## 2. T3 Code: the client mechanism

### 2.1 Feed configuration (OBSERVED)

`resources/app-update.yml` inside the AppImage:

```yaml
owner: pingdotgg
repo: t3code
provider: github
releaseType: prerelease
channel: nightly
updaterCacheDirName: t3code-updater
```

The build writes this file through `resolveGitHubPublishConfig`, which returns `{provider:"github", owner, repo, releaseType: nightly ? "prerelease" : "release", channel: "nightly" if nightly}`.

### 2.2 How the feed resolves (DERIVED)

1. GitHubProvider reads the repository's releases Atom feed.
2. Nightly: it picks the newest matching prerelease tag. Stable: it uses `/releases/latest`.
3. It downloads `/releases/download/<tag>/<channel><platform-suffix>.yml`, e.g. `nightly-linux.yml`, `nightly-linux-arm64.yml`, `nightly-mac.yml`, or `nightly.yml` (Windows).
4. It compares the manifest `version` with the running version.
5. It downloads the asset named in `path`/`files` and verifies `sha512` (otherwise `ERR_CHECKSUM_MISMATCH`).

Published manifest (OBSERVED, verbatim; the AppImage size matches the local file):

```yaml
version: 0.0.46-nightly.20261013.3042
files:
  - url: T3-Code-0.0.46-nightly.20261013.3042-x86_64.AppImage
    sha512: SCXnx/ghHomINYVcmAVI+e9ZN+06cmTfhj3X5vuhYgFtKgtAcQFuQFWy+fg8gbxbTONPyNNAO60V/5Vf9LONgw==
    size: 164762524
    blockMapSize: 174496
  - url: T3-Code-0.0.46-nightly.20261013.3042-amd64.deb
    sha512: Xk3I9cbiwANHQqpnsUEejuKXSYq8SzpEvITst55yeeGcf+I9l83qyjnW3vK1CdQX+VPT6CpzWhvBIvzlIkVbKQ==
    size: 139186396
path: T3-Code-0.0.46-nightly.20261013.3042-x86_64.AppImage
sha512: SCXnx/ghHomINYVcmAVI+e9ZN+06cmTfhj3X5vuhYgFtKgtAcQFuQFWy+fg8gbxbTONPyNNAO60V/5Vf9LONgw==
releaseDate: "2026-10-13T01:24:48.834Z"
```

### 2.3 Lifecycle (OBSERVED in `dist-electron/main.cjs`)

| Step                | Behaviour                                                                                                                                      | Location                        |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------- |
| Configure           | After Electron is ready: `setChannel`, `setAllowPrerelease`, `setFeedURL`                                                                      | `ElectronUpdater.ts`            |
| Defaults overridden | `setAutoDownload(false)`, `setAutoInstallOnAppQuit(false)`                                                                                     | main.cjs:141846–141847          |
| Check               | 15 s after start, then every 4 min, plus the menu "Check for Updates..." and renderer requests                                                 | main.cjs:141284, 141673, 157993 |
| Download            | Explicit IPC `desktop:update-download`; progress broadcast at 10 % steps                                                                       | main.cjs:141115+                |
| Install             | Validate state → mark quitting → write restart marker → stop backends (5 s timeouts) → `quitAndInstall({isSilent:true, isForceRunAfter:true})` | main.cjs:141587–141619          |
| Recover             | On failure: clear quitting, remove marker, restart backends, keep the download for retry                                                       | same                            |

State machine: `disabled → idle → checking → available | up-to-date → downloading → downloaded → (install)`, with `error` reachable from any step. State carries the current/available version, channel, release notes, percentage, timestamps, error context and retry eligibility.

### 2.4 Channel switching (OBSERVED)

- Settings offers **Stable** (`latest`) and **Nightly** (`nightly`).
- Default: `resolveDefaultDesktopUpdateChannel`. A version with `-nightly.<date>.<run>` selects nightly; anything else selects latest. An explicit choice is persisted.
- A switch resets state and checks at once, with downgrade allowed temporarily. Nightly turns on prereleases, downgrade and the full changelog.
- The client rejects offers from the wrong track.

### 2.5 IPC and UI (OBSERVED; runtime registration INFERRED)

Preload channels: `desktop:update-get-state`, `desktop:update-set-channel`, `desktop:update-check`, `desktop:update-download`, `desktop:update-install`, and the `desktop:update-state` push. Main validates each request and result against named schemas. The UI shows progress and release notes, an "Update downloaded" toast, and a restart confirmation that warns running tasks will be interrupted.

### 2.6 Platforms and integrity

| Platform       | Update artifact                                              | Differential                      | Signature check                                       |
| -------------- | ------------------------------------------------------------ | --------------------------------- | ----------------------------------------------------- |
| Linux AppImage | AppImage (needs `APPIMAGE` env)                              | Embedded blockmap, full fallback  | None. SHA-512 checksum only                           |
| macOS          | **ZIP** (not DMG), through Squirrel.Mac and a loopback proxy | Yes (off under Rosetta migration) | OS code signing, if the release is signed             |
| Windows        | NSIS EXE + sidecar `.blockmap`                               | Yes, full fallback                | Authenticode, only when `publisherName` is configured |

UNKNOWN: whether the macOS/Windows releases are actually signed (the signing secrets are optional and skipped when absent), real install success, and installed permissions. A checksum is not a publisher signature.

## 3. T3 Code: the GitHub pipeline (OBSERVED in the repo)

- **Version and tag:** `scripts/resolve-nightly-release.ts` sets base = `apps/desktop/package.json` version with patch +1. The tag is `v<base>-nightly.<YYYYMMDD>.<runNumber>`, e.g. `v0.0.46-nightly.20261013.3042`. Stable is `v0.0.46`.
- **Triggers** (`.github/workflows/release.yml`): cron `8,38 * * * *`, a push of tag `v*.*.*` (excluding `-nightly`/`-preview`), and `workflow_dispatch`.
- **Gate** (`.github/scripts/check-nightly-release.cjs`): publish only when it is ≥ 6 h since the last nightly **and** new commits exist. Concurrency group `release-nightly`.
- **Stable** builds the commit of the latest nightly (`resolveLatestNightlyCommit`), so stable is a promoted, already-tested nightly.
- **Build matrix** (`release-desktop.yml`): mac arm64/x64, linux x64/arm64, win x64/arm64. `scripts/build-desktop-artifact.ts` runs `electron-builder … --publish never`. The config is generated in TS (`createBuildConfig`); there is no `electron-builder.yml`.
- **Manifest merge:** `scripts/merge-update-manifests.ts` merges the per-arch mac/win manifests into one (`release.yml:846–871`).
- **Upload:** `softprops/action-gh-release` (`release.yml:913–942`). Nightly and preview: `prerelease: true`, `make_latest: false`. Stable: `prerelease: false`, `make_latest: true`.
- **Assets per release:** installers, blockmaps, CLI tarballs, `SHA256SUMS`, the channel manifests and `builder-debug.yml`. Old nightlies are not pruned.
- **Signing secrets (all optional):** macOS `CSC_LINK`, `CSC_KEY_PASSWORD`, `APPLE_API_KEY/_ID/_ISSUER`, `APPLE_TEAM_ID`. Windows uses Azure Trusted Signing `AZURE_*`. Linux is unsigned.

## 4. Mapping to openpiso Admin

### 4.1 Current state (OBSERVED in openpiso)

- `apps/admin` ("OpenPiso Central Terminal"): Tauri v2 + Svelte 5, Rust crate `openpiso-admin`, Windows-only (`x86_64-pc-windows-msvc`).
- The Kiosk is a Tauri window labelled `kiosk` (fullscreen, no decorations, always on top) at `tauri.conf.json:13–27`.
- The web-admin is a static Svelte build embedded as a resource (`tauri.conf.json:34–36`). The Admin process serves it at `http://<server>:8080/admin`. The Admin process also serves the `:3210` client API + SSE (`web_admin.rs:303`).
- `bundle.active = false` (`tauri.conf.json:32–33`). `scripts/build-installers.ts` builds NSIS with cargo-xwin/makensis into `dist/installers/admin-web-admin/`.
- Lockstep versioning via `scripts/bump-version.ts` (now 0.11.4). There is no `.github/`, no tags, no updater and no signing. The remote is `github.com/H3atman/openpiso`.
- `tauri-plugin-single-instance` and `tauri-plugin-autostart` (HKCU Run, ADR-0055) are already present.

### 4.2 Concept mapping

| T3 Code (Electron)                            | openpiso Admin (Tauri v2)                                                                     |
| --------------------------------------------- | --------------------------------------------------------------------------------------------- |
| electron-updater 6.8.9                        | `tauri-plugin-updater` (Rust side + optional JS)                                              |
| `app-update.yml` (provider: github)           | `plugins.updater.endpoints` in `tauri.conf.json`                                              |
| `nightly-*.yml` / `latest*.yml` manifests     | `latest.json` (one per channel)                                                               |
| SHA-512 checksum (+ optional Authenticode)    | **minisign signature required** (`.sig` per artifact, pubkey in config)                       |
| NSIS EXE + blockmap                           | NSIS setup EXE (no differential download in Tauri)                                            |
| `releaseType: prerelease` / `make_latest`     | Same GitHub flags; nightly uses a fixed moving tag                                            |
| `setAutoDownload(false)`                      | Call `check()`, then `download()` / `install()` only on admin action                          |
| `quitAndInstall({isSilent, isForceRunAfter})` | `installMode: "passive"` + NSIS relaunch, or `app.restart()`                                  |
| IPC `desktop:update-*`                        | Tauri commands `update_get_state/check/download/install/set_channel` + `update://state` event |
| Settings "Stable / Nightly"                   | web-admin Settings → Updates panel                                                            |
| `release.yml` + cron gate                     | `.github/workflows/release.yml` on `windows-latest`                                           |
| `resolve-nightly-release.ts`                  | Extend `scripts/bump-version.ts` with a `--nightly` stamp                                     |

### 4.3 Feed design (GitHub only)

Tauri needs one static JSON per channel:

```json
{
  "version": "0.11.5",
  "notes": "…",
  "pub_date": "2026-10-13T01:24:48Z",
  "platforms": {
    "windows-x86_64": {
      "signature": "<contents of OpenPiso-Admin_0.11.5_x64-setup.exe.sig>",
      "url": "https://github.com/H3atman/openpiso/releases/download/v0.11.5/OpenPiso-Admin_0.11.5_x64-setup.exe"
    }
  }
}
```

Endpoints:

- **Stable:** `https://github.com/H3atman/openpiso/releases/latest/download/latest.json`. GitHub resolves `latest` to the newest non-prerelease, non-draft release, which is what T3's `make_latest=true` relies on.
- **Nightly:** `https://github.com/H3atman/openpiso/releases/download/nightly/latest.json`. Keep one prerelease with the fixed tag `nightly` and overwrite its assets on each build. This avoids the Atom-feed scan that electron-updater does and that Tauri lacks. Optional: also publish an immutable `v0.11.5-nightly.20261013.N` prerelease for history.

Pick the channel at runtime. Tauri v2 lets Rust build the updater with `app.updater_builder().endpoints(vec![url])`, so the persisted channel setting selects the URL. Use T3's rule: the default channel follows the version string (`-nightly.` → nightly), and an explicit choice overrides it.

Version format: Tauri compares SemVer. `0.11.5-nightly.20261013.42` sorts **below** `0.11.5`, so a nightly user moves to stable once the stable release ships. To move from nightly back to an older stable, allow downgrade the way T3 does: use a custom `version_comparator` that accepts any different version when the channel has just changed.

### 4.4 Signing

| Layer                    | Required?                               | How                                                                                                                                                                                            |
| ------------------------ | --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Tauri updater (minisign) | **Yes**: Tauri refuses unsigned updates | `bun tauri signer generate`. Put the private key and password in GH secrets `TAURI_SIGNING_PRIVATE_KEY` / `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`. Put the public key in `plugins.updater.pubkey` |
| Windows Authenticode     | Optional (SmartScreen reputation only)  | Later: Azure Trusted Signing, same as T3 (`.agents/skills/signing-tauri-apps`)                                                                                                                 |

Store the private key offline as well. Losing it strands every installed Admin, because it can no longer accept updates.

### 4.5 Windows install path for the Admin + Kiosk

1. The admin presses **Install** in web-admin, or confirms in the Kiosk.
2. Rust checks that no paid session is active (counterpart of T3's "running tasks will be interrupted" warning). Refuse, or schedule for idle time.
3. Rust stops the `:3210` client API and the `:8080` web-admin server cleanly, and tells clients over SSE that the server is restarting (T3's "stop backends with timeouts").
4. Rust writes a restart marker (version + timestamp) in the app data directory.
5. `update.download_and_install()` with `windows.installMode = "passive"`. The NSIS installer closes the running Admin, replaces files and relaunches.
6. On start, Rust reads the marker. On success it reports the new version (`telemetry.rs:93` already sends `admin_version` locally) and deletes the marker. If the version did not change, it surfaces an error state and keeps the downloaded installer for retry.

Pitfalls to design for:

- **single-instance:** the relaunched process must not be rejected by the old process. Exit fully before NSIS relaunches; passive mode closes the app first.
- **autostart (HKCU Run):** keep the install path stable (per-user vs per-machine must not change between versions), or the Run entry breaks.
- **Kiosk always-on-top:** NSIS passive UI may hide behind the kiosk. Hide the kiosk window, or use `quiet` mode with your own progress screen.
- **Client Guard service** (SYSTEM): out of scope for phase 1. The Client app keeps its manual path (`docs/installation.md:220–241`).
- **Diskless master images:** an in-place update on a diskless node is lost at reboot. Only the Admin/server machine gets in-app updates. Diskless images keep the master-image workflow (`docs/diskless.md`).

### 4.6 Authority and the local-first scope

- **Network:** the updater is openpiso's first outbound internet call (`README.md:107–115` says LAN-first, no cloud without approval). Make it **off by default**, a setting the operator enables, and **check only on demand** or at most daily. Do not copy T3's every-4-minutes check.
- **Admin endpoints (ADR-0024):** expose check/download/install only as Tauri commands to the local window, plus web-admin routes that require the existing local admin auth. Never expose them on the public client API (`:3210`).
- **Rust owns authority:** the Svelte UI only displays state and sends intents. Rust verifies, gates on session state and installs.
- **Offline path:** keep `dist/installers/admin-web-admin/*.exe` installable by hand. Optionally let web-admin accept an uploaded installer plus `.sig` and verify it with the same pubkey. This gives offline shops the same signed path.

## 5. GitHub Actions spec (Admin only)

`.github/workflows/release.yml`:

```yaml
name: release
on:
  schedule: [{ cron: "17 */6 * * *" }] # nightly candidate
  push: { tags: ["v*.*.*", "!v*-nightly*"] } # stable
  workflow_dispatch:
concurrency: { group: release-$, { { github.ref } }, cancel-in-progress: false }
permissions: { contents: write }

jobs:
  gate: # nightly only: skip unless new commits since the "nightly" tag
    runs-on: ubuntu-latest
    outputs:
      {
        publish: $,
        { { steps.g.outputs.publish } },
        version: $,
        { { steps.g.outputs.version } },
      }
  build-admin:
    needs: gate
    if: needs.gate.outputs.publish == 'true'
    runs-on: windows-latest
    env:
      TAURI_SIGNING_PRIVATE_KEY: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY }}
      TAURI_SIGNING_PRIVATE_KEY_PASSWORD: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY_PASSWORD }}
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - uses: dtolnay/rust-toolchain@stable
      - run: bun install --frozen-lockfile
      - run: bun run scripts/bump-version.ts --set ${{ needs.gate.outputs.version }} # in CI only
      - run: bun run --cwd apps/web-admin build
      - uses: tauri-apps/tauri-action@v0
        with:
          projectPath: apps/admin
          tagName: ${{ github.event_name == 'push' && github.ref_name || 'nightly' }}
          releaseName: OpenPiso Admin ${{ needs.gate.outputs.version }}
          prerelease: ${{ github.event_name != 'push' }}
          includeUpdaterJson: true
          updaterJsonPreferNsis: true
```

Notes:

- `tauri-action` builds, uploads the NSIS installer and `.sig`, and generates `latest.json`. That replaces T3's custom build script, manifest merge and `softprops` upload.
- Native Windows runners replace cargo-xwin, so local `build-installers.ts` stays for offline builds only.
- The gate copies `check-nightly-release.cjs`: compare `HEAD` with the commit of the `nightly` tag. Optionally add a minimum gap. One run every 6 h already gives the gap.
- Stable copies T3's promotion model. Tag the commit that the current nightly was built from (`git tag v0.11.5 <nightly-sha>`), so stable equals a tested nightly.
- Mark stable releases `make_latest: true` (the default for non-prereleases). The `nightly` release must stay `prerelease: true`, or `releases/latest` would point at it.

## 6. Phased plan

| Phase                 | Scope                                                                                                                                                       | Exit check                                                                                                |
| --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| 0. Prerequisites      | Enable `bundle.active` (NSIS target, `createUpdaterArtifacts: true`), generate the minisign keypair, add GH secrets, decide per-user vs per-machine install | Local `tauri build` emits `*-setup.exe` + `.sig`                                                          |
| 1. Pipeline           | `release.yml` with tauri-action, stable on tag, manual nightly                                                                                              | A GitHub release has `latest.json` with a valid signature URL                                             |
| 2. Client core (Rust) | `tauri-plugin-updater`, `UpdateState` machine (T3's states), commands, `update://state` event, opt-in setting, channel URL selection                        | Install 0.11.5 → app offers 0.11.6 from GitHub, verifies the signature, installs passively and relaunches |
| 3. Kiosk/web-admin UX | Settings → Updates (channel, check, release notes, progress, Install), Kiosk banner, active-session gate, client SSE notice                                 | An operator updates from the browser at `:8080/admin` without touching the kiosk                          |
| 4. Nightly            | Cron + gate, moving `nightly` prerelease, channel switch with downgrade                                                                                     | A nightly-channel machine follows nightlies; switching to stable returns to the latest stable             |
| 5. Later              | Authenticode, offline signed-upload path, Client app + Client Guard service                                                                                 | Separate spec                                                                                             |

## 7. Evidence and REA tools used

REA evidence retained in `/tmp/rea-t3-updater-evidence/`:

| Evidence                                                              | What                                                                |
| --------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `ev_e9ff46cac18af1139edfc0a0bc4a83f72b0db080e4c146ebe7d665aa56f61463` | AppImage inventory (`inspect_artifact`)                             |
| `ev_98ab412fdd8eba9c5eeeef880201cc008b8c337ff5ec0424ca41b27b916f29cc` | Preload JS application graph. MCP and CLI produced the identical ID |
| `ev_3b2753b0a560cc684878fd349a1615e790140ea12b1571c57e4e3280814ae810` | MCP semantic trace of the install path                              |
| `ev_9ff4d8a84970d9d8592c579ee15b2ae80e17f20669da29c83812e774ae724c22` | CLI semantic trace of the download path                             |

| REA tool                                             | Interface | Result                                                                                                       |
| ---------------------------------------------------- | --------- | ------------------------------------------------------------------------------------------------------------ |
| `open_binary`, `inspect_artifact`                    | MCP       | ok: AppImage inventory                                                                                       |
| `analyze_javascript_application` (scoped to preload) | MCP + CLI | ok: same Evidence ID through both adapters                                                                   |
| `trace_javascript_semantics`                         | MCP + CLI | ok, partial: reaches the download/install callbacks; the Electron `ipcRenderer.invoke` edge stays unresolved |
| `export_evidence_bundle`, `close_binary`             | MCP       | ok                                                                                                           |
| `analyze_javascript_application` (whole app.asar)    | CLI / MCP | **failed**: JS heap out of memory; the MCP connection closed                                                 |
| `inspect_binary_layout` (AppImage)                   | MCP       | **rejected**: file size over limit. Inventory showed only the ELF runtime, not the embedded SquashFS         |

Main-process and UI findings come from direct inspection of the packaged text (`dist-electron/main.cjs` line references above). The REA Evidence covers the preload bridge. The pipeline facts come from reading `pingdotgg/t3code` and the published `nightly-linux.yml`.

REA follow-ups found during this work:

- Whole-ASAR analysis of a large Electron app runs out of memory. Needs streaming or module-scoped analysis.
- AppImage inventory should descend into the embedded SquashFS payload (offset after the ELF runtime).

## 8. Open questions

- Per-user or per-machine install for Admin? This decides the NSIS `installMode` and whether UAC appears during an update.
- Should updates be allowed while a paid session is running, or only at idle?
- Is a public GitHub repo acceptable for the release assets? A private repo needs a token in the client, which breaks the "no secrets on the client" property.
- Should nightly be exposed to shops at all, or only to the maintainer's test machine (like T3's "preview" channel)?

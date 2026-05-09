# Rename desktop app: Voquill → Verbatim

## Context

The product is rebranding from Voquill to Verbatim. Existing users on macOS, Windows, and Linux must experience the rename as a normal auto-update — they install the next release, and the next time they look at the Dock / Start Menu / Activities, the app is named "Verbatim". All their data, models (multi-GB), settings, mic permissions, API keys, and dictation history must remain intact and usable. This plan covers the **desktop app only**. Mobile is out of scope. Domain, Firebase, GitHub org, and `@voquill/*` npm scope are out of scope.

**Core architectural decision: keep the bundle identifier `com.voquill.desktop` (and all variants) unchanged forever.** The bundle ID is the OS's primary key for "this app" — it controls app data dirs, keychain access, TCC permissions (mic/accessibility), preferences, file associations, and uninstaller registry entries. Changing it would force every existing user through a multi-GB data migration on first launch, re-prompt for microphone/accessibility permissions on macOS, and orphan keychain entries. Keeping it means the rename is invisible to the OS — only `productName`, the binary name, the visible `.app` directory name, and packaging metadata change. The string "voquill" survives in `~/Library/Application Support/`, registry paths, and apt's package-relations metadata, where users never look. Cosmetic compromise; zero migration risk.

**Updater endpoints, git tags, GitHub release tags, and the updater pubkey all stay stable.** Existing apps in the wild are configured to poll specific URLs (e.g., `https://github.com/voquill/voquill/releases/download/desktop-prod/latest.json`) and verify against a specific pubkey. Changing any of those breaks the update path for users still on the old build. We rebrand *what* gets published; we don't move *where* it gets published.

## Strategy summary

| Layer | What changes | What stays |
|---|---|---|
| Visible UI / window titles / Dock / Start Menu | "Voquill" → "Verbatim" | — |
| `.app` directory name on macOS | `/Applications/Voquill.app` → `/Applications/Verbatim.app` (via first-launch self-relocation) | — |
| Tauri `productName`, `mainBinaryName` | renamed | — |
| Cargo package + binary name | `Voquill` → `Verbatim` | — |
| `macos/Voquill.entitlements` filename | renamed to `Verbatim.entitlements` | — |
| Linux deb/rpm package name | `voquill-desktop` → `verbatim-desktop` (with `Replaces`/`Conflicts`/`Provides`/`Obsoletes`) | apt repo, rpm repo URLs |
| Windows installer | NSIS `pre-install` hook silently uninstalls prior Voquill registration | install path is overwritten by hook then new install runs |
| Homebrew cask | `voquill-desktop` → `verbatim-desktop` (with `old_names`) | tap repo URL |
| AUR PKGBUILD | `voquill-bin` → `verbatim-bin` (with `replaces`/`conflicts`) | — |
| Release artifact filenames (DMG/PKG/MSI/deb/rpm) | "Voquill_*" → "Verbatim_*" | — |
| Diagnostic zip filenames (user-visible) | `voquill-*.zip` → `verbatim-*.zip` | — |
| **Bundle identifier** | unchanged | `com.voquill.desktop` and all variants |
| **Updater pubkey + endpoint URLs** | unchanged | latest.json URLs, GitHub release tags |
| **SQLite DB filename** | unchanged | `voquill.db` (lives inside `~/Library/Application Support/com.voquill.desktop/`) |
| **App data directories** | unchanged | all platforms |
| **Git tags, channel release tags** | unchanged | `desktop-prod-v*`, `desktop-prod`, etc. |
| **Firebase, voquill.com, GitHub org, @voquill npm scope** | unchanged | — |

## Implementation phases

### Phase 1 — Tauri config and Rust package

Files:
- `apps/desktop/src-tauri/tauri.conf.json` — `productName: "Verbatim"`, `mainBinaryName: "Verbatim"`, window title `"Verbatim"`. Update `bundle.macOS.entitlements` reference to renamed entitlements file. **Do not touch `identifier`.**
- `apps/desktop/src-tauri/tauri.dev.conf.json` — `productName: "Verbatim (dev)"`, etc.
- `apps/desktop/src-tauri/tauri.prod.conf.json` — same as base, `mainBinaryName: "Verbatim"`.
- `apps/desktop/src-tauri/tauri.enterprise.conf.json` — `productName: "Verbatim Enterprise"`, `mainBinaryName: "verbatim-desktop-enterprise"`.
- `apps/desktop/src-tauri/tauri.enterprise-dev.conf.json` — corresponding enterprise-dev variants.
- `apps/desktop/src-tauri/tauri.local.conf.json` — `productName: "Verbatim (local)"`.
- `apps/desktop/src-tauri/Cargo.toml` — `name = "Verbatim"`, `default-run = "Verbatim"` (must match new `mainBinaryName`).
- `apps/desktop/src-tauri/Info.plist` — replace "Voquill" strings.
- `apps/desktop/src-tauri/macos/Voquill.entitlements` — **rename file** to `Verbatim.entitlements`. Contents unchanged.
- `apps/windows-installer/src-tauri/tauri.conf.json` — `productName: "Verbatim Installer"`, identifier stays `com.voquill.installer`.

**Critical rule across all configs:** `identifier` fields stay byte-for-byte identical. Run `grep -r '"identifier"' apps/desktop/src-tauri apps/windows-installer/src-tauri` after editing to verify nothing under `identifier` mentions "verbatim".

### Phase 2 — macOS `.app` self-relocation (first-launch one-shot)

After auto-update on macOS, Tauri's updater replaces the `.app` *contents* in place — `/Applications/Voquill.app/` on disk still has that directory name even though `CFBundleName` inside is now "Verbatim". Finder shows "Verbatim", but the on-disk path is forensically wrong. Fix: ship a one-shot self-relocation in the Rust entry point.

Add to `apps/desktop/src-tauri/src/lib.rs` (or wherever `tauri::Builder::default()` is constructed), gated to macOS, run **before** `Builder::default()`:

```rust
#[cfg(target_os = "macos")]
fn relocate_app_bundle_if_needed() {
    use std::path::PathBuf;
    use std::process::Command;

    let Ok(exe) = std::env::current_exe() else { return; };
    // exe path: /Applications/Voquill.app/Contents/MacOS/Verbatim
    let bundle = exe.ancestors().nth(3).map(PathBuf::from);
    let Some(bundle) = bundle else { return; };
    let Some(parent) = bundle.parent() else { return; };
    let Some(name) = bundle.file_name().and_then(|s| s.to_str()) else { return; };

    if name != "Voquill.app" { return; }                  // already renamed or non-standard install
    if parent != std::path::Path::new("/Applications") { return; } // user-installed elsewhere; leave alone

    let new_path = parent.join("Verbatim.app");
    if new_path.exists() { return; }                      // orphan from prior attempt; bail, log

    if std::fs::rename(&bundle, &new_path).is_err() { return; }
    let _ = Command::new("/usr/bin/open").arg(&new_path).spawn();
    std::process::exit(0);
}
```

Edge cases handled:
- User installed to a non-standard path (e.g., `~/Applications`) — bail, leave `.app` named whatever.
- `Verbatim.app` already exists in `/Applications/` — bail (orphan from a prior failed attempt; log via the existing logging facility but don't disrupt launch).
- Rename fails (permissions, file in use) — bail silently; user still has a working "Verbatim" app, just with a Voquill.app directory name. They can manually rename it if it bothers them.

Quarantine attribute: macOS may add `com.apple.quarantine` xattr after the rename (treating it like a freshly-moved app). Test whether re-launching from `/Applications/Verbatim.app` triggers a Gatekeeper prompt. If so, copy the xattr from the old path with `xattr -cr` or use `MDItemRemove` before relaunch.

### Phase 3 — Windows NSIS pre-install cleanup

Add a custom NSIS hook that detects an existing Voquill install (via the uninstaller's registered key) and runs its uninstaller silently before the new install proceeds.

Tauri 2 supports `bundle.windows.nsis.installerHooks` pointing to an `.nsh` file. Create `apps/desktop/src-tauri/nsis/installer-hooks.nsh`:

```nsh
!macro NSIS_HOOK_PREINSTALL
  ; Read the uninstall string for the prior Voquill install
  ; The Tauri NSIS uninstaller is registered under HKCU under productName_appId
  ReadRegStr $0 HKCU "Software\Microsoft\Windows\CurrentVersion\Uninstall\com.voquill.desktop" "QuietUninstallString"
  ${If} $0 != ""
    ExecWait '$0'
  ${EndIf}
  ; Same for the older productName-based key, if Tauri used a different scheme historically
  ReadRegStr $0 HKCU "Software\Microsoft\Windows\CurrentVersion\Uninstall\Voquill" "QuietUninstallString"
  ${If} $0 != ""
    ExecWait '$0'
  ${EndIf}
!macroend
```

**Research item before implementing:** confirm the exact registry path Tauri 2's NSIS template uses. Tauri's templates have varied between `Software\Microsoft\Windows\CurrentVersion\Uninstall\{identifier}` and `\{productName}`. Inspect a current Voquill install on a Windows machine (or the NSIS template at `~/.cargo/registry/.../tauri-bundler/.../main.nsi`) to confirm. The hook above tries both common variants — if neither matches, add the discovered key.

Reference the hook in `tauri.prod.conf.json` and `tauri.enterprise.conf.json`:
```json
"bundle": {
  "windows": {
    "nsis": {
      "installerHooks": "./nsis/installer-hooks.nsh"
    }
  }
}
```

The user data at `%APPDATA%\com.voquill.desktop\` is **not** touched by the old uninstaller (Tauri uninstallers leave user data by default). After the new install completes, Verbatim launches against the same `%APPDATA%\com.voquill.desktop\` directory, finds the existing DB and models, and proceeds normally. Confirmed by inspecting the NSIS template's uninstaller section.

### Phase 4 — Linux deb/rpm clean replacement

`.deb` (apt):
- Tauri 2's `bundle.linux.deb` config exposes `depends`, `section`, `priority` but **does not natively support `Replaces`/`Conflicts`/`Provides`** in the control file. Workaround: post-bundle CI step.
- Add a step in `_release-desktop-impl.yml` after the deb is produced:
  ```bash
  dpkg-deb -R "$DEB_PATH" deb-extracted
  printf 'Replaces: voquill-desktop\nConflicts: voquill-desktop\nProvides: voquill-desktop\n' >> deb-extracted/DEBIAN/control
  dpkg-deb -b deb-extracted "$REPACKED_DEB_PATH"
  ```
  Then upload `$REPACKED_DEB_PATH` to the apt repo instead of the original.
- `apt upgrade` on a machine running `voquill-desktop` will replace it with `verbatim-desktop` cleanly. The user data dir (`~/.config/com.voquill.desktop/`) is owned by the user, not the package, so it survives.

`.rpm` (dnf/yum):
- Add `Obsoletes: voquill-desktop` to the rpm spec. Tauri 2's `bundle.linux.rpm` may support `obsoletes` directly — **research item: confirm field name in Tauri 2 docs**. If not, post-bundle hook with `rpm --addsign` workflow already exists; insert a spec-edit step similarly.
- `dnf upgrade` cleanly replaces.

`AppImage`:
- AppImage is filename-based — users download one file. Rename `Voquill_*.AppImage` → `Verbatim_*.AppImage` in artifact-naming logic. Old AppImage files on user's disk continue to work but won't auto-update past the rename release without one final updater push (covered by phase 6).

### Phase 5 — Homebrew cask + AUR

Homebrew cask (in tap repo `voquill/homebrew-voquill`):
- The release workflow currently writes `Casks/voquill-desktop.rb` and `Casks/voquill-desktop-dev.rb`. Update workflow generation to:
  - Write the new cask at `Casks/verbatim-desktop.rb` with `name "Verbatim"`, the new DMG URL, and `old_names ["voquill-desktop"]`.
  - Replace the old cask file's contents with a deprecation stub: `deprecate! date: "<release-date>", because: :renamed` and a `caveats` block telling users `brew upgrade` will handle migration.
- Homebrew's `old_names` mechanism makes `brew upgrade voquill-desktop` resolve to `verbatim-desktop` automatically. The `deprecate!` directive handles users who explicitly type `brew install voquill-desktop`.
- Workflow location: `_release-desktop-impl.yml` lines ~1081–1166 (the Homebrew tap publish step).

AUR PKGBUILD (`apps/desktop/src-tauri/archlinux/PKGBUILD`):
- Rename package: `pkgname=verbatim-bin` (was `voquill-bin`).
- Update download URL to point to `Verbatim_*.deb` artifact.
- Add `replaces=('voquill-bin')` and `conflicts=('voquill-bin')`.
- The AUR is community-maintained; pushing the rename requires a separate AUR repo update. If the maintainer is the team, schedule a coordinated push.

### Phase 6 — Release workflow updates

`.github/workflows/_release-desktop-impl.yml`:
- DMG/PKG naming logic (lines ~658–714): replace `Voquill_` / `VoquillDev_` / `VoquillEnterprise_` / `VoquillEnterpriseDev_` prefixes with `Verbatim_` / `VerbatimDev_` / `VerbatimEnterprise_` / `VerbatimEnterpriseDev_`.
- Linux package name vars (lines ~569–597): `voquill-desktop*` → `verbatim-desktop*`.
- Windows portable installer EXE name (lines ~839, 862, 869): `voquill-installer.exe` → `verbatim-installer.exe`, signed output `Voquill_Portable_Installer.exe` → `Verbatim_Portable_Installer.exe`.
- Insert the deb-control-injection step described in Phase 4.
- Insert the rpm-spec edit step (or use Tauri config field if supported).
- Update Homebrew cask generation per Phase 5.
- **Do not change**: git tag patterns (`desktop-prod-v*` etc.), channel release tags (`desktop-prod`, `desktop-dev`, `desktop-enterprise`, `desktop-enterprise-dev`), latest.json URL paths, updater pubkey env var name, GitHub repo URLs.

`.github/workflows/build-desktop.yml`:
- Artifact upload name (line ~146): `voquill-${{ matrix.tauri_platform }}` → `verbatim-${{ matrix.tauri_platform }}`. Internal CI artifact name; doesn't affect end users.

`.github/workflows/release.yml`:
- Google Chat notification text (line ~221): `🎙️ *Voquill*` → `🎙️ *Verbatim*`.

### Phase 7 — User-visible filenames in code

`apps/desktop/src-tauri/src/commands.rs` (and any other Rust referencing log/diagnostic filenames):
- `voquill-diagnostics.zip` → `verbatim-diagnostics.zip`
- `voquill-{short_id}.zip` → `verbatim-{short_id}.zip`
- `.voquill_write_probe` temp file prefix → `.verbatim_write_probe`

These are user-visible (the user picks where to save the diagnostic zip; they see the filename). The DB filename `voquill.db` is **not** in this category — it lives inside `~/Library/Application Support/com.voquill.desktop/` and users almost never look there. Renaming it would force a SQLite migration we don't need. Leave `voquill.db` alone.

Search for any remaining user-visible "Voquill" strings in TS/React: window titles in components, About dialogs, error messages, support email subjects. Run `grep -rni voquill apps/desktop/src apps/desktop/src-tauri/src` and review each hit. Translation files (`packages/i18n` or similar) — update default messages.

### Phase 8 — Verification

End-to-end test plan, to be run on each platform before publishing the rename release to the prod channel:

1. **macOS update path**:
   - Install current production `Voquill.app` (e.g., `desktop-prod-v<current>`).
   - Launch, sign in, dictate a few times to populate DB and models.
   - Push a release to `desktop-dev` channel (or a private test channel) with the rename changes.
   - Open the installed app, trigger update via the in-app updater.
   - **Expected**: app downloads, restarts; window/Dock/menu bar now show "Verbatim". On the *next* launch after the first one (because relocation triggers `exit(0)`), the `.app` is at `/Applications/Verbatim.app`. Mic permission, accessibility permission, dictation history, models, API keys all intact.
   - Verify `~/Library/Application Support/com.voquill.desktop/` is unchanged (same DB, same models).
   - Verify Spotlight finds "Verbatim" within ~30 seconds.

2. **Windows update path**:
   - Install current production `Voquill_<v>_x64-setup.exe`.
   - Populate DB.
   - Update via in-app updater (which downloads new NSIS installer and runs it).
   - **Expected**: pre-install hook silently uninstalls old; new "Verbatim" install completes; app launches as "Verbatim"; Add/Remove Programs shows only "Verbatim". `%APPDATA%\com.voquill.desktop\` unchanged.

3. **Linux apt update path**:
   - On a Debian/Ubuntu VM, `apt install voquill-desktop` from the prod apt repo.
   - Populate DB.
   - Push the new `verbatim-desktop` deb to the apt repo.
   - `apt update && apt upgrade` on the VM.
   - **Expected**: apt prompts (or auto-applies) the replacement of `voquill-desktop` by `verbatim-desktop`. `~/.config/com.voquill.desktop/` unchanged. Launch verifies app loads existing data.

4. **Linux rpm update path**: parallel test with `dnf upgrade`.

5. **Homebrew cask path**:
   - `brew install voquill/voquill/voquill-desktop` on a Mac with no prior install.
   - After publishing the rename: `brew upgrade` should resolve via `old_names` and replace with `verbatim-desktop`. App launches as "Verbatim".

6. **AUR path**: `yay -Syu` on Arch with `voquill-bin` installed pulls `verbatim-bin` and replaces.

7. **Smoke check on a fresh-install machine** (no prior Voquill at all): install Verbatim from the new DMG/MSI/deb/rpm. Sign in, dictate, verify everything works. This catches any latent assumption that the app data dir was created by a prior Voquill install.

8. **First-launch relocation edge cases on macOS**:
   - User installed to `~/Applications/Voquill.app` (non-standard) — verify the relocation bails and the app still launches.
   - `/Applications/Verbatim.app` already exists (orphan) — verify bail and launch from current location.
   - User's `/Applications` is read-only or the user is non-admin — verify bail and launch.

## Files to modify

**Tauri/Rust config**:
- `apps/desktop/src-tauri/tauri.conf.json`
- `apps/desktop/src-tauri/tauri.dev.conf.json`
- `apps/desktop/src-tauri/tauri.prod.conf.json`
- `apps/desktop/src-tauri/tauri.enterprise.conf.json`
- `apps/desktop/src-tauri/tauri.enterprise-dev.conf.json`
- `apps/desktop/src-tauri/tauri.local.conf.json`
- `apps/desktop/src-tauri/Cargo.toml`
- `apps/desktop/src-tauri/Info.plist`
- `apps/desktop/src-tauri/macos/Voquill.entitlements` → rename to `Verbatim.entitlements`
- `apps/windows-installer/src-tauri/tauri.conf.json`

**Rust source**:
- `apps/desktop/src-tauri/src/lib.rs` (or `main.rs`) — add `relocate_app_bundle_if_needed`
- `apps/desktop/src-tauri/src/commands.rs` — diagnostic/log filenames

**New files**:
- `apps/desktop/src-tauri/nsis/installer-hooks.nsh`

**Workflows**:
- `.github/workflows/_release-desktop-impl.yml` (artifact names, deb-control injection, rpm spec edit, Homebrew cask generation)
- `.github/workflows/build-desktop.yml` (CI artifact name)
- `.github/workflows/release.yml` (notification text)

**Distribution metadata**:
- `apps/desktop/src-tauri/archlinux/PKGBUILD`
- (External tap repo `voquill/homebrew-voquill` — modified by CI workflow generation, not committed in this repo)

**Frontend strings**:
- Anywhere `grep -rni voquill apps/desktop/src` returns user-visible matches (About dialog, support links, FormattedMessage `defaultMessage` strings).

## Research items to confirm before coding

1. **Tauri 2 NSIS uninstaller registry key** — exact path under `HKCU\Software\Microsoft\Windows\CurrentVersion\Uninstall\<???>`. Inspect a real Voquill Windows install or the bundler template. Adjust the NSIS hook accordingly.
2. **Tauri 2 deb `Replaces`/`Conflicts`/`Provides` support** — check whether `bundle.linux.deb` exposes these natively in the current Tauri version (would simplify Phase 4). Fall back to post-build dpkg-deb repack if not.
3. **Tauri 2 rpm `Obsoletes` support** — same question for rpm.
4. **Tauri macOS updater behavior** when the new `.app` bundle name differs from the running one — confirm via inspection of `tauri-plugin-updater` source that it replaces contents in place (preserves on-disk path), which is what Phase 2 assumes.
5. **macOS Gatekeeper / quarantine xattr behavior** after `fs::rename` of an `.app` to a new path — test whether the relocated app re-triggers the "downloaded from Internet" prompt. If yes, mitigation may be needed.

These don't change the plan's structure, but they affect specific implementation details. Resolve them as the first step of execution.

## Out of scope (explicitly)

- Mobile app rename — separate plan.
- `voquill.com` domain, Firebase project IDs (`voquill-prod`, `voquill-dev`), `https://*.voquill.com` capability allowlist — all stay.
- GitHub org `github.com/voquill/*` (including `voquill/voquill`, `voquill/apt`, `voquill/rpm`, `voquill/homebrew-voquill`) — all stay.
- `@voquill/*` npm package scope — stays.
- Bundle identifier (`com.voquill.desktop` and variants) — stays forever.
- SQLite DB filename `voquill.db` — stays.
- Updater endpoint URLs and pubkey — stay.
- Git tag patterns and channel release tags — stay.
- Icon files in `apps/desktop/src-tauri/icons/` — generic filenames (`icon.icns`, `icon.ico`), no rename needed unless the brand artwork itself is changing.

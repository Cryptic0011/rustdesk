# Roofing Pros USA Branded RustDesk Client Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the Synology-hosted RustDesk server reachable from the internet at `rpremote.builtbyflux.com`, and ship branded, preconfigured "Roofing Pros USA" Windows/macOS clients built by GitHub Actions on a fork.

**Architecture:** Server side is configuration only (Meraki MX port forwards → Synology `192.168.0.200`, DNS A record, container hardening). Client side is a small branding + config diff on fork `Cryptic0011/rustdesk` (branch `roofingpros`) plus a forked `Cryptic0011/hbb_common` submodule carrying the hardcoded server constants, built by a trimmed CI workflow.

**Tech Stack:** Docker (Synology Container Manager), Meraki Dashboard, GitHub Actions, Rust + Flutter (built in CI only — nothing compiles locally), ImageMagick/sips/iconutil for icons.

**Spec:** `docs/superpowers/specs/2026-06-11-rustdesk-branded-client-design.md`

**Working values (fill in as produced):**
- Public hostname: `rpremote.builtbyflux.com` → `97.68.138.154`
- Synology LAN IP: `192.168.0.200`
- Server public key `<PUBKEY>`: ____________________ (from Task 2)
- RustDesk version being built: 1.4.7 (CI `env.VERSION`)

> Tasks 1–4 are interactive (DSM, Meraki Dashboard, DNS host) — the user
> drives a browser; we verify each step from this Mac. Tasks 5–16 are
> hands-on in this repo. Task 2 must precede Task 5 (key feeds the fork).
> Task ordering otherwise: 1–4 any time before 13.

---

### Task 1: DNS A record

**Files:** none (external DNS host for `builtbyflux.com`)

- [ ] **Step 1: User adds the record** at wherever `builtbyflux.com` DNS is managed (registrar or Cloudflare):
  - Type `A`, Name `rpremote`, Value `97.68.138.154`, TTL 300 (or default). If Cloudflare: set the cloud to **DNS only (grey)** — proxied orange-cloud only handles HTTP and will break RustDesk ports.

- [ ] **Step 2: Verify resolution**

```bash
dig +short rpremote.builtbyflux.com @8.8.8.8
```

Expected: `97.68.138.154` (may take a few minutes after creation).

---

### Task 2: Extract the server's public key from the Synology container

**Files:** none (Synology DSM)

- [ ] **Step 1: Find the containers.** In DSM → Container Manager → Container, note the names of the hbbs (ID server) and hbbr (relay) containers. Typical names: `hbbs`, `hbbr` (or one combined `rustdesk-server` container running S6).

- [ ] **Step 2: Read the public key.** Easiest without SSH: DSM → Container Manager → the hbbs container → Details → Terminal → Create → `bash`, then:

```bash
cat /root/id_ed25519.pub; echo; ls /root/ /data 2>/dev/null
```

Expected: a single Base64 line like `3jcN3iX9q0LqcS0iF4iruWzGUuVZeWF2tVUOPdpRrWk=` (44 chars ending `=`). If `/root/id_ed25519.pub` doesn't exist, the `ls` shows where the key files live (commonly `/data/id_ed25519.pub` for `rustdesk/rustdesk-server` images with a mapped data volume — also visible in DSM at the volume mount, e.g. `docker/rustdesk/data/id_ed25519.pub` in File Station).

Alternative via SSH if enabled: `sudo docker exec hbbs cat /root/id_ed25519.pub`

- [ ] **Step 3: Record `<PUBKEY>`** in the header of this plan and paste it into the conversation. Sanity check: it is the `.pub` file (44-char Base64), NOT the longer secret `id_ed25519` — the secret never leaves the Synology.

- [ ] **Step 4: Check whether existing clients use the key.** On any office machine with the current stock RustDesk: Settings → Network → ID/Relay server. Record whether the **Key** field is filled (matters for Task 3).

---

### Task 3: Harden the server container (key required + encrypted only)

**Files:** none (Synology DSM)

The container must reject clients that don't present the key BEFORE the ports open to the internet (Task 4 same day).

- [ ] **Step 1: Inspect current run config.** Container Manager → hbbs container → Details → Settings: note the image (`rustdesk/rustdesk-server`), command (e.g. `hbbs -r 192.168.0.200`), env vars, volume mappings, network mode.

- [ ] **Step 2: Recreate hbbs hardened.** Stop the container → Edit (or duplicate settings into a new one if Edit is locked):
  - Command: `hbbs -r rpremote.builtbyflux.com:21117 -k _`
  - Add env var: `ENCRYPTED_ONLY=1`
  - Keep the SAME volume mapping (so the existing key pair is reused — this is what keeps existing clients trusting the server).
- [ ] **Step 3: Recreate hbbr hardened.** Same procedure: command `hbbr -k _`, env `ENCRYPTED_ONLY=1`, same volumes.

- [ ] **Step 4: Verify key unchanged.** Re-run Task 2 Step 2; the printed key must be IDENTICAL to `<PUBKEY>`. If it changed, the volume mapping was lost — stop, restore the old mapping (or copy the old `id_ed25519`/`id_ed25519.pub` back into the data dir via File Station) and restart.

- [ ] **Step 5: Verify an office client still connects.**
  - If Task 2 Step 4 found the Key field FILLED: client should show "Ready" within ~30 s, no action.
  - If EMPTY: it will now show "Key mismatch"/offline. Paste `<PUBKEY>` into Settings → Network → Key on that machine; confirm "Ready". Plan ~30 s per remaining office machine, or skip — they get fixed automatically when the branded client is installed (Task 15/16).
  - Also confirm a quick connection between two office machines still works.

---

### Task 4: Meraki MX port forwarding

**Files:** none (dashboard.meraki.com)

- [ ] **Step 1: Add four rules.** Dashboard → Security & SD-WAN → Configure → Firewall → Port forwarding rules → Add, then Save. Every rule: Uplink `Internet 1`, LAN IP `192.168.0.200`, Allowed remote IPs `Any`:

| Description | Protocol | Public port | Local port |
|---|---|---|---|
| RustDesk NAT test | TCP | 21115 | 21115 |
| RustDesk ID/rendezvous | TCP | 21116 | 21116 |
| RustDesk ID/rendezvous | UDP | 21116 | 21116 |
| RustDesk relay | TCP | 21117 | 21117 |

- [ ] **Step 2: Verify TCP from outside.** From this Mac on a phone hotspot (or any off-network machine), or right away via https://canyouseeme.org against ports 21115/21116/21117:

```bash
nc -vz -G 5 rpremote.builtbyflux.com 21115 && nc -vz -G 5 rpremote.builtbyflux.com 21116 && nc -vz -G 5 rpremote.builtbyflux.com 21117
```

Expected: three `succeeded!` lines. UDP 21116 can't be probed this way — it's verified by the end-to-end client test (Task 14). If a port fails: check the rule's uplink is Internet 1, then confirm the Synology firewall (DSM → Security → Firewall) allows 21115-21117 from all.

---

### Task 5: Fork hbb_common and hardcode the server constants

**Files:**
- Modify: `libs/hbb_common/src/config.rs:120-121` (in the submodule, pushed to the fork)

- [ ] **Step 1: Create the fork**

```bash
gh repo fork rustdesk/hbb_common --clone=false
```

Expected: `✓ Created fork Cryptic0011/hbb_common` (or "already exists").

- [ ] **Step 2: Branch inside the submodule**

```bash
cd /Users/graysonpatterson/Grayson/rustdesk/libs/hbb_common
git remote add fork https://github.com/Cryptic0011/hbb_common.git 2>/dev/null; git fetch fork
git checkout -b roofingpros 387603f47cbb15c0d3dc3d67ae3396d3eb707daf
```

- [ ] **Step 3: Edit the constants.** In `libs/hbb_common/src/config.rs`, replace lines 120-121:

```rust
pub const RENDEZVOUS_SERVERS: &[&str] = &["rs-ny.rustdesk.com"];
pub const RS_PUB_KEY: &str = "OeVuKk5nlHiXp+APNn0Y3pC1Iwpwn44JGqrQCsWqmBw=";
```

with (substituting the real `<PUBKEY>` from Task 2):

```rust
pub const RENDEZVOUS_SERVERS: &[&str] = &["rpremote.builtbyflux.com"];
pub const RS_PUB_KEY: &str = "<PUBKEY>";
```

- [ ] **Step 4: Sanity-check the edit**

```bash
grep -n "rpremote\|RS_PUB_KEY" src/config.rs | head -3
```

Expected: line 120 shows `rpremote.builtbyflux.com`, line 121 shows the new key, no other unintended hits.

- [ ] **Step 5: Commit and push to the fork**

```bash
git add src/config.rs
git commit -m "Hardcode Roofing Pros USA rendezvous server and public key"
git push fork roofingpros
```

---

### Task 6: Fork rustdesk and repoint the submodule

**Files:**
- Modify: `.gitmodules`
- Modify: `libs/hbb_common` (gitlink)

- [ ] **Step 1: Fork and add remote**

```bash
cd /Users/graysonpatterson/Grayson/rustdesk
gh repo fork rustdesk/rustdesk --clone=false
git remote add fork https://github.com/Cryptic0011/rustdesk.git 2>/dev/null
```

- [ ] **Step 2: Repoint `.gitmodules`.** Change the hbb_common URL:

```
[submodule "libs/hbb_common"]
	path = libs/hbb_common
	url = https://github.com/Cryptic0011/hbb_common.git
```

then sync:

```bash
git submodule sync libs/hbb_common
```

- [ ] **Step 3: Commit the new submodule pointer**

```bash
git add .gitmodules libs/hbb_common
git commit -m "Point hbb_common submodule at Roofing Pros USA fork (custom server constants)"
```

- [ ] **Step 4: Verify a fresh-clone perspective**

```bash
git submodule status libs/hbb_common
```

Expected: the SHA equals the commit pushed in Task 5 Step 5 (check with `cd libs/hbb_common && git rev-parse HEAD`), with no `+` or `-` prefix.

---

### Task 7: Brand the display name (4 files)

**Files:**
- Modify: `flutter/lib/desktop/widgets/tabbar_widget.dart:644`
- Modify: `flutter/lib/common.dart:2997` (`getWindowName`)
- Modify: `flutter/windows/runner/Runner.rc:93` area (ProductName, FileDescription)
- Modify: `flutter/macos/Runner/Info.plist` (add CFBundleDisplayName)

No test suite covers these UI strings; verification is `grep` now + visual check in Task 14/15.

- [ ] **Step 1: Main-window brand text.** `flutter/lib/desktop/widgets/tabbar_widget.dart` line 644: change

```dart
                              "RustDesk",
```

to

```dart
                              "Roofing Pros USA",
```

- [ ] **Step 2: Window titles.** In `flutter/lib/common.dart`, function `getWindowName` (~line 2996), change

```dart
  final name = bind.mainGetAppNameSync();
```

to

```dart
  final name = "Roofing Pros USA";
```

(Display-only: `bind.mainGetAppNameSync()` elsewhere still drives config paths, which we deliberately do not touch.)

- [ ] **Step 3: Windows file properties.** In `flutter/windows/runner/Runner.rc` (~line 93), change

```
            VALUE "FileDescription", "RustDesk Remote Desktop" "\0"
```
to
```
            VALUE "FileDescription", "Roofing Pros USA Remote Desktop" "\0"
```
and
```
            VALUE "ProductName", "RustDesk" "\0"
```
to
```
            VALUE "ProductName", "Roofing Pros USA" "\0"
```

- [ ] **Step 4: macOS Dock/Finder name.** In `flutter/macos/Runner/Info.plist`, directly after the `CFBundleName` pair (lines 15-16):

```xml
	<key>CFBundleName</key>
	<string>$(PRODUCT_NAME)</string>
```

insert:

```xml
	<key>CFBundleDisplayName</key>
	<string>Roofing Pros USA</string>
```

- [ ] **Step 5: Verify and commit**

```bash
grep -rn "Roofing Pros USA" flutter/lib flutter/windows/runner/Runner.rc flutter/macos/Runner/Info.plist
git add -A && git commit -m "Brand user-visible name as Roofing Pros USA"
```

Expected: 5 hits (tabbar, common.dart, Runner.rc ×2, Info.plist).

---

### Task 8: Generate and replace icons

**Files:**
- Modify (replace content): `res/icon.png`, `res/mac-icon.png`, `res/icon.ico`, `res/tray-icon.ico`, `flutter/windows/runner/resources/app_icon.ico`, `flutter/macos/Runner/AppIcon.icns`
- Create: `flutter/assets/logo.png`
- Source: `/Users/graysonpatterson/Grayson/RoofingProsAutomation/Sources/RoofingProsAutomation/Resources/Assets.xcassets/AppIcon.appiconset/icon_512x512.png`

Deliberately NOT replaced: `res/mac-tray-light-x2.png` / `res/mac-tray-dark-x2.png` (macOS renders them as monochrome templates — full-color logo would become a solid blob; stock glyph stays) and `flutter/assets/icon.svg` (no vector source).

- [ ] **Step 1: Ensure ImageMagick**

```bash
command -v magick || brew install imagemagick
```

- [ ] **Step 2: Stage the source**

```bash
cd /Users/graysonpatterson/Grayson/rustdesk
cp "/Users/graysonpatterson/Grayson/RoofingProsAutomation/Sources/RoofingProsAutomation/Resources/Assets.xcassets/AppIcon.appiconset/icon_512x512.png" /tmp/rp_logo_512.png
```

- [ ] **Step 3: PNG sources + in-app logo**

```bash
cp /tmp/rp_logo_512.png res/icon.png
cp /tmp/rp_logo_512.png res/mac-icon.png
sips -Z 60 /tmp/rp_logo_512.png --out flutter/assets/logo.png
```

(`flutter/pubspec.yaml` already bundles the whole `assets/` dir at line 153-154, and `flutter/lib/common.dart:3719` auto-loads `assets/logo.png` — no code change needed.)

- [ ] **Step 4: Windows icons**

```bash
magick /tmp/rp_logo_512.png -define icon:auto-resize=256,128,64,48,32,16 res/icon.ico
cp res/icon.ico flutter/windows/runner/resources/app_icon.ico
magick /tmp/rp_logo_512.png -define icon:auto-resize=32,16 res/tray-icon.ico
```

- [ ] **Step 5: macOS .icns**

```bash
mkdir -p /tmp/AppIcon.iconset
for s in 16 32 64 128 256 512; do
  sips -z $s $s /tmp/rp_logo_512.png --out /tmp/AppIcon.iconset/icon_${s}x${s}.png >/dev/null
done
cp /tmp/AppIcon.iconset/icon_32x32.png /tmp/AppIcon.iconset/icon_16x16@2x.png
cp /tmp/AppIcon.iconset/icon_64x64.png /tmp/AppIcon.iconset/icon_32x32@2x.png
cp /tmp/AppIcon.iconset/icon_256x256.png /tmp/AppIcon.iconset/icon_128x128@2x.png
cp /tmp/AppIcon.iconset/icon_512x512.png /tmp/AppIcon.iconset/icon_256x256@2x.png
rm /tmp/AppIcon.iconset/icon_64x64.png
iconutil -c icns /tmp/AppIcon.iconset -o flutter/macos/Runner/AppIcon.icns
```

- [ ] **Step 6: Verify all six artifacts**

```bash
file res/icon.png res/mac-icon.png res/icon.ico res/tray-icon.ico flutter/windows/runner/resources/app_icon.ico flutter/macos/Runner/AppIcon.icns flutter/assets/logo.png
```

Expected: PNGs report `512 x 512` (logo.png `60 x 60`), `.ico` files report `MS Windows icon resource - 6 icons` (tray: 2), `.icns` reports `Mac OS X icon`.

- [ ] **Step 7: Commit**

```bash
git add res/ flutter/windows/runner/resources/ flutter/macos/Runner/AppIcon.icns flutter/assets/logo.png
git commit -m "Replace icons with Roofing Pros USA logo"
```

---

### Task 9: Trimmed CI workflow (custom-client.yml)

**Files:**
- Create: `.github/workflows/custom-client.yml` (assembled from `.github/workflows/flutter-build.yml` — windows job lines 65-273, macOS job lines 556-764, env lines 20-48, helper jobs lines 51-63)

- [ ] **Step 1: Assemble the file**

```bash
cd /Users/graysonpatterson/Grayson/rustdesk
{
  printf 'name: Custom client (Roofing Pros USA)\n\non:\n  workflow_dispatch:\n\n'
  sed -n '20,48p' .github/workflows/flutter-build.yml
  printf '\njobs:\n'
  sed -n '51,63p' .github/workflows/flutter-build.yml
  printf '\n'
  sed -n '65,273p' .github/workflows/flutter-build.yml
  printf '\n'
  sed -n '556,764p' .github/workflows/flutter-build.yml
} > .github/workflows/custom-client.yml
```

This copy drops the `Publish Release` steps (windows 274-282, macOS 765-772) and all other platform jobs; artifacts are collected from the run page instead.

- [ ] **Step 2: Replace the `inputs.*` references** (the copied env block came from a `workflow_call` workflow; ours is `workflow_dispatch`):

```bash
sed -i '' \
  -e 's|"\${{ inputs.upload-tag }}"|"custom"|' \
  -e 's|"\${{ inputs.upload-artifact }}"|"true"|' \
  -e 's|\${{ inputs.upload-artifact }}|true|' \
  .github/workflows/custom-client.yml
grep -n "inputs\." .github/workflows/custom-client.yml
```

Expected: grep returns NOTHING. If any `inputs.` remains (upstream may add references between our pinned lines), replace each the same way: `upload-artifact` → `true`, `upload-tag` → `"custom"`.

- [ ] **Step 3: Validate YAML parses**

```bash
python3 -c "import yaml,sys; yaml.safe_load(open('.github/workflows/custom-client.yml')); print('YAML OK')"
```

Expected: `YAML OK`. (If PyYAML missing: `pip3 install pyyaml`.) Also sanity-check job list:

```bash
python3 -c "import yaml; print(list(yaml.safe_load(open('.github/workflows/custom-client.yml'))['jobs']))"
```

Expected: `['generate-bridge', 'build-RustDeskTempTopMostWindow', 'build-for-windows-flutter', 'build-for-macOS']`

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/custom-client.yml
git commit -m "Add trimmed workflow_dispatch build for Windows+macOS custom client"
```

---

### Task 10: Push branch and enable Actions on the fork

- [ ] **Step 1: Push**

```bash
git push fork roofingpros
```

- [ ] **Step 2: Enable workflows on the fork.** Fresh forks have Actions disabled. Open https://github.com/Cryptic0011/rustdesk/actions and click "I understand my workflows, go ahead and enable them". Verify via:

```bash
gh workflow list --repo Cryptic0011/rustdesk | head
```

Expected: a list including `Custom client (Roofing Pros USA)` (workflows must be enabled for this to return rows).

---

### Task 11: Trigger the build and babysit it

- [ ] **Step 1: Dispatch**

```bash
gh workflow run custom-client.yml --repo Cryptic0011/rustdesk --ref roofingpros
sleep 10
gh run list --repo Cryptic0011/rustdesk --workflow=custom-client.yml --limit 1
```

Expected: a run in `in_progress`. Note its `<RUN_ID>`.

- [ ] **Step 2: Watch (~60-75 min; vcpkg is the long pole on first run)**

```bash
gh run watch <RUN_ID> --repo Cryptic0011/rustdesk --exit-status
```

Expected: exit 0, jobs `build-for-windows-flutter` and `build-for-macOS` (×2 arch) green. If a job fails, pull logs with `gh run view <RUN_ID> --repo Cryptic0011/rustdesk --log-failed` and debug — the most likely failure class is our own copied-line drift in custom-client.yml (compare against flutter-build.yml), not the build itself.

---

### Task 12: Download and package artifacts

- [ ] **Step 1: Download**

```bash
mkdir -p ~/Desktop/RoofingProsRemote && cd ~/Desktop/RoofingProsRemote
gh run download <RUN_ID> --repo Cryptic0011/rustdesk
ls -R .
```

Expected dirs: `rustdesk-unsigned-windows-x86_64/` (loose exe/dlls — ignore; the installer exe is inside it as `rustdesk-1.4.7-x86_64.exe` if present) and `rustdesk-unsigned-macos-x86_64/`, `rustdesk-unsigned-macos-aarch64/` (each one `.dmg`). The Windows self-extracting installer `rustdesk-1.4.7-x86_64.exe` is uploaded by the portable step — locate with `find . -name '*.exe'`.

- [ ] **Step 2: Rename for distribution**

```bash
cd ~/Desktop/RoofingProsRemote
find . -name 'rustdesk-1.4.7-x86_64.exe' -exec cp {} RoofingProsUSA-1.4.7-windows.exe \;
cp rustdesk-unsigned-macos-aarch64/*.dmg RoofingProsUSA-1.4.7-mac-applesilicon.dmg
cp rustdesk-unsigned-macos-x86_64/*.dmg RoofingProsUSA-1.4.7-mac-intel.dmg
ls -la RoofingProsUSA-*
```

Expected: three renamed files, exe ~20-40 MB, dmgs ~40-70 MB.

---

### Task 13: macOS verification (this Mac)

- [ ] **Step 1: Install.** Open `RoofingProsUSA-1.4.7-mac-applesilicon.dmg` (use the intel one if this Mac is Intel — check `uname -m`), drag to Applications. First launch: right-click `RustDesk.app` → Open → Open (unsigned-app bypass).

- [ ] **Step 2: Verify branding.** Dock label and window title read "Roofing Pros USA"; Dock icon is the teal RP logo; main window shows the logo and brand text.

- [ ] **Step 3: Verify connectivity.** Status bar at bottom of main window shows "Ready" (green). Settings → Network should show the hardcoded server (fields locked or pre-filled with `rpremote.builtbyflux.com`). This validates DNS + TCP/UDP 21116 + key end-to-end if this Mac is OFF the office network (hotspot); on-network it validates hairpin NAT instead — do whichever is available now, the other in Task 14.

- [ ] **Step 4: Control a real machine.** Enter an office machine's RustDesk ID + its permanent password (or have someone accept) → screen appears, input works. This validates relay TCP 21117 too (forced relay if needed: connection menu → "Always connect via relay").

---

### Task 14: Windows + remaining network verification

- [ ] **Step 1: Windows install.** On one office Windows PC: run `RoofingProsUSA-1.4.7-windows.exe` → SmartScreen "More info → Run anyway" → in-app Install. Verify window title/brand text/taskbar+tray icon, status "Ready", and that its ID matches the machine's previous RustDesk ID (config persisted).

- [ ] **Step 2: Cross-network test.** From home or hotspot Mac (Task 13 build), connect to that Windows PC. Both directions if practical.

- [ ] **Step 3: Hairpin test (if Task 13 ran off-network).** On an office machine with the branded client (public hostname baked in): status must be "Ready" and a connection to another office machine must work from inside the LAN.

- [ ] **Step 4: Stock-client regression check.** One untouched office machine still pointing at `192.168.0.200` (with key, per Task 3): still "Ready", still connectable. Confirms transition safety.

---

### Task 15: Distribute

- [ ] **Step 1: Upload** the three `RoofingProsUSA-*` files to a SharePoint/OneDrive folder shared with the team.

- [ ] **Step 2: Post install note** alongside them:

> **Roofing Pros USA Remote — install**
> Windows: download `RoofingProsUSA-1.4.7-windows.exe`, run it (if SmartScreen appears: "More info" → "Run anyway"), click Install. Done — no settings needed.
> Mac: download the Apple Silicon or Intel dmg, drag to Applications, FIRST launch only: right-click the app → Open → Open.
> Your computer's ID and password stay the same as before. The old RustDesk app can be uninstalled after this is installed.

- [ ] **Step 3 (cleanup): commit any remaining repo changes** and confirm `git status` clean on `roofingpros`; push.

---

### Task 16: Documentation for future maintenance

**Files:**
- Create: `docs/superpowers/specs/2026-06-11-rustdesk-ops-notes.md`

- [ ] **Step 1: Write ops notes** capturing: fork URLs, branch name, how to rebuild (sync fork → rebase `roofingpros` → re-run custom-client.yml), where the key lives (Synology container volume — back up `id_ed25519*` files via File Station copy), the four Meraki rules, the DNS record, and the accepted branding limitations list from the spec.

- [ ] **Step 2: Commit and push**

```bash
git add docs/superpowers/specs/2026-06-11-rustdesk-ops-notes.md
git commit -m "Add ops notes for branded client maintenance"
git push fork roofingpros
```

---

## Self-review notes

- Spec coverage: server exposure (T1-T4), key reuse + hardening (T2-T3), hairpin (T13/T14), branding name (T7) + icons (T8), hardcoded server (T5-T6), CI build (T9-T11), distribution + install note (T12, T15), maintenance (T16), stock-client regression (T14.4). Error-handling rows from spec map to T4.2 fallback checks and T11.2 log triage.
- Known judgment calls encoded: MSI left stock (we distribute the .exe), mac tray + icon.svg left stock, `PRODUCT_NAME`/`APP_NAME` untouched — all recorded as accepted limitations in the spec.
- Line numbers (Runner.rc:93, tabbar:644, common.dart:2997, config.rs:120) verified against working tree at commit `7e983eee1`.

# Roofing Pros USA — Remote Access via RustDesk (Design)

**Date:** 2026-06-11
**Status:** Approved pending user spec review

## Problem

The team's RustDesk server runs in Docker on a Synology NAS and is only
reachable at its LAN address (`192.168.0.200`). Remote work days mean no
access. Tailscale-per-machine and RustDesk Pro were rejected (cost/hassle).

## Goals

1. Reach team machines from outside the office network (e.g. from home).
2. A branded client ("Roofing Pros USA", custom logo) that is preconfigured
   with the server address and key — zero setup for end users.
3. Keep the existing Synology server; no migration, no new hosting cost.
4. Existing stock-RustDesk installs keep working during the transition.

## Non-goals (out of scope)

- Full white-label (executable/service/registry renames stay `rustdesk`).
- Web client (ports 21118/21119 stay closed).
- Mobile (iOS/Android) builds.
- Code signing certificates (users accept the one-time unsigned-app prompt).

## Decisions already made

| Topic | Decision |
|---|---|
| Server hosting | Keep Synology Docker container (bot-hosting.net is a Discord-bot host; cannot run RustDesk) |
| Reachability | Meraki MX port-forward rules + DNS A record on static office IP |
| Office WAN | Static public IP `97.68.138.154/29` (Spectrum) on MX Internet 1 — no CGNAT, no DDNS needed |
| Branding depth | Display name + logo only |
| Client platforms | Windows + macOS |
| Build method | Fork `rustdesk/rustdesk`, GitHub Actions CI (GitHub account: `Cryptic0011`) |
| App display name | "Roofing Pros USA" |
| Logo source | `~/Grayson/RoofingProsAutomation/Sources/RoofingProsAutomation/Resources/Assets.xcassets/AppIcon.appiconset/icon_512x512.png` (512×512 PNG, verified) |

## Architecture

```
Home laptop (branded client)
        │  rpremote.builtbyflux.com → 97.68.138.154 (static)
        ▼
Meraki MX ── port forwards TCP 21115-21117, UDP 21116 ──► Synology 192.168.0.200
                                                           └─ Docker: hbbs (ID/rendezvous) + hbbr (relay)
Office desktops (stock clients, still pointed at 192.168.0.200) ──► same hbbs/hbbr
```

Direct (peer-to-peer) connections are attempted first; `hbbr` relays when
NAT prevents them.

## Component 1: Server exposure (Synology + Meraki MX)

- **Port forwards** in the Meraki Dashboard (dashboard.meraki.com →
  Security & SD-WAN → Configure → Firewall → Port forwarding rules), each
  with Uplink = Internet 1, LAN IP = `192.168.0.200`, Allowed remote IPs =
  `any`: TCP 21115, TCP 21116, UDP 21116, TCP 21117. Nothing else.
  Requires Dashboard write access (the local `*.devices.meraki.direct`
  status page cannot configure forwarding).
- **Hostname:** DNS A record `rpremote.builtbyflux.com → 97.68.138.154`
  (office IP is static, so no DDNS). The hostname (not the raw IP) is what
  gets baked into clients, so a future ISP/IP change is a DNS edit, not a
  client rebuild.
- **Hairpin NAT:** Meraki MX enables hairpin routing for port-forward rules
  by default, so in-office machines can use the public hostname too —
  verified in Meraki documentation; confirmed during testing anyway.
- **Key reuse:** copy the existing public key from the container data dir
  (`id_ed25519.pub`) so current installs keep trusting the server.
- **Hardening** (container env/args): require the key (`-k _`) and set
  `ENCRYPTED_ONLY=1` so anonymous internet clients cannot register or relay.
  DSM and other Synology services are NOT exposed — only the four RustDesk
  ports.
- **Key-enforcement risk:** if existing office installs were configured with
  the server address but an empty Key field, enabling enforcement will
  disconnect them until the key is pasted into their settings (one-time,
  ~30 seconds per machine). Implementation checks one existing client first;
  enforcement is enabled the same day the ports are opened — exposure
  without key enforcement is not acceptable.

## Component 2: Branded client (this repo)

- Fork `rustdesk/rustdesk` → `Cryptic0011/rustdesk`; work on branch
  `roofingpros`.
- **Branding diff** (kept deliberately small for future upstream rebases):
  - App display name → "Roofing Pros USA" (Flutter UI strings/config,
    `app_name`).
  - Icons: regenerate all required sizes (Windows `.ico`, macOS `.icns`,
    Flutter assets, tray icons) from the 512×512 logo into `res/` and
    `flutter/` asset locations.
- **Server config baked at build time** via the officially supported env vars
  `RENDEZVOUS_SERVER` (DDNS hostname) and `RS_PUB_KEY` (server public key),
  stored as GitHub Actions **secrets** — never committed. Hardcoding hides
  the custom-server settings UI, so users cannot misconfigure it.
- **Build:** trigger the fork's existing `flutter-build` GitHub Actions
  workflow; collect Windows `.exe` installer and macOS `.dmg` artifacts.
- License note: RustDesk is AGPL-3.0; the fork stays public on GitHub, which
  satisfies source availability.

## Component 3: Distribution & rollout

1. Test installers on one Windows machine and one Mac from a phone hotspot
   (true outside-network test).
2. **Hairpin-NAT check:** from inside the office, connect via the public
   hostname. Meraki MX hairpins port-forwarded traffic by default, so this
   is expected to pass; if it somehow fails, the contingency is a local DNS
   override mapping the hostname → `192.168.0.200`.
3. Share installers via SharePoint/OneDrive (M365 tenant already in use).
4. Existing stock installs keep working (same server, same key); machines
   migrate to the branded build opportunistically. Machine IDs persist across
   the reinstall because ID/config live on the machine, not in the client
   binary.

## Error handling / failure modes

- **Server unreachable from outside:** verify in order — hostname resolves
  to `97.68.138.154` → MX forwarding rules hit the Synology → container
  ports listening. Each step has a one-line test in the implementation plan.
  (CGNAT ruled out: the MX uplink has a static public /29.)
- **Unsigned-app prompts:** macOS right-click → Open (first launch only);
  Windows SmartScreen "More info → Run anyway". Include a one-paragraph
  user-facing install note alongside the installers.

## Testing

- Server: `nc -vz` (TCP) / `nc -vzu` (UDP) checks against the public
  hostname from off-network; hbbs/hbbr log inspection on first client
  registration.
- Client: install on Windows + macOS; confirm preconfigured server shows
  connected (green status), branding renders (name, window icon, tray icon);
  connect Mac↔Windows from hotspot; verify office-internal connection via
  public hostname (hairpin test); confirm an untouched stock install still
  connects via `192.168.0.200`.

## Maintenance

- Upstream updates are opt-in: sync fork, rebase the `roofingpros` branch
  (small diff by design), re-run CI. No auto-update channel is configured.
- If the server key or hostname ever changes, only GitHub secrets change;
  re-run CI and redistribute.

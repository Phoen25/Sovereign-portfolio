# M-01 Operational Runbook
## SIOE-X Sovereign Infrastructure Lab (SIL) — Phase 1, Linux Foundations & Operational Systems

**Document standard:** Vol 5 Lab 1, Level 2 (full runbook, troubleshooting included, all configuration values documented with rationale, reproducible by another engineer). Assessed against Vol 6, M-01 rubric, Documentation Quality category.

**The test this document is written to pass:** a qualified engineer who has never seen this server should be able to take full operational ownership of it within 48 hours, using only this document.

**Honesty convention used throughout:** every configuration detail below is marked either *(verified)* — directly confirmed via command output during this build, with a date — or *(inherited, unverified)* — present from prior hardening work, never independently confirmed in this session. Do not treat an unverified detail as fact. Confirm it before depending on it. This convention exists because this server's own history (see Change Log, INC-001) is a direct lesson in why.

---

## 1. System Overview

**What this server is.** The Phase 1 instance of the SIOE-X Sovereign Infrastructure Lab (SIL) — a single, continuously evolving infrastructure environment, not a disposable practice environment. Every phase of the SIOE-X curriculum from M-01 onward builds directly onto this server; nothing here is meant to be discarded and rebuilt.

**What it runs.** Hardened SSH access, a UFW-managed firewall, fail2ban intrusion prevention, a WireGuard VPN, an XRDP remote desktop restricted to the VPN subnet, and a Prometheus + Node Exporter + Grafana observability stack.

**Who maintains it.** Wavamuno Shafiki (FIQ PRO), Founder & Chief Architect, SIOE-X. Sole operator as of this document's authorship.

**How to connect.**
```
ssh empire
```
This is a Termux SSH config alias (`~/.ssh/config`, `Host empire`) targeting `ghostadmin@<public-ip>` with key-based authentication only, using an ed25519 keypair held in Termux's proot environment at `/data/data/com.termux/files/home/.ssh/id_ed25519`.

**Provider and identity.**

| Field | Value |
|---|---|
| Provider / Region | AWS EC2, eu-west-2 (London) |
| Public IP *(verified 2026-06-28, via WireGuard endpoint)* | <redacted> |
| Internal IP (ens5) | 10.0.1.201 |
| Hostname | ip-10-0-1-201 |
| OS | Ubuntu 24.04.4 LTS |
| Kernel *(verified 2026-06-28, post-reboot)* | 6.17.0-1019-aws |
| Primary interface | ens5 — **not** eth0; this is a Nitro-based instance and SP-1's original session plans assumed the older naming scheme |
| Operational user | ghostadmin, uid=1001, groups: ghostadmin, sudo, users, sioe-ops |

⚠ The public IP above will change if the instance is ever stopped and restarted without an Elastic IP attached. Confirm it has not drifted before relying on the value in this document.

---

## 2. Architecture Diagram

```
                              INTERNET
                                 │
                    ┌────────────┴─────────────┐
                    │  UFW (default-deny)       │
                    │  backend: iptables-nft     │
                    │  see ADR-001 — never edit  │
                    │  nftables tables directly  │
                    └────────────┬─────────────┘
        ┌───────────────┬────────┴────────┬───────────────┐
        │                │                 │               │
   tcp/22 (any)    udp/51820 (any)   tcp/3389         (all else: DROP)
     SSH              WireGuard      10.0.2.0/24 ONLY
        │                │                 │
        ▼                ▼                 ▼
   ┌─────────┐    ┌─────────────┐   ┌──────────┐
   │  sshd   │    │    wg0      │   │  XRDP /  │
   │key-only │    │ 10.0.2.1/24 │   │ lightdm  │
   └────┬────┘    │ port 51820  │   └──────────┘
        │         └──────┬──────┘
        │                │
        │         ┌──────┴───────┐
        │         │ peer: phone  │
        │         │ 10.0.2.2/32  │
        │         └──────────────┘
        ▼
  ┌────────────────────────────────────────────┐
  │   ghostadmin  (uid 1001, sudo, sioe-ops)     │
  └──────────────────┬───────────────────────────┘
                      │
        ┌─────────────┼───────────────────┐
        ▼             ▼                    ▼
  ┌───────────┐ ┌──────────────┐   ┌──────────────┐
  │ / (root)  │ │   /data      │   │  fail2ban    │
  │ 8GB nvme0 │ │  20GB nvme1  │   │  jail: sshd  │
  │ ⚠ elevated│ │  ext4, UUID  │   └──────────────┘
  │ usage —   │ │  15316f43... │
  │ see §6    │ └──────┬───────┘
  └───────────┘        │
        ┌───────────────┼────────────────────┐
        ▼               ▼                    ▼
  ┌────────────┐ ┌─────────────┐   ┌───────────────┐
  │ /data/     │ │ /data/      │   │ /data/scripts │
  │ prometheus │ │ grafana     │   │ setup.sh      │
  └─────┬──────┘ └──────┬──────┘   │ practice.sh   │
        │               │          │ greet.sh      │
        ▼               ▼          └───────────────┘
  ┌────────────┐ ┌─────────────┐
  │ Prometheus │◄┤  Grafana    │  ◄── SSH tunnel ONLY
  │   :9090    │ │   :3000     │      (ssh -L 3000:localhost:3000 empire)
  └─────┬──────┘ └─────────────┘      never exposed via UFW
        │
        ▼ scrapes every 15s
  ┌───────────────┐
  │ Node Exporter │
  │    :9100      │
  └───────────────┘
```

---

## 3. Service Inventory

| Service | Purpose | Port | Data Directory | systemd Unit | Runs As | Logs |
|---|---|---|---|---|---|---|
| sshd | Remote shell | 22/tcp | n/a | ssh.service | root | `/var/log/auth.log` |
| ufw | Firewall | n/a | n/a | ufw.service | root | `journalctl -u ufw` |
| fail2ban | Brute-force protection | n/a | n/a | fail2ban.service | root | `/var/log/fail2ban.log` |
| WireGuard | VPN tunnel | 51820/udp | n/a | wg-quick@wg0.service | root | `journalctl -u wg-quick@wg0` |
| XRDP / lightdm | Remote desktop (VPN-only) | 3389/tcp | n/a | xrdp.service, lightdm.service | root | `journalctl -u xrdp` |
| Prometheus | Metrics collection | 9090/tcp | `/data/prometheus` | prometheus.service | prometheus (system, no shell) | `journalctl -u prometheus` |
| Node Exporter | System metrics | 9100/tcp | n/a | node_exporter.service | node_exporter (system, no shell) | `journalctl -u node_exporter` |
| Grafana | Dashboards | 3000/tcp (tunnel only) | `/data/grafana` | grafana-server.service | grafana (system) | `journalctl -u grafana-server` |

---

## 4. Configuration Reference

### SSH — `/etc/ssh/sshd_config`
- `PermitRootLogin no` *(verified 2026-06-21, re-verified 2026-06-30)* — eliminates root as an SSH target entirely.
- `PasswordAuthentication no` *(verified both dates)* — key-only auth, removes the brute-force password vector.
- `PubkeyAuthentication yes` *(verified both dates)*.

### UFW — managed via the `ufw` command
**Never hand-edit the underlying nftables tables.** This server's firewall is UFW, backed by iptables-nft — confirmed by UFW's own warning (`table ip filter is managed by iptables-nft, do not touch!`). Full rationale in `sioe-x-infrastructure/docs/adrs/ADR-001-ufw-over-nftables.md`.

| Rule | Scope | Rationale |
|---|---|---|
| 22/tcp ALLOW | Anywhere | SSH access |
| 51820/udp ALLOW | Anywhere | WireGuard |
| 3389/tcp ALLOW | **10.0.2.0/24 only** | XRDP — deliberately scoped to the VPN subnet, never exposed publicly |

Default policy: deny incoming, allow outgoing, both IPv4 and IPv6.

### fail2ban — `/etc/fail2ban/jail.local` *(inherited, unverified)*
File contents were never directly read in this session — only `fail2ban-client status` was used, which is a live status query, not a config file confirmation. Treat the jail's exact `maxretry`/`findtime`/`bantime` values as unknown until you run `cat /etc/fail2ban/jail.local` yourself.

What **is** verified: jail `sshd` is active and integrated into the firewall via an `f2b-sshd` chain. Baseline observed 2026-06-23: 1,639 total failed attempts, 72 total bans, 0 currently banned.

### WireGuard — `/etc/wireguard/wg0.conf` *(verified 2026-06-28)*
```
[Interface]
Address = 10.0.2.1/24
ListenPort = 51820
PostUp   = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens5 -j MASQUERADE

[Peer]
PublicKey  = <phone's actual public key — see Common Operations §5.7 to confirm current value>
AllowedIPs = 10.0.2.2/32
```
- Address `10.0.2.1/24` was deliberately chosen to share the exact subnet UFW already trusts for XRDP — not arbitrary.
- ⚠ **This service was found disabled at boot on 2026-06-28** — it ran correctly every time it was started manually, but had no boot-persistence at all. See Change Log and INC-002. Confirm `systemctl is-enabled wg-quick@wg0` returns `enabled` before trusting this as a reliable second access path.

### Prometheus — `/etc/prometheus/prometheus.yml` *(verified 2026-06-23)*
```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'prometheus'
    static_configs: [{ targets: ['localhost:9090'] }]
  - job_name: 'node'
    static_configs: [{ targets: ['localhost:9100'] }]
```
`--storage.tsdb.path=/data/prometheus/` — deliberately not the default `/var/lib/prometheus/`. This was a direct response to the root-disk constraint found during the M-01 Session 4 audit, applied *before* installation rather than after.

### Grafana — `/etc/grafana/grafana.ini` *(verified 2026-06-25)*
- `data = /data/grafana` — same rationale as Prometheus; redirected before the first start, so the database was never initialized on the root disk at all.
- Installed via the modern apt signed-by method (`/etc/apt/keyrings/grafana.gpg`), **not** the deprecated `apt-key` method — required on Ubuntu 24.04.
- Access is SSH-tunnel only: `ssh -L 3000:localhost:3000 empire`, then browse to `http://localhost:3000`. Never add a UFW rule for port 3000.

### Storage — `/etc/fstab` *(verified 2026-06-21)*
```
UUID=<uuid-redacted>  /data  ext4  defaults,nofail  0  2
```
`nofail` is deliberate — if this volume is ever unavailable at boot, the server still boots rather than hanging. Ownership: `ghostadmin:sioe-ops`, mode `775`.

---

## 5. Common Operations

**5.1 — Restart a service**
```bash
sudo systemctl restart <service-name>
sudo systemctl status <service-name>
```

**5.2 — Check service logs**
```bash
journalctl -u <service-name> -n 50          # last 50 lines
journalctl -u <service-name> -f             # live follow
```

**5.3 — Verify the firewall**
```bash
sudo ufw status verbose
```
Confirm the three expected rules (22/tcp, 51820/udp, 3389/tcp scoped to 10.0.2.0/24) are present and nothing else has been added.

**5.4 — Add a new SSH key**
```bash
echo "<new-public-key>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```
Test the new key from a second, separate terminal **before** removing any existing key.

**5.5 — Connect to Grafana**
```bash
ssh -L 3000:localhost:3000 empire
```
Leave that session open and idle; browse to `http://localhost:3000` in a separate window.

**5.6 — Run the idempotent setup script**
```bash
bash /data/scripts/setup.sh
```
Safe to run at any time — every check is designed to `SKIP` if already satisfied. A second consecutive run with zero non-`SKIP` lines is the expected, healthy result.
⚠ As of 2026-06-30, this script's security-check section is mid-refactor (moving from ad-hoc per-check logic to a single rules-based `verify_baseline()` function — see Open Items). Confirm the refactor is complete and re-tested for idempotency before fully trusting its SSH-hardening line.

**5.7 — Check WireGuard peer status**
```bash
sudo wg show
```
Look for a `latest handshake:` line under the peer entry. No handshake line means no client has connected recently — not necessarily broken, but worth investigating if you expected one.

---

## 6. Troubleshooting Guide

**SSH connection refused**
First diagnostic: are you even able to reach the host at all? `ping <public-ip>`. If that fails, check the AWS Console — confirm the instance is running and the public IP hasn't changed (no Elastic IP is currently attached). If ping succeeds but SSH refuses, check UFW (§5.3) for an accidental rule change, and check `sudo systemctl status sshd` via the WireGuard tunnel or AWS Session Manager as a fallback access path.

**Prometheus targets showing down**
```bash
curl -s 'localhost:9090/api/v1/targets' | grep -o '"health":"[a-z]*"'
```
If `node` shows down but `prometheus` shows up: check `systemctl status node_exporter`. If both show down: check `systemctl status prometheus` itself — the scrape engine may be stopped, not just one target.

**Grafana dashboard blank / no data**
Confirm the SSH tunnel is actually active (§5.5) — a blank dashboard with no error is very often just a missing tunnel, not a real fault. If the tunnel is confirmed active, check the data source: Grafana UI → Configuration → Data Sources → Prometheus → Save & Test. If that fails, Prometheus itself may be down — check the section above.

**Disk space warning**
```bash
df -h | grep -E '^/dev/root|Filesystem'
```
⚠ **Known, ongoing condition as of 2026-06-30: root disk usage has fluctuated between 63% and 94.6% over this server's history, currently sitting around 87–88%.** This is structural — the desktop environment (XRDP/lightdm stack) in `/usr` accounts for the majority of root disk usage on an 8GB root volume, not a leak. Mitigation: `sudo apt-get autoremove --purge -y` reclaims old kernel images reliably; run this any time more than one `linux-image-*-aws` package is installed (`dpkg -l | grep linux-image` to check). This is not yet permanently resolved — treat it as an item requiring periodic attention, not a closed issue.

**WireGuard handshake never appears**
Compare the public key in the server's `[Peer]` block against the actual key the client app is using — **do not assume they match.** On 2026-06-28, the WireGuard Android app silently generated its own keypair instead of using a pre-generated one, and the resulting key mismatch produced exactly this symptom: tunnel shows "connected," `tx` bytes increase, but no handshake ever completes. Verify with `sudo wg show` on the server and the app's own "Public key" field side by side, byte for byte.

---

## 7. Recovery Procedures

**Instance rebooted and services do not start**
Reconnect via SSH (or WireGuard, if SSH is unreachable — confirmed as a working second path, 2026-06-28). Run:
```bash
systemctl is-active prometheus node_exporter grafana-server fail2ban wg-quick@wg0
```
Any service showing `inactive` rather than `active`: check `systemctl is-enabled <service>` first. A service that is correctly built but never `enable`d will run perfectly when started manually and silently stay down forever after every reboot — this exact failure mode happened with WireGuard on this server (see Change Log). Enable, then start, then re-verify with a second reboot before trusting the fix.

**The firewall locks you out**
Do not panic-edit UFW rules over a connection you may lose mid-command. Use the AWS Console's Session Manager (no SSH required) or the WireGuard tunnel as an out-of-band path, run `sudo ufw status verbose`, identify the offending rule, and remove it precisely (`sudo ufw delete <rule-number>`) rather than disabling UFW entirely.

**Data volume unmounted or fails to mount**
```bash
sudo blkid /dev/nvme1n1                      # confirm UUID matches fstab
sudo mount -a                                  # attempt remount from fstab
df -h | grep /data
```
If `mount -a` errors citing an unrecognized filesystem type, **count the fields in the fstab line** before assuming the value itself is wrong — a missing field shifts every subsequent value left by one position and produces exactly this misleading error. This precise failure occurred during this server's setup (INC-001) and cost roughly 25 minutes to diagnose the first time; it should cost under two minutes now that the pattern is documented.

---

## 8. Change Log

| Date | Change |
|---|---|
| Pre-2026-06-19 | Initial hardening (inherited, predates this runbook): SSH key auth, non-root user `ghostadmin`, UFW, fail2ban, WireGuard server-side config, XRDP scoped to VPN subnet. |
| 2026-06-21 | M-01 Sessions 1–3: audited and confirmed inherited hardening intact. Discovered UFW (not raw nftables) manages the firewall — ADR-001 authored. Discovered `ens5` interface naming (not `eth0`). |
| 2026-06-21 | M-01 Session 4: 20GB EBS volume (`nvme1n1`) created in eu-west-2a, formatted ext4, mounted at `/data`. **INC-001** — fstab entry initially missing its filesystem-type field, causing `mount -a` to fail with a misleading "unknown filesystem type" error; root cause found by counting fields against the standard fstab format, fixed via `sed`. |
| 2026-06-23 | M-01 Session 5: Prometheus 2.54.0 + Node Exporter 1.8.2 deployed; both targets confirmed `"health":"up"`. |
| 2026-06-25 | M-01 Session 6: Grafana 13.1.0 deployed, data path redirected to `/data/grafana` pre-install, Node Exporter Full dashboard (community ID 1860) imported and confirmed rendering live data. |
| 2026-06-26 | M-01 Session 7: Bash scripting practice. Two real bugs found and fixed: a script writing its log to the wrong filename despite reporting success, and a confirmed-working `ERR` trap on a deliberate failure (line 27). |
| 2026-06-28 | Disk cleanup; WireGuard peer added (gap flagged 2026-06-21, closed here). **INC-002** — WireGuard Android app silently generated its own keypair instead of using the pre-generated one; diagnosed via byte-for-byte public key comparison, fixed by updating the server's `[Peer]` block. **INC-003** — `wg-quick@wg0` found disabled at boot (ran correctly when started manually, never persisted across a restart); enabled, then verified surviving a second, deliberate cold reboot. Kernel updated 1015 → 1019. |
| 2026-06-28 | M-01 Session 8: idempotent `setup.sh` written, covering Sessions 1–7 plus the WireGuard boot-enable fix as a permanent, scripted check. Verified genuinely idempotent — second consecutive run produced 100% `SKIP` lines. |
| 2026-06-30 | Discovered `setup.sh`'s SSH-hardening check produced a false `WARNING` despite `permitrootlogin` being correctly set to `no` — root cause isolated to the script's own grep pattern, not the server. **Refactor in progress**, moving from scattered ad-hoc checks to a single rules-based `verify_baseline()` function (see Open Items). |
| 2026-06-30 | M-01 Session 9: this runbook authored. |

### Open Items (carried forward, not yet closed)
- `setup.sh` rules-based refactor (`verify_baseline()`) — in progress as of this document's authorship. Re-run the two-pass idempotency test once complete before considering it closed.
- Root disk usage — structurally elevated (63–94.6% observed range), mitigated but not permanently resolved. Revisit if it approaches 95%+ again.
- fail2ban's `jail.local` contents — never directly read this session; confirm before relying on assumed `maxretry`/`bantime` values.
- Ashby's Law audit (Pre-Foundation Lesson 8) — total known failure modes vs. documented runbook responses has not yet been formally counted. This document is a direct contribution toward closing that gap, but the explicit count has not been done.

---

## 9. Successor Handover Checklist

A new engineer taking over this server should confirm every item below before considering themselves operationally independent.

- [ ] Can connect via `ssh empire` (or has regenerated the SSH config alias with current IP and key path)
- [ ] Has confirmed the current public IP has not drifted from §1 (no Elastic IP is attached — this can change on stop/start)
- [ ] Can run `sudo ufw status verbose` and explain what each of the three rules permits and why
- [ ] Can connect to Grafana via SSH tunnel and sees live data on the Node Exporter Full dashboard
- [ ] Has run `sudo wg show` and confirmed a recent handshake exists for at least one peer
- [ ] Has read INC-001, INC-002, and INC-003 in full and understands what each failure actually was, not just that it was "fixed"
- [ ] Has run `/data/scripts/setup.sh` twice in succession and confirmed the second run is fully idempotent (all `SKIP` lines)
- [ ] Is aware of the current root-disk usage and the `autoremove --purge` mitigation procedure
- [ ] Has reviewed the Open Items list above and knows which are still genuinely outstanding, not assumed resolved
- [ ] Has access to (or has been added as a collaborator on) all four GitHub repositories: `SIOE-X-Engineering-Log`, `sioe-x-infrastructure`, `sioe-x-applications`, `Sovereign-portfolio`

---

*This document satisfies the M-01 Session 9 deliverable per SP-1 and is assessed under Vol 6's Documentation Quality rubric category. A sanitized copy — public IP, keys, and UFW source restrictions removed. A complete version belongs in my  private git - repo, `SIOE-X-Engineering-Log` not this one.
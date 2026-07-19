# netplan-wifi

A single-file, interactive **Wi-Fi + Ethernet switcher for Ubuntu Server** (netplan + systemd-networkd + cloud-init).

Most headless Ubuntu boxes have no GUI network picker — changing Wi-Fi means hand-editing
YAML. `netplan-wifi` gives you a friendly terminal menu instead: it **scans** for nearby
networks, lets you **pick one**, **remembers passwords**, and writes the config so it
**survives reboots**. It can also configure **all** your saved networks at once so the
machine automatically connects to whichever is in range and fails over if one drops.

**Ethernet is always configured** with a lower route-metric than Wi-Fi, so a plugged-in
cable is preferred automatically and the box falls back to Wi-Fi the moment it's unplugged
— no action needed. Run `wifi --status` any time to see what's connected, whether you have
internet, and your IPs.

```
  Available networks:
    1) HomeNet                          -48 dBm  (saved)
    2) Office-5G                         -61 dBm
    3) CoffeeShop                        -72 dBm
    4) <enter SSID manually / hidden network>
    r) rescan / refresh

  Choose a network [1-4, or r to rescan]:
```

> ⚠️ **Scope:** This targets Ubuntu/Debian systems that use **netplan with the
> `systemd-networkd` renderer and cloud-init** (the default on Ubuntu Server / cloud images).
> It is **not** for desktops running NetworkManager. See [Requirements](#requirements).

---

## Features

- **Scan & connect** — lists nearby SSIDs (sorted by signal) via `iw`; pick by number.
- **One-and-done** — pick a network, done. No "use saved password?", "apply?", or
  "enable failover?" prompts — the sensible answers are the defaults, so you're connected in
  a single choice.
- **Ethernet preferred** — a wired interface is always configured (DHCP, `optional`, lower
  route-metric). Plug in a cable → the box uses it; unplug → it falls back to Wi-Fi. Boot is
  never blocked when the cable is absent.
- **Status at a glance** — `wifi --status` shows connected/not, internet yes/no, which link
  is active, plus LAN and public IP.
- **Rescan/refresh** — press `r` to re-scan without restarting.
- **Hidden networks** — enter an SSID manually.
- **Saved passwords** — once you connect, the password is remembered (root-only store at
  `/etc/wifi-credentials`, mode `600`) and reused automatically. Saved networks show a
  `(saved)` tag. Manage them with `--list` and `--forget`.
- **Masked password entry** — shows `*` as you type, supports backspace, and **Tab toggles
  reveal** so you can verify what you typed.
- **Automatic failover (on by default)** — connecting writes *every* saved network into the
  config. The system (wpa_supplicant) then joins whichever is in range and fails over
  automatically if one goes offline — no daemon, no polling, works across reboots. `--auto`
  reapplies the same across all saved networks on demand.
- **Reboot-persistent** — writes cloud-init's network *source* file so the setting is
  re-applied on every boot, plus the live netplan file for immediate effect.
- **Safe by default** — backs up the existing netplan file (timestamped) before every change
  and prints the exact restore command.
- **Self-installing** — the first run installs the `wifi` command onto your `PATH` and
  installs its one dependency (`iw`).

---

## Install

Clone and run it once — it installs itself (copies to `/usr/local/bin/wifi`, installs `iw`):

```bash
git clone https://github.com/<your-username>/netplan-wifi.git
cd netplan-wifi
sudo ./wifi --install
```

After that, just run `wifi` from anywhere.

> The script re-executes itself with `sudo` automatically if you don't start it as root —
> writing netplan config and scanning the radio both require root.

---

## Usage

```
wifi                  Scan and connect to a network (interactive, one-and-done)
wifi --status         Show connection / internet / IP status
wifi --auto           Configure ALL saved networks for auto connect/failover
wifi --list           List saved networks
wifi --forget [SSID]  Forget a saved network (interactive picker if no SSID)
wifi --install        (Re)install the 'wifi' command + dependencies
wifi --help           Show help
```

### Connect to a network

```bash
wifi
```
Pick a number (or `r` to rescan). If the network is new, type the password — it's masked
with `*`; press **Tab** to show/hide it. That's it: no confirmation prompts — it applies,
persists, saves the password, and enables failover across your saved networks automatically.
Reconnecting to a saved network needs just the one menu choice.

### Check status

```bash
wifi --status
```
```
  === Network status ===
  Status    : CONNECTED  (via Ethernet (enp0s31f6))
  Wi-Fi     : connected to "HomeNet"
  Ethernet  : cable connected
  Internet  : yes
  LAN IP    : 192.168.1.112
  Public IP : 203.0.113.9
```

### Automatic connect / failover across saved networks

```bash
wifi --auto
```
Writes all your saved networks into one config block. The machine will join whichever is
available and **fail over automatically** when the current one disappears. Ideal for a
laptop/SBC that moves between a few known locations.

### Manage saved networks

```bash
wifi --list                 # show saved SSIDs
wifi --forget "Office-5G"   # forget one by name
wifi --forget               # interactive picker
```

---

## How it works

`netplan-wifi` writes **two** files on apply:

| File | Purpose |
| --- | --- |
| `/etc/cloud/cloud.cfg.d/90-wifi.cfg` | **Persistent source.** cloud-init re-renders this into the netplan file on every boot, so your setting survives reboots and image re-runs. |
| `/etc/netplan/50-cloud-init.yaml` | **Live config.** Backed up first (`*.bak-<timestamp>`), then rewritten and applied immediately with `netplan generate && netplan apply`. |

Both files are written mode `600`.

Saved passwords live in `/etc/wifi-credentials` (mode `600`, root-only), base64-encoded so
SSIDs/passwords with spaces or special characters round-trip cleanly. **This file is never
committed** (see [Security](#security)).

Automatic failover is native, not scripted: netplan lets one interface declare multiple
`access-points:`, and `wpa_supplicant` selects whichever is in range and roams when one
drops. Connecting emits all saved networks into that block (so failover is on by default);
`--auto` reapplies the same on demand.

### Ethernet preference

Every apply also emits an `ethernets:` block for the wired interface:

```yaml
  ethernets:
    enp0s31f6:
      dhcp4: true
      optional: true            # never blocks boot when no cable is plugged in
      dhcp4-overrides:
        route-metric: 100       # lower than Wi-Fi (600) => wired wins when present
```

Because the wired route-metric (`100`) is lower than Wi-Fi's (`600`), the kernel prefers the
cable whenever it has a link, and automatically falls back to Wi-Fi when it's unplugged. No
sidecar daemon (e.g. `dhcpcd`) is involved — it's all systemd-networkd via netplan.

### Interface names

The script defaults to Wi-Fi interface `wlp0s20f3` and wired interface `enp0s31f6`. If yours
differ, change the `IFACE=` / `ETH_IFACE=` lines at the top of `wifi` (find them with
`ip link` or `iw dev`).

---

## Requirements

- Ubuntu/Debian with **netplan** using the **`systemd-networkd` renderer** + **cloud-init**
  (default on Ubuntu Server and most cloud images).
- `iw` (auto-installed on first run via `apt-get`).
- `bash`, `coreutils` (`base64`, `seq`), standard on any Ubuntu/Debian install.
- Root (the script self-elevates with `sudo`).

Not supported: desktops managed by **NetworkManager** (use `nmcli`/`nmtui` there).

---

## Security

- **No credentials are stored in this repository.** Saved Wi-Fi passwords live only in
  `/etc/wifi-credentials` on your machine (root-only, mode `600`), well outside the repo.
- `.gitignore` additionally blocks any `*credentials*`, `*.secret`, netplan/cloud-init YAML,
  and backup files from ever being committed.
- The credential store is base64-encoded for safe parsing — **base64 is encoding, not
  encryption.** Anyone with root on the box can read saved passwords (same as the netplan
  file itself). Treat the machine's root access accordingly.

---

## License

[MIT](LICENSE) © 2026 Andre Millan

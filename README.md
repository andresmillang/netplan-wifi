# netplan-wifi

A single-file, interactive **Wi-Fi switcher for Ubuntu Server** (netplan + systemd-networkd + cloud-init).

Most headless Ubuntu boxes have no GUI network picker — changing Wi-Fi means hand-editing
YAML. `netplan-wifi` gives you a friendly terminal menu instead: it **scans** for nearby
networks, lets you **pick one**, **remembers passwords**, and writes the config so it
**survives reboots**. It can also configure **all** your saved networks at once so the
machine automatically connects to whichever is in range and fails over if one drops.

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
- **Rescan/refresh** — press `r` to re-scan without restarting.
- **Hidden networks** — enter an SSID manually.
- **Saved passwords** — once you connect, the password is remembered (root-only store at
  `/etc/wifi-credentials`, mode `600`). Saved networks show a `(saved)` tag and won't ask
  again. Manage them with `--list` and `--forget`.
- **Masked password entry** — shows `*` as you type, supports backspace, and **Tab toggles
  reveal** so you can verify what you typed.
- **Automatic failover** — `--auto` writes *every* saved network into the config. The system
  (wpa_supplicant) then joins whichever is in range and fails over automatically if one goes
  offline — no daemon, no polling, works across reboots.
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
wifi                  Scan and connect to a network (interactive)
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
Pick a number (or `r` to rescan). Type the password — it's masked with `*`; press **Tab**
to show/hide it. Confirm, and it applies + persists.

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
drops. `--auto` simply emits all saved networks into that block.

### Interface name

The script defaults to interface `wlp0s20f3`. If your Wi-Fi interface differs, change the
`IFACE=` line at the top of `wifi` (find yours with `ip link` or `iw dev`).

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

# HP t740 Home Server: Ubuntu Server 26.04 LTS from Scratch

A complete, step-by-step record of turning a refurbished **HP t740 thin client** into a headless home server running **Ubuntu Server 26.04.1 LTS**, hardened for SSH access, reachable from anywhere with **Tailscale**, and ready for **Docker** workloads (FastAPI apps, Redis/Valkey, PostgreSQL and similar).

Every step below was done on real hardware in October 2026, including the problems I hit and how I fixed them.

## End result

- Ubuntu Server 26.04.1 LTS on a 512 GB M.2 SATA SSD, booting in seconds
- No monitor or keyboard needed: managed over SSH from a Linux Mint laptop
- SSH with keys only, password login disabled
- `ufw` firewall on, automatic security updates on
- Fixed LAN IP via a DHCP reservation
- Remote access from anywhere through Tailscale, with no ports opened on the router
- Docker Engine and the Compose plugin from Docker's official repository

## Values used in this guide

Replace these with your own wherever they appear.

| What | Value in this guide |
| --- | --- |
| Server hostname | `nubecode1` |
| Server username | `nubecode` |
| Server LAN IP | `192.168.68.59` |
| Ethernet interface | `enp1s0f0` |
| Wi-Fi interface | `wlo1` |
| Client machine | Linux Mint (Ubuntu 24.04 "noble" base) |

> **Public repo warning:** never commit your Wi-Fi password, MAC addresses, private keys or Tailscale auth keys. Every secret in this guide is a placeholder.

## Contents

1. [Hardware](#1-hardware)
2. [Hardware preparation](#2-hardware-preparation)
3. [Create the USB installer on Linux Mint](#3-create-the-usb-installer-on-linux-mint)
4. [BIOS keys and settings](#4-bios-keys-and-settings)
5. [Install Ubuntu Server](#5-install-ubuntu-server)
6. [First boot](#6-first-boot)
7. [Network configuration and faster boot](#7-network-configuration-and-faster-boot)
8. [Reserve the server's IP address](#8-reserve-the-servers-ip-address)
9. [SSH keys and SSH hardening](#9-ssh-keys-and-ssh-hardening)
10. [Firewall with ufw](#10-firewall-with-ufw)
11. [Automatic security updates](#11-automatic-security-updates)
12. [Remote access with Tailscale](#12-remote-access-with-tailscale)
13. [Install Docker Engine](#13-install-docker-engine)
14. [Smoke test with Docker Compose](#14-smoke-test-with-docker-compose)
15. [Everyday commands](#15-everyday-commands)
16. [Troubleshooting](#16-troubleshooting)
17. [What's next](#17-whats-next)
18. [References](#18-references)

---

## 1. Hardware

| Part | Details |
| --- | --- |
| Machine | HP t740 Thin Client (refurbished) |
| CPU | AMD Ryzen Embedded V1756B, 4 cores |
| GPU | AMD Radeon Vega 8 (integrated) |
| RAM | DDR4 SO-DIMM, 2 slots (under the perforated metal shield) |
| Boot drive | Intenso M.2 SSD SATA III Top, 512 GB, M.2 2280, B+M key |
| Spare storage | Built-in eMMC module (64 GB on my unit, shows as about 58 GB), left unused |
| Ethernet | Realtek RTL8111/8168 Gigabit (`enp1s0f0`) |
| Wi-Fi | Intel Wireless-AC 9x6x "Thunder Peak" (`wlo1`) |
| Video out | 4 × DisplayPort 1.2 on the back. No HDMI. |
| Power | HP 90 W Smart adapter, 19.5 V / 4.62 A, 4.5 × 3.0 mm blue tip |
| Expansion | 1 × PCIe slot (not used here) |

### M.2 slot map

The two storage slots sit next to each other at the bottom of the board, and the board labels them:

| Board label | Accepts | Notes |
| --- | --- | --- |
| `EMMC` | eMMC module **or** an NVMe SSD, 2230 to 2280 | The only slot that takes NVMe. Ships with the eMMC module. |
| `SATA` | M.2 **SATA** SSD only, 2230 to 2280 | Empty on my unit. This is where the Intenso SSD went. |

A third, smaller M.2 slot (2230) holds the Wi-Fi card.

**How to tell SATA from NVMe drives:** look at the gold connector. Two notches (B+M key) means SATA. One notch (M key) means NVMe. An NVMe drive will not work in the `SATA` slot.

---

## 2. Hardware preparation

### 2.1 Display: getting a picture at all

My first problem was a black screen even though the power light was on. Things to check, in order:

- **Use a plain DisplayPort-to-DisplayPort cable** into one of the rear DP ports. Cheap passive DP-to-HDMI adapters are a common reason thin clients show nothing.
- **Set the monitor's input to DisplayPort by hand** in its menu instead of auto-detect.
- **If the monitor has a "DP version" setting**, try DP 1.1 and turn off MST/daisy-chain.
- **Wait 2 to 3 minutes on the first boot.** Ryzen systems retrain memory after a RAM change, and the screen stays black meanwhile.
- **Watch the power LED.** A repeating blink pattern is an HP error code. Count the blinks and look the code up in the HP hardware reference guide.

### 2.2 Power adapter

The t740 needs HP's **90 W Smart adapter: 19.5 V, 4.62 A, blue tip with a 4.5 × 3.0 mm barrel** (HP spare part `L64042-001`).

- Most genuine HP 90 W blue-tip laptop chargers also work. Check the label: 19.5 V and 90 W (or 4.62 A).
- A 65 W blue-tip charger fits the socket but is underpowered. Avoid it.
- The "Smart" part is an ID pin in the centre of the plug that tells the machine what adapter is connected. Cheap no-name adapters sometimes skip it.
- The adapter's mains socket is a **C5 "cloverleaf"** (3-pin, earthed). The cable you need is **Schuko (EU plug) to C5**, not the trapezoid C13 "kettle" cable used for PCs and monitors. My third-party adapter shipped with the wrong one.
- If you use a third-party adapter, check how warm it gets during the first few days. Warm is normal. Too hot to hold is not.

### 2.3 Install the SSD

1. Unplug power and open the case (see HP's *t740 Hardware Reference Guide* for your unit's access panel).
2. Find the slot labelled `SATA` next to the eMMC module.
3. Slide the SSD into the connector at a slight angle, then press it down flat.
4. Screw it down at the far end, about 80 mm from the connector. On my board, the existing screw sat on a standoff closer to the connector, so it had to move to the 2280 position.

You don't need to touch the RAM shield, the PCIe slot or the coin cell.

> **About the CR2032 coin cell:** it powers the clock and keeps BIOS settings while the machine is unplugged. Replace it only if the date or BIOS settings reset every time you unplug the power. Removing it for about 5 minutes is also how you reset the BIOS to defaults.

### 2.4 Check the drive is detected

Power on and press **F10** to open the BIOS. The SSD should be listed under storage information. You can also run **F2** (HP PC Hardware Diagnostics UEFI).

Expected messages **before** an operating system is installed:

| Message | Meaning |
| --- | --- |
| `Boot Device Not Found ... Hard Disk - (3F0)` | No bootable OS on any drive. Normal for a blank unit. |
| `No hard drive installed` in HP diagnostics | Shown on my unit with only the eMMC fitted. The diagnostics don't treat the eMMC as a hard drive. |
| F9 boot menu shows only `IPv4/IPv6 (TFTP/HTTP)` network entries | The SSD is blank, so there's nothing to boot from it yet. |

---

## 3. Create the USB installer on Linux Mint

### 3.1 Download the ISO

Go to <https://ubuntu.com/download/server> and download **Ubuntu Server 26.04.1 LTS** for **Intel or AMD 64-bit architecture** (amd64). The file is about 3 GB:

```text
ubuntu-26.04.1-live-server-amd64.iso
```

LTS means five years of free security and maintenance updates. Ignore the ARM, POWER and RISC-V downloads; the t740 is amd64.

### 3.2 Verify the download (recommended)

```bash
cd ~/Downloads
wget https://releases.ubuntu.com/26.04/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing
```

Expected output:

```text
ubuntu-26.04.1-live-server-amd64.iso: OK
```

### 3.3 Write the ISO to a USB stick

Use a stick of 4 GB or more. **Everything on it will be erased.** You don't need to format it first, because writing the image replaces the whole stick.

Rufus is Windows-only. On Linux Mint, use the built-in **USB Image Writer**:

1. Plug in the USB stick.
2. Right-click the `.iso` file and choose **Make bootable USB stick** (or open **USB Image Writer** from the menu).
3. Select the ISO and the USB stick, click **Write**, and enter your password.
4. Wait for the success message, then eject the stick.

Terminal alternative:

```bash
lsblk    # identify the USB stick, e.g. /dev/sdb. Double-check: the wrong letter wipes that disk.
sudo dd if=ubuntu-26.04.1-live-server-amd64.iso of=/dev/sdX bs=4M status=progress conv=fsync
```

Replace `sdX` with your stick's device name (the whole disk, not a partition like `sdb1`).

To reuse the stick for files later, Mint's **USB Stick Formatter** can format it back to FAT32.

---

## 4. BIOS keys and settings

| Key at power-on | Opens |
| --- | --- |
| **F10** | BIOS setup |
| **F9** | One-time boot menu |
| **F2** | HP PC Hardware Diagnostics |
| **Esc** | Startup menu listing the keys above |

Settings worth checking in **F10**:

- **USB storage boot: enabled.** Thin clients sometimes ship with it off. If the USB stick doesn't appear in the F9 menu, this is why.
- **Network (PXE) boot: optional to disable,** so the machine stops trying to boot from the network first.
- **Secure Boot: leave it on.** Ubuntu boots fine with it.

Save and exit.

---

## 5. Install Ubuntu Server

Plug the USB stick into a **rear USB 3** port, power on, press **F9**, and choose the `UEFI: <your USB stick>` entry. In the boot menu that follows, pick **Try or Install Ubuntu Server**.

**Installer controls:** arrow keys or Tab to move, **Space** to tick or untick a box, **Enter** to select. The screen order below is what I saw on 26.04.1. It may vary slightly.

### 5.1 Language and keyboard

Pick your language and keyboard layout.

### 5.2 Type of installation

```text
(X) Ubuntu Server
( ) Ubuntu Server (minimized)
[ ] Search for third-party drivers
```

- Keep **Ubuntu Server**. The minimized version removes everyday tools like editors and man pages, which is frustrating on a box you're learning on.
- Leave **Search for third-party drivers** unticked. The t740's AMD graphics, Realtek Ethernet and Intel Wi-Fi all work with the standard kernel.

### 5.3 Network configuration

The installer lists both interfaces:

```text
enp1s0f0   eth    not connected   (Realtek RTL8111/8168)
wlo1       wlan   not connected   (Intel Wireless-AC 9x6x)
```

**Option A, Ethernet (most reliable):** plug a cable from the router into the t740. `enp1s0f0` gets an IP address within a few seconds. If it doesn't, select it → **Edit IPv4** → **Automatic (DHCP)**.

**Option B, Wi-Fi:** select `wlo1` → **Edit Wifi**, pick or type your network name (SSID), enter the password, choose **Save**, and wait until `wlo1` shows an address like `192.168.x.x`.

Also set `enp1s0f0` → **Edit IPv4** → **Automatic (DHCP)** even without a cable, so a cable works later without editing files.

> ⚠️ **Known issue I hit:** right after saving the Wi-Fi details, the installer crashed with *"Sorry, an unknown error occurred"*. Nothing had been written to disk yet. I chose **Close report**, rebooted (holding the power button when the screen stuck on "Rebooting"), booted the USB again, and the second Wi-Fi attempt worked. If it keeps crashing, use Ethernet, Android **USB tethering** (seen as a wired network), or **Continue without network** and set up Wi-Fi after the install as shown in [section 7.1](#71-netplan-make-ethernet-optional).

Avoid **Continue without network** if you can: the installer can't download anything (including OpenSSH) without it.

Press **Done**.

### 5.4 Proxy

Leave **Proxy address** blank. Proxies are only needed on some company networks. Press **Done**.

### 5.5 Ubuntu archive mirror

Keep the default mirror. The installer tests the connection; press **Done** when it passes.

### 5.6 Guided storage configuration

```text
(X) Use an entire disk
    [ Intenso_SSD_... local disk 476.939G ]
    [X] Set up this disk as an LVM group
        [ ] Encrypt the LVM group with LUKS
( ) Custom storage layout
```

- **Choose the SSD**, not the eMMC. 512 GB shows as about 476.9 GB because the installer uses binary units.
- **LVM on.** The Logical Volume Manager puts a flexible layer between the disk and your filesystems, so you can resize volumes, add a second disk or take snapshots later without reinstalling.
- **Encryption off.** LUKS would ask for a passphrase at every boot, which is impractical for a headless server you reboot remotely.

Press **Done**.

### 5.7 Storage summary

Check the size of `ubuntu-lv`. The guided layout sometimes gives the root volume only part of the disk. If it's much smaller than the volume group, select `ubuntu-lv` → **Edit**, set **Size** to the maximum shown, and **Save**. On my install it already used the full 473.886 GB.

Expected layout:

| Mount point | Size | Type | Lives on |
| --- | --- | --- | --- |
| `/boot/efi` | 1 GB | fat32 | SSD partition 1 |
| `/boot` | 2 GB | ext4 | SSD partition 2 |
| `/` | about 474 GB | ext4 | LVM: `ubuntu-vg/ubuntu-lv` on SSD partition 3 |

The summary also lists:

- **The eMMC** (shown by an ID like `0x9c861d1b`, about 58 GB) under *Available devices*. It is not touched.
- **The USB stick** (e.g. `SanDisk Cruzer Blade`) as *in use*, because the installer runs from it. It is not erased.

Press **Done**, then **Continue** on the "destructive action" warning. Only the SSD is wiped.

### 5.8 Profile

Enter your name, the **server name** (hostname, e.g. `nubecode1`), a **username** and a **password**. Remember the password: you need it for `sudo`, and for console login if you ever lock yourself out of SSH.

### 5.9 Ubuntu Pro

Choose **Skip for now**. You can enable it later; it's free for personal use on up to five machines.

### 5.10 SSH configuration

```text
[X] Install OpenSSH server
[X] Allow password authentication over SSH
[ Import SSH key ]
```

- **Tick Install OpenSSH server** (Space). Without it you'd need the monitor and keyboard for everything.
- **Keep password authentication on for now.** Section 9 replaces it with keys.
- **Import SSH key** is optional. If your GitHub account has SSH public keys, choose **from GitHub** and enter your username; your laptop can then log in without a password from the first boot.

Press **Done**.

### 5.11 Featured server snaps

Tick **nothing** and press **Done**. In particular, don't install Docker from this list: the snap version has quirks. Docker comes from Docker's official repository in section 13, and services like Prometheus or Nextcloud are cleaner to run as containers later.

### 5.12 Install and reboot

The installation runs for a few minutes. When **Reboot Now** appears, select it, remove the USB stick when prompted, and press **Enter**.

If the screen stays on "Rebooting..." for more than a minute, press **Enter** once (it may be waiting for the stick to be removed). If nothing happens, **hold the power button for about 10 seconds**. That's safe at this point, because **Reboot Now** only appears after the installation has finished.

---

## 6. First boot

Power on without the USB stick. You'll see boot messages, then a login prompt:

```text
Ubuntu 26.04.1 LTS nubecode1 tty1

nubecode1 login:
```

Log in. The welcome banner shows disk usage, temperature and the IP address, for example:

```text
Usage of /:  1.6% of 465.38GB
IPv4 address for wlo1: 192.168.68.59
```

> If the boot pauses for about 2 minutes on `systemd-networkd-wait-online`, that's fixed in [section 7](#7-network-configuration-and-faster-boot). Just wait for the login prompt this time.

### 6.1 Find the IP address

```bash
ip -brief address
```

Note the `inet` address on `wlo1` (Wi-Fi) or `enp1s0f0` (Ethernet).

### 6.2 Switch to SSH

From now on, work from your laptop. On Linux Mint:

```bash
ssh nubecode@192.168.68.59
```

The first time, SSH shows the server's key fingerprint and asks to confirm. Type `yes`.

### 6.3 Update everything

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

### 6.4 Set the timezone

The server starts on UTC.

```bash
timedatectl list-timezones | grep Europe    # find yours
sudo timedatectl set-timezone Europe/Amsterdam
timedatectl                                  # check
```

---

## 7. Network configuration and faster boot

**Problem:** my first boots took several minutes. Two things caused it:

1. **`systemd-networkd-wait-online`** waits until *every* configured network interface is up. The Ethernet port had no cable, so it never came up, and the service waited until its timeout.
2. **cloud-init**, a provisioning tool for cloud VMs, also waits for the network on every boot. A home server doesn't need it.

### 7.1 Netplan: make Ethernet optional

Ubuntu Server's network configuration lives in **netplan** YAML files:

```bash
ls /etc/netplan/
sudo nano /etc/netplan/50-cloud-init.yaml
```

Your file will look similar to this. Add `optional: true` under the Ethernet interface (YAML uses spaces, never tabs):

```yaml
network:
  version: 2
  ethernets:
    enp1s0f0:
      dhcp4: true
      optional: true          # don't block boot when no cable is plugged in
  wifis:
    wlo1:
      dhcp4: true
      access-points:
        "YOUR_WIFI_NAME":
          password: "YOUR_WIFI_PASSWORD"
```

If you skipped networking during the install, create the `wifis:` part yourself in the same way.

The file holds your Wi-Fi password, so make it readable by root only (netplan warns otherwise):

```bash
sudo chmod 600 /etc/netplan/*.yaml
```

Apply it safely:

```bash
sudo netplan try
```

`netplan try` applies the change and rolls it back automatically after 120 seconds unless you press **Enter** to keep it, so a typo can't cut off your SSH session for good. `sudo netplan apply` applies without the safety net.

### 7.2 Count the network as online once any interface is up

First check the original command path:

```bash
systemctl cat systemd-networkd-wait-online.service | grep ExecStart
```

Then create an override:

```bash
sudo systemctl edit systemd-networkd-wait-online.service
```

The editor opens with comment lines. Put these lines **between** the two marker comments near the top (`### Anything between here and the comment below will become the contents of the drop-in file` and the line after it), using the path you found above:

```ini
[Service]
ExecStart=
ExecStart=/usr/lib/systemd/systemd-networkd-wait-online --any --timeout=30
```

The empty `ExecStart=` clears the original command before setting the new one. `--any` means one working interface is enough, and `--timeout=30` caps the wait at 30 seconds.

Save and close. `systemctl edit` reloads systemd for you.

### 7.3 Disable cloud-init

```bash
sudo touch /etc/cloud/cloud-init.disabled
```

The netplan file it generated stays in place and keeps working.

### 7.4 Measure the result

```bash
sudo reboot
# after reconnecting:
systemd-analyze
systemd-analyze blame | head -n 10
```

`systemd-analyze` prints the total boot time; `blame` lists the slowest services. After these fixes my boot went from several minutes to a few seconds of userspace time.

---

## 8. Reserve the server's IP address

Your router hands out IP addresses with DHCP and may give the server a different one after a reboot. A **DHCP reservation** tells the router to always give the server the same address. Nothing changes on the server.

### 8.1 Find the device that hands out addresses

```bash
ip route | grep default
```

The address after `via` is your gateway, the device running DHCP for this network. In my case it's `192.168.68.1`, the main unit of a mesh Wi-Fi system, not the ISP's router.

### 8.2 Find the server's MAC addresses

```bash
ip link show wlo1        # Wi-Fi: the value after link/ether
ip link show enp1s0f0    # Ethernet: a different MAC
```

### 8.3 Add the reservation

In the admin page or app of the device from 8.1, find **Address Reservation**, **DHCP Reservation** or **Static Lease**, usually under LAN or DHCP settings. Add the Wi-Fi MAC with IP `192.168.68.59`. Add the Ethernet MAC too if you might switch to a cable later.

On a TP-Link Deco mesh (default range `192.168.68.x`): **Deco app → More → Advanced → Address Reservation**.

Why not set a static IP in netplan instead? If it overlaps with the range the router hands out, another device can get the same address. The reservation avoids that conflict.

### 8.4 Mesh in router mode: a common trap

If your mesh runs in **router mode** behind the ISP router, it creates a separate network (here `192.168.68.x`). Devices on the ISP router's network (e.g. `192.168.1.x` or `192.168.178.x`) usually **can't reach** the server. Fix it by connecting your laptop to the mesh, by switching the mesh to **Access Point mode**, or by using Tailscale (section 12), which doesn't care about either.

---

## 9. SSH keys and SSH hardening

Passwords can be guessed; keys can't. This section switches SSH to key-only login.

### 9.1 Create a key on your laptop (Linux Mint)

Skip this if `~/.ssh/id_ed25519` already exists.

```bash
ssh-keygen -t ed25519 -C "you@your-laptop"
```

Accept the default path. A passphrase is recommended; it protects the key if your laptop is stolen.

### 9.2 Copy the public key to the server

```bash
ssh-copy-id nubecode@192.168.68.59
```

This appends your public key to `~/.ssh/authorized_keys` on the server. You type the server password one last time.

### 9.3 Test key login

```bash
ssh nubecode@192.168.68.59
```

It should log in without asking for the **server** password (it may ask for your **key passphrase**, which is different).

### 9.4 Optional: a short alias

On the laptop, create or edit `~/.ssh/config`:

```text
Host t740
    HostName 192.168.68.59
    User nubecode
    IdentityFile ~/.ssh/id_ed25519
```

Now `ssh t740` works. After setting up Tailscale you can change `HostName` to `nubecode1`.

### 9.5 Disable password login on the server

**Keep your current SSH session open** until 9.6 passes, so a mistake can't lock you out.

```bash
sudo tee /etc/ssh/sshd_config.d/00-hardening.conf > /dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF

sudo sshd -t && sudo systemctl restart ssh
```

`sshd -t` checks the configuration for errors before the restart.

Why the `00-` prefix: the installer created `/etc/ssh/sshd_config.d/50-cloud-init.conf` containing `PasswordAuthentication yes`. sshd reads these files in alphabetical order and **the first value it finds wins**, so `00-hardening.conf` overrides it.

### 9.6 Verify

On the server:

```bash
sudo sshd -T | grep -Ei '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin)'
```

All three should say `no`.

From the laptop, in a **new** terminal, force a password attempt:

```bash
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password nubecode@192.168.68.59
```

Expected: `Permission denied (publickey).` Then check normal key login still works with `ssh nubecode@192.168.68.59`.

> **Locked out?** This setting only affects SSH. You can always log in with your password at the server's own screen and keyboard and fix the file.

---

## 10. Firewall with ufw

`ufw` (Uncomplicated Firewall) is installed but inactive by default. Allow SSH **before** enabling it:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

Expected status: active, with `22/tcp (OpenSSH)` allowed.

> **Docker bypasses ufw.** Ports you publish from containers (`-p 8000:8000` or `ports:` in Compose) are opened by Docker's own firewall rules, which ufw doesn't see. See [section 13.7](#137-docker-and-the-firewall) for how to handle it.

---

## 11. Automatic security updates

Ubuntu Server installs and enables `unattended-upgrades` by default. Confirm it:

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
```

Both lines should end in `"1"`:

```text
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

Useful checks:

```bash
systemctl status unattended-upgrades          # service running
ls /var/log/unattended-upgrades/              # what was installed and when
cat /var/run/reboot-required 2>/dev/null      # present = a kernel update needs a reboot
sudo unattended-upgrade --dry-run --debug     # simulate a run
```

### Optional: reboot automatically when needed

Kernel updates only take effect after a reboot. To let the server reboot itself at 04:00 when an update requires it:

```bash
sudo tee /etc/apt/apt.conf.d/52unattended-upgrades-local > /dev/null <<'EOF'
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
EOF
```

Files in `apt.conf.d` are read in order, so `52-...` overrides the defaults in `50unattended-upgrades` without editing that file.

By default only Ubuntu's own security updates are installed automatically. Packages from third-party repositories, such as Docker and Tailscale, update when you run `sudo apt upgrade`.

---

## 12. Remote access with Tailscale

### 12.1 Why

Right now SSH only works from inside the home network. Home routers block unsolicited incoming connections, and with a mesh in router mode there are **two** routers in the way, so classic port forwarding would have to be set up on both.

**Never forward port 22 to the internet.** An exposed SSH port gets automated login attempts within minutes.

**Tailscale** builds a private, encrypted network (a "tailnet") between your own devices on top of WireGuard. It works through double routers, opens no ports, and is free for personal use. Every device that should reach the server must run Tailscale and be logged in to the same account.

### 12.2 Install on the server

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

`tailscale up` prints a login URL. Open it in a browser on any device and sign in. Then:

```bash
tailscale ip -4      # the server's Tailscale address, 100.x.y.z
tailscale status     # devices in your tailnet
```

### 12.3 Install on your laptop (Linux Mint)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Log in with the **same account** as on the server.

> **Mint gotcha:** the install script stops if *any* apt repository on your system fails during `apt-get update`. On my laptop a broken signing key for a different repository (Cursor's) aborted it, and `tailscale` was reported as "command not found". Tailscale's own repository had been added fine, so this finished the job:
>
> ```bash
> sudo apt install -y tailscale
> sudo tailscale up
> ```
>
> Fix the broken repository separately (for Cursor, reinstalling its current `.deb` restores the key).

Apps also exist for Windows, macOS, Android and iOS.

### 12.4 Connect

```bash
ssh nubecode@nubecode1          # MagicDNS name
# or
ssh nubecode@100.x.y.z          # Tailscale IP from `tailscale status`
```

Both keep working wherever you are and whatever local IP the server has.

### 12.5 Disable key expiry for the server

By default a device's Tailscale login expires after a set period (180 days), and the server would drop off the tailnet while you're away. In the admin console at <https://login.tailscale.com/admin/machines>: open **nubecode1** → **⋯** → **Disable key expiry**.

### 12.6 Protect the account

Anyone who gets into the account you sign in to Tailscale with (Google, GitHub, Microsoft and so on) can add their own device to your tailnet. Turn on **two-factor authentication** for that account.

### 12.7 Test from outside

Install the Tailscale app on your phone, switch to **mobile data** (Wi-Fi off), and connect with an SSH app such as Termius to `nubecode1`.

### Notes

- No ufw rule is needed for Tailscale itself. SSH over Tailscale works because port 22 is already allowed.
- To allow everything from your own tailnet devices (useful later for web services), you can add `sudo ufw allow in on tailscale0`.
- To publish a web service to *anyone* without Tailscale on their side, Tailscale has a feature called **Funnel** (HTTP/HTTPS only).

---

## 13. Install Docker Engine

The plan: the host stays minimal, and everything else (Valkey/Redis, PostgreSQL, FastAPI apps, a reverse proxy, monitoring) runs as containers defined in Compose files. That keeps the host clean, makes the setup reproducible from Git, and mirrors how production systems run.

These steps follow Docker's official instructions for Ubuntu, which list **Ubuntu 26.04 (LTS)** as supported. Use Docker's packages, not Ubuntu's `docker.io` package or the snap.

### 13.1 Remove conflicting packages

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc docker-buildx podman-docker containerd runc | cut -f1)
```

On a fresh install, apt will likely report that none of these are installed. That's fine.

### 13.2 Add Docker's apt repository

```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```

The `Suites:` line fills in your Ubuntu codename automatically (`resolute` for 26.04).

### 13.3 Install Docker Engine and plugins

```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

| Package | What it is |
| --- | --- |
| `docker-ce` | The Docker Engine daemon |
| `docker-ce-cli` | The `docker` command |
| `containerd.io` | The container runtime Docker uses underneath |
| `docker-buildx-plugin` | Modern image builder (multi-stage, multi-platform builds) |
| `docker-compose-plugin` | `docker compose` (v2, no hyphen) |

### 13.4 Verify

```bash
sudo systemctl status docker      # should be active (running)
systemctl is-enabled docker       # should be enabled (starts at boot)
sudo docker run hello-world       # downloads a test image and prints a success message
docker compose version
```

### 13.5 Run Docker without sudo

```bash
sudo usermod -aG docker $USER
```

Log out and back in (or run `newgrp docker` in the current shell), then test:

```bash
docker run hello-world
```

> **Security note:** membership in the `docker` group is effectively root access on the host, since a container can mount the host filesystem. Only add accounts you'd trust with `sudo`.

### 13.6 Limit container log size

By default, container logs grow without limit, which slowly fills the disk on a server that runs 24/7. Cap them:

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

sudo systemctl restart docker
```

This applies to containers created **after** the change. Recreate existing ones (`docker compose up -d --force-recreate`) to apply it to them.

### 13.7 Docker and the firewall

Docker's documentation warns that ports you expose from containers **bypass ufw and firewalld rules**. Practical rules for this server:

| You want the service reachable from... | Publish it as |
| --- | --- |
| The server only (e.g. a database used by another container or a local script) | `127.0.0.1:6379:6379` |
| Your Tailscale devices only | `100.x.y.z:8000:8000` (the server's Tailscale IP) |
| Every device on the home network | `8000:8000` (binds to all interfaces) |
| Other containers only | No `ports:` at all. Containers in the same Compose project reach each other by service name. |

Because the server sits behind your router(s), "all interfaces" still means the home network, not the internet, unless you forward ports.

### 13.8 Updating Docker

Docker updates arrive through apt with everything else:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 14. Smoke test with Docker Compose

A small stack to confirm Compose, named volumes and a loopback-only port all work, using **Valkey** (the open-source Redis fork).

```bash
mkdir -p ~/stacks/valkey
cd ~/stacks/valkey
nano compose.yaml
```

```yaml
services:
  valkey:
    image: valkey/valkey:9.1-alpine
    container_name: valkey
    restart: unless-stopped
    command: ["valkey-server", "--appendonly", "yes"]   # persist data to disk
    ports:
      - "127.0.0.1:6379:6379"                          # reachable from the server only
    volumes:
      - valkey-data:/data

volumes:
  valkey-data:
```

Run it:

```bash
docker compose up -d
docker compose ps                                   # STATUS should be "Up"
docker exec -it valkey valkey-cli ping             # → PONG
docker exec -it valkey valkey-cli set hello world  # → OK
docker exec -it valkey valkey-cli get hello        # → "world"
docker compose logs -f                              # Ctrl+C to stop following
```

Test persistence:

```bash
docker compose down        # removes the container, keeps the volume
docker compose up -d
docker exec -it valkey valkey-cli get hello        # still "world"
```

Clean up completely, including the data:

```bash
docker compose down -v
```

`restart: unless-stopped` means the container starts again automatically after a reboot unless you stopped it yourself.

---

## 15. Everyday commands

| Task | Command |
| --- | --- |
| Connect from anywhere | `ssh nubecode@nubecode1` |
| Update the system | `sudo apt update && sudo apt upgrade -y` |
| Reboot / shut down | `sudo reboot` / `sudo poweroff` |
| Is a reboot needed? | `cat /var/run/reboot-required` |
| Disk usage | `df -h` |
| Docker disk usage | `docker system df` |
| Remove unused images and build cache | `docker system prune` |
| Running containers | `docker ps` |
| CPU, RAM, processes | `htop` |
| Temperatures | `sudo apt install lm-sensors` once, then `sensors` |
| IP addresses | `ip -brief address` |
| Boot time | `systemd-analyze` |
| Service logs | `journalctl -u ssh -n 50` |
| Firewall status | `sudo ufw status verbose` |
| Tailscale status | `tailscale status` |

---

## 16. Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| Power light on, no picture | Passive DP-to-HDMI adapter, wrong monitor input, or memory training | Plain DP-to-DP cable, set the input to DisplayPort, wait 2 to 3 minutes. See 2.1. |
| `Boot Device Not Found (3F0)` | No OS installed (or no drive) | Install an SSD and Ubuntu. See 2.3 and 5. |
| F9 menu shows only network boot entries | Blank SSD, USB not detected | Plug in the installer stick; enable USB storage boot in F10. |
| Installer crashes after entering Wi-Fi details | Installer Wi-Fi bug | Reboot and retry; or use Ethernet, USB tethering, or install without network. See 5.3. |
| Stuck on "Rebooting..." after install | Installer waiting for input | Press Enter; otherwise hold power for 10 seconds. See 5.12. |
| Boot takes 2+ minutes | `systemd-networkd-wait-online` and cloud-init waiting for an unplugged port | See section 7. |
| `ssh: connect ... timed out` at home | Laptop on a different network (mesh in router mode), wrong IP, server off | Same network as the server, check `ip -brief address`, or use Tailscale. See 8.4. |
| `Permission denied (publickey)` | Key not on the server, or wrong username | Log in at the server's console and add the key to `~/.ssh/authorized_keys`, or temporarily re-enable passwords. |
| `tailscale: command not found` after the install script | Another apt repository errored and aborted the script | `sudo apt install -y tailscale`. See 12.3. |
| Server gone from the tailnet after months | Tailscale key expired | Re-run `sudo tailscale up` on the server; disable key expiry. See 12.5. |
| `permission denied ... docker.sock` | User not in the `docker` group yet | `sudo usermod -aG docker $USER`, then log out and back in. |
| Reserved IP not applied | Old DHCP lease still active, or wrong MAC reserved | Reboot the server; compare the MAC with `ip link show wlo1`. |
| A container port is reachable despite ufw | Docker bypasses ufw | Bind to `127.0.0.1` or the Tailscale IP. See 13.7. |

---

## 17. What's next

- **Keep every `compose.yaml` in Git** so the whole server can be rebuilt from a repository.
- **Backups:** the unused eMMC (about 58 GB) can be formatted and mounted as a local backup target for volumes and configs. Keep an off-site copy too.
- **Deploy a real service:** a FastAPI app with PostgreSQL and Valkey in one Compose project, reachable over Tailscale.

---

## 18. References

- Ubuntu Server download: <https://ubuntu.com/download/server>
- Ubuntu Server basic installation tutorial: <https://ubuntu.com/server/docs/tutorial/basic-installation/>
- Install Docker Engine on Ubuntu: <https://docs.docker.com/engine/install/ubuntu/>
- Docker post-installation steps for Linux: <https://docs.docker.com/engine/install/linux-postinstall/>
- Docker packet filtering and firewalls: <https://docs.docker.com/engine/network/packet-filtering-firewalls/>
- Tailscale for Linux: <https://tailscale.com/download/linux>
- Valkey Docker image: <https://hub.docker.com/r/valkey/valkey>
- HP t740 hardware overview (ServeTheHome review): <https://www.servethehome.com/hp-t740-thin-client-review-tinyminimicro-with-pcie-slot-amd-ryzen/2/>

---

*Written October 2026 on Ubuntu Server 26.04.1 LTS (kernel 7.0). Commands and menus may change in later releases.*

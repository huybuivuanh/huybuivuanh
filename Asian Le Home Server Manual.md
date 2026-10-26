# Asian Le Home Server Manual

Oct 25, 2026 · @Vu Anh Huy Bui

## Overview

A self-hosted Google Drive replacement for the restaurant and family: Nextcloud at **asianle.cloud**, running on an old HP ProLiant, with a mirrored data drive, nightly encrypted backups and phone alerts. Built Oct 22 to 25, 2026. Passwords, tokens and keys are not stored here; they live in the password manager.

&#91;embedded content: how the server is reached · public and admin paths\]

Everyone reaches Nextcloud through Cloudflare's tunnel; only your own devices reach the admin tools, through Tailscale. No router ports are open.

### Hardware

| Part | Detail |
| --- | --- |
| Server | HP ProLiant (ML110 G6 era), Phoenix BIOS dated 2010, legacy BIOS only (no UEFI) |
| CPU | Intel Core i3-540, 3.07 GHz (no hardware AES, so encryption is slower) |
| RAM | 6 GB (5.2 GB usable), DDR3 |
| BIOS keys | F10 = ROM-Based Setup, F12 = network boot, Esc = boot summary |
| BIOS settings changed | Power on after AC power loss = Yes; boot order = HP 250 GB drive first |

### Drives

| Linux name | Drive | Size | Role | Mounted at |
| --- | --- | --- | --- | --- |
| sdc | HP VB0250EAVER (enterprise) | 250 GB | Ubuntu + Docker | `/` and `/boot` |
| sda | WD WD5000AAKX, serial WCAYUP096456 | 500 GB | Mirror member | `/dev/md0` → `/mnt/data` |
| sdb | WD WD5000AAKX, serial WCAYUP379412 | 500 GB | Mirror member | `/dev/md0` → `/mnt/data` |
| sdd | Toshiba MQ01ABD100 (2.5" laptop) | 1 TB | Nightly backups | `/mnt/backup` |

Drive letters can change between boots; check with `lsblk -o NAME,SIZE,MODEL,SERIAL` before any command that names a drive. At setup the two WDs had about 127,400 power-on hours (14.5 years), the HP about 49,400, the Toshiba about 6,700. All had zero bad sectors.

### Names and addresses

| Thing | Value |
| --- | --- |
| Hostname | asian-le-server |
| Linux user | asianlerestaurant |
| Public Nextcloud URL | https://asianle.cloud |
| Domain registrar and DNS | Cloudflare (bought Oct 2026, separate from the restaurant's Namecheap domain) |
| Cloudflare tunnel name | asian-le-server, route asianle.cloud → http://localhost:11000 |
| Local IP (home eero network) | 192.168.5.136 (eero range 192.168.4.0/22) |
| Network card | enp30s0, MAC 3c:d9:2b:7a:31:e8 |
| Tailscale name | asian-le-server (100.x address) |
| Cockpit dashboard | https://asian-le-server:9090 |
| Nextcloud AIO admin page | https://asian-le-server:8080 (open it from Nextcloud, see Phase 5) |
| Nextcloud data folder | /mnt/data/ncdata |
| Backup repository | /mnt/backup/borg |

### Home network note

The house has two Wi-Fi networks: the SaskTel modem (172.16.1.x) and an eero acting as its own router (192.168.4.0/22). The server is plugged into the eero. A laptop on the SaskTel Wi-Fi cannot reach 192.168.5.136; use Tailscale or join the eero Wi-Fi.

## Phase 1: Install Ubuntu Server

Ubuntu Server LTS (regular, not minimized), no desktop, installed on the HP 250 GB drive in legacy BIOS mode. The old Linux Mint install failed to boot with a blinking dash because GRUB was missing or on the wrong drive; a fresh install fixed it.

1. Download the latest Ubuntu Server LTS ISO from ubuntu.com/download/server.
2. Write it to a USB stick (4 GB+) with balenaEtcher, or Rufus with Partition scheme = MBR and Target = BIOS.
3. Boot the server, press **F10** at the HP logo, move USB to the top of the boot order, save.
4. Language and keyboard: English. Network: DHCP over ethernet. Skip proxy, accept the mirror.
5. Storage: "Use an entire disk" on the **VB0250EAVER** (250 GB). Leave LVM on, leave LUKS encryption **off** (it would need a passphrase typed at every boot).
6. On the summary, check `ubuntu-lv` uses the full size (about 231 GB). The installer sometimes defaults to about 100 GB; edit it to the maximum if so.
7. Leave the other drives untouched in the installer.
8. Profile: name, server name `asian-le-server`, user `asianlerestaurant`. Skip Ubuntu Pro.
9. Check **Install OpenSSH server**. Select **no** featured snaps, especially not Docker (the snap version causes problems).
10. Reboot, remove the USB. If it shows the blinking dash, press F10 and put the 250 GB drive first in the boot order.

Result: `/boot` 2 GB ext4, BIOS grub spacer 1 MB, and the rest as an LVM volume mounted at `/`.

## Phase 2: Base setup

Update the system, add the drive-health tools and the Cockpit web dashboard, then set the clock. Correct time matters for logins, two-factor codes, certificates and backup schedules.

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install smartmontools cockpit -y

sudo timedatectl set-timezone America/Regina
sudo timedatectl set-ntp true
sudo hwclock --systohc
timedatectl          # expect: System clock synchronized: yes
```

- **smartmontools** gives `smartctl` (read a drive's health) and `smartd` (background monitor, Phase 7).
- **Cockpit** is at https://asian-le-server:9090; log in with the Ubuntu user. The certificate warning is expected.
- If the BIOS clock keeps resetting after power loss, the motherboard CR2032 coin battery is dead (about $5).

## Phase 3: Storage

The two WD 500 GB drives form a RAID 1 mirror (`/dev/md0`, about 458 GB usable) for Nextcloud data; the Toshiba 1 TB holds backups. A mirror survives one dead drive with no downtime; the backup covers deletions, ransomware and both WDs dying.

### 3.1 Check drive health

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,FSTYPE,MOUNTPOINT
for d in sda sdb sdc sdd; do echo "=== /dev/$d ==="; sudo smartctl -i -A /dev/$d | grep -E "Device Model|Power_On_Hours|Reallocated_Sector|Current_Pending|Offline_Uncorrectable|UDMA_CRC"; done
```

Reallocated, Pending and Offline\_Uncorrectable should be 0. A high UDMA\_CRC count points to a bad cable, not the drive.

### 3.2 Build the mirror (erases both WDs)

```bash
# clear leftovers from the drives' previous use
sudo mdadm --stop /dev/md127
sudo mdadm --zero-superblock /dev/sdb1
sudo wipefs -a /dev/sda1 /dev/sdb1
sudo wipefs -a /dev/sda /dev/sdb

# create RAID 1; answer y to continue and y to the write-intent bitmap
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sda /dev/sdb

# format and mount
sudo mkfs.ext4 -L data /dev/md0
sudo mkdir -p /mnt/data
sudo tune2fs -m 1 /dev/md0      # reserve 1% instead of 5%, gains ~18 GB

# survive reboots
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
echo "UUID=$(sudo blkid -s UUID -o value /dev/md0) /mnt/data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a

df -h /mnt/data
cat /proc/mdstat                # [UU] = both drives active
```

- The **write-intent bitmap** means a resync after a power cut or a dropped drive takes seconds, not 80 minutes.
- **fstab uses the UUID**, so it mounts correctly even if drive letters change. **nofail** lets the server boot even if the drive is missing.
- The first full sync takes about 80 minutes in the background.

### 3.3 Backup drive (erases the Toshiba)

```bash
sudo wipefs -a /dev/sdd1
sudo wipefs -a /dev/sdd
sudo parted -s /dev/sdd mklabel gpt mkpart backup ext4 0% 100%
sudo partprobe /dev/sdd
sudo mkfs.ext4 -m 0 -L backup /dev/sdd1
sudo mkdir -p /mnt/backup
echo "UUID=$(sudo blkid -s UUID -o value /dev/sdd1) /mnt/backup ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
df -h /mnt/data /mnt/backup     # ~458G and ~916G
```

### Why these choices

- **Mirror, not combining the two 500s into 1 TB.** Combining doubles the failure risk and the 1 TB backup drive could not hold history for 930 GB of data.
- **Old drives are acceptable** because of the mirror plus the nightly backup. Replace a WD when one fails; for more space later, mirror two new NAS drives (WD Red Plus or Seagate IronWolf).

## Phase 4: Docker, domain and Cloudflare Tunnel

Docker runs each app in its own container. Cloudflare Tunnel publishes Nextcloud at asianle.cloud through an outbound connection, so no router ports are opened, the home IP stays hidden, and it keeps working if the server moves to the office.

### 4.1 Docker (official installer, not the snap)

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
exit                              # log out and back in for the group to apply
docker run --rm hello-world
```

### 4.2 Domain

- Bought **asianle.cloud** on Cloudflare Registrar, separate from the restaurant domain at Namecheap, so the restaurant's 5 websites and email are never touched.
- Cloudflare Registrar does not sell `.ca`; a `.ca` would be bought at Namecheap with nameservers pointed to Cloudflare.

### 4.3 Create the tunnel (Cloudflare dashboard)

1. **Networking → Tunnels → Create tunnel**, type Cloudflared, name `asian-le-server`.
2. On the install screen choose **Docker** and copy the token after `--token`. Keep it private.
3. Open the tunnel → **Routes → Add route → Published application**.
4. Hostname: subdomain **empty**, domain `asianle.cloud`. Service URL: `http://localhost:11000`. Add route. The DNS record is created automatically.

### 4.4 Run cloudflared on the server

Use this command, not the shorter one Cloudflare shows:

```bash
docker run -d --name cloudflared --restart unless-stopped --network host \
  cloudflare/cloudflared:latest tunnel --no-autoupdate run --token YOUR_TOKEN
```

- `-d`: runs in the background, survives closing SSH.
- `--restart unless-stopped`: comes back after reboots.
- `--network host`: lets it reach `localhost:11000` on the server. Without it the tunnel points at the container itself and Nextcloud is unreachable.

The tunnel shows **Healthy** in Cloudflare within a minute. If the token leaks, use **Rotate token** on the tunnel page and rerun the command.

### 4.5 Cloudflare settings for Nextcloud

- **Rocket Loader: off** (Speed → Optimization). It rewrites JavaScript and breaks Nextcloud pages.
- **Bot Fight Mode: off** (Security). It can block the sync apps.
- Free plan limit: **100 MB per upload request**. Chunked uploads work around it (Client apps section).

## Phase 5: Nextcloud AIO

Nextcloud All-in-One (AIO v14.4.1, Hub 26 Spring) runs in reverse proxy mode: Apache listens only inside the server on port 11000, and the Cloudflare tunnel is the only way in from outside.

### 5.1 Start the mastercontainer

```bash
sudo mkdir -p /mnt/data/ncdata
docker run -d \
  --init \
  --sig-proxy=false \
  --name nextcloud-aio-mastercontainer \
  --restart always \
  --publish 8080:8080 \
  --env APACHE_PORT=11000 \
  --env APACHE_IP_BINDING=127.0.0.1 \
  --env SKIP_DOMAIN_VALIDATION=true \
  --env NEXTCLOUD_DATADIR=/mnt/data/ncdata \
  --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  nextcloud/all-in-one:latest
```

| Setting | Why |
| --- | --- |
| `--publish 8080:8080` | AIO admin page, reachable on the LAN or Tailscale only |
| `APACHE_PORT=11000`, `APACHE_IP_BINDING=127.0.0.1` | Nextcloud only listens inside the server; the tunnel connects to it |
| `SKIP_DOMAIN_VALIDATION=true` | AIO's domain check cannot work through a tunnel |
| `NEXTCLOUD_DATADIR=/mnt/data/ncdata` | All user files go on the mirror. Set once at first install; cannot easily change later. |
| docker.sock volume | Lets the mastercontainer create and manage the other containers |

### 5.2 Setup in the AIO interface

1. Open https://SERVER-IP:8080, accept the certificate warning, and save the **AIO passphrase** in the password manager.
2. Domain: `asianle.cloud` (cannot be changed later).
3. Leave "Install Nextcloud Hub 26 Summer" unchecked (Spring is the more established release).
4. Office suite: **Nextcloud Office powered by Euro-Office** (default; best Microsoft-format compatibility). Collabora is the alternative and can be swapped later.
5. Optional containers: **Imaginary on** (iPhone HEIC previews). Off: ClamAV, Fulltextsearch, Talk, Talk Recording, Docker Socket Proxy, HaRP, Whiteboard. Talk needs ports the tunnel cannot pass; the others cost 1 to 2 GB of RAM.
6. Click **Save changes**, type `America/Regina` in the timezone field and **Submit timezone**.
7. Click **Download and start containers**. First start takes 5 to 15 minutes.
8. Save the **initial admin password** shown under the container list.

Running containers: apache, nextcloud, database (PostgreSQL), redis, notify-push, eurooffice, imaginary, mastercontainer, plus cloudflared.

### 5.3 Opening the AIO page later

While Nextcloud runs, direct login to port 8080 is blocked. Use **Nextcloud → Administration settings → Nextcloud All-in-One → Open Nextcloud AIO Interface**. The link points at the server's local IP, so the laptop must be on the eero network, or replace the IP in the address bar with `asian-le-server` while on Tailscale. The link is single use.

The AIO page also suggests `sudo docker stop nextcloud-aio-apache` to unblock login. That takes asianle.cloud offline (502 error) until **Start containers** is clicked.

### 5.4 Nextcloud settings

- Personal account created and added to the **admin** group; the `admin` account is kept for emergencies.
- New accounts start empty, without sample files:

```bash
docker exec --user www-data nextcloud-aio-nextcloud php occ config:system:set skeletondirectory --value=""
```

- File locks ("Temporary files lock" app) kept on: they stop two people editing one file in different places. Unlock via the file's ⋯ menu, or restart Redis if locks get stuck.
- Sharing: with an account (they log in) or by link (no login). Link options: View only, Allow upload and editing, File request (upload only), Custom (Read, Create, Edit, Delete). Create = upload into a folder link. Download is on unless Hide download is checked. Use a password and expiry on edit links.
- Guests editing through a link get Euro-Office in the browser and type a display name.

## Phase 6: Backups

AIO's built-in BorgBackup writes an encrypted, deduplicated backup of files, database and settings to `/mnt/backup` every night at 4 AM Saskatchewan time. Restore was tested and works.

### Setup (AIO page → Backup and restore)

1. Local backup location: `/mnt/backup` (no trailing slash). Leave **Remote borg repo** empty.
2. Save the **Borg backup password** AIO shows in the password manager. Without it the backups cannot be restored. Never keep it only on the server.
3. Click **Create backup** for the first run.
4. Daily backup time: **10:00**. The field is in UTC; 10:00 UTC = 4:00 AM in Regina (no daylight saving).
5. Automatic updates: **on**. AIO backs up first, then updates the containers, the mastercontainer and (Saturdays) Nextcloud apps.

### What to expect

- Nextcloud is **offline during the backup** so the copy is consistent. Apps pause and retry; the website shows a 502.
- The first backup after a lot of new data is slow: about 100 GB took over 45 minutes (old CPU without AES acceleration, 2.5" target drive). Later nights only copy changes and take a few minutes.
- "Waiting for backup container to stop" every 30 seconds in the mastercontainer log is normal while Borg works. Check progress with `watch -n 30 df -h /mnt/backup`; if Used keeps growing, it is working.
- Only if Used stops growing for 15+ minutes and the page still says running: `sudo docker exec nextcloud-aio-mastercontainer rm /mnt/docker-aio-config/data/daily_backup_running`, then reload the page.
- Old versions are pruned automatically; restore from any kept point on the AIO page.

### Restore test (done once, repeat after big changes)

1. Put a test file in Nextcloud, click **Create backup**.
2. Delete the file and empty Deleted files.
3. On the AIO page, pick that backup and **Restore**.
4. Confirm the file is back. A restore returns everything to that moment.

### The 3 layers

| Layer | Protects against | Where |
| --- | --- | --- |
| Mirror | One drive dying | WD + WD, `/mnt/data` |
| Nightly Borg backup | Deletion, bad updates, ransomware, both WDs dying | Toshiba, `/mnt/backup` |
| Offsite copy (not set up yet) | Fire, theft, power surge killing the whole box | Planned: rclone `/mnt/backup` to Backblaze B2 |

## Phase 7: Phone alerts

The server pushes alerts to the **ntfy** phone app through one script, `/usr/local/bin/notify`. Both the mirror monitor (mdadm) and the drive monitor (smartd) call it. Email is not set up; there is no mail system on the server.

### 7.1 Phone

Install **ntfy**, tap +, subscribe to a hard-to-guess topic (stored in the password manager; anyone who knows the name can read the alerts). On iPhone allow Lock Screen, Banners and Sounds for ntfy. On mobile data, alerts can arrive about 30 seconds late; that is normal.

### 7.2 The notify script

```bash
sudo tee /usr/local/bin/notify > /dev/null << 'EOF'
#!/bin/sh
case "$1" in Rebuild[0-9]*) exit 0;; esac
MSG="${SMARTD_MESSAGE:-$*}"
curl -s -H "Title: asian-le-server" -H "Priority: high" -d "$MSG" https://ntfy.sh/YOUR_TOPIC > /dev/null
EOF
sudo chmod +x /usr/local/bin/notify
notify "Test from the server"
```

- The `case` line skips mdadm's progress messages (Rebuild20, 40, 60, 80) but keeps Started, Finished and failures.
- Changing topics later: `sudo sed -i 's|ntfy.sh/OLD_TOPIC|ntfy.sh/NEW_TOPIC|' /usr/local/bin/notify`, and subscribe to the new topic in the app.

### 7.3 Mirror alerts (mdadm)

```bash
echo "PROGRAM /usr/local/bin/notify" | sudo tee -a /etc/mdadm/mdadm.conf
sudo sed -i 's/^MAILADDR/#MAILADDR/' /etc/mdadm/mdadm.conf   # stops 'sendmail: not found'
sudo systemctl restart mdmonitor
sudo mdadm --monitor --scan --test --oneshot                  # sends TestMessage /dev/md0
```

| Alert | Meaning | Action |
| --- | --- | --- |
| Fail, DegradedArray | A mirror drive dropped out | Check `cat /proc/mdstat`; plan a replacement |
| RebuildStarted / RebuildFinished | Routine monthly check, or a resync | None if mdstat shows \[UU\] |
| SpareActive | A drive rejoined the mirror | None |

### 7.4 Drive health alerts (smartd)

```bash
sudo cp /etc/smartd.conf /etc/smartd.conf.bak
echo 'DEVICESCAN -a -o on -S on -n standby,q -s (S/../.././02|L/../../7/01) -W 4,45,50 -m <nomailer> -M exec /usr/local/bin/notify' | sudo tee /etc/smartd.conf
sudo systemctl restart smartmontools
```

- `-a`: watch all health attributes. `-s`: short self-test daily at 2 AM, long test Sundays at 1 AM. `-W 4,45,50`: alert on a 4°C jump, note at 45°C, warn at 50°C.
- To test, add `  -M test ` at the end and restart: one alert per drive. Then remove it, or every boot sends 4 test alerts.

## Phase 8: Tailscale (private admin access)

Tailscale puts the server, laptop and phone on a private encrypted network (WireGuard), so SSH, Cockpit and the AIO page work from anywhere without exposing them to the internet. Cloudflare is the public door for Nextcloud; Tailscale is the staff-only door for admin.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up            # open the printed link and sign in
tailscale ip -4              # the server's 100.x address
```

1. Install the Tailscale app on the laptop and phone, signed in with the **same account**.
2. In login.tailscale.com → Machines → asian-le-server → **Disable key expiry**, so the server never silently drops off after 180 days.
3. Keep two-factor on the account used to sign in to Tailscale; it is the key to the private network.

| Task | From anywhere, with Tailscale on |
| --- | --- |
| SSH | `ssh asianlerestaurant@asian-le-server` |
| Cockpit | https://asian-le-server:9090 |
| AIO page | https://asian-le-server:8080 |

- If the name does not resolve, use the 100.x IP.
- On Windows, a Tailscale adapter showing 169.254.x.x means Tailscale is not connected; click the tray icon → Connect.
- Tailscale only carries traffic between your own devices; normal browsing is unaffected. Free Personal plan: 3 users, 100 devices.

## Phase 9: Freeze protection

The server froze silently around 1:20 AM on Oct 23 with nothing in the logs, which points to hardware (RAM, power supply or board). These settings make it reboot itself and notify the phone if that happens again. Confirm they are in place with `sudo wdctl` (should show a 30-second timeout) and `cat /etc/sysctl.d/99-reboot-on-freeze.conf`.

```bash
# hardware watchdog: the board reboots the server if Linux stops responding for 30 s
sudo modprobe iTCO_wdt
echo iTCO_wdt | sudo tee /etc/modules-load.d/watchdog.conf
sudo sed -i 's/^#\?RuntimeWatchdogSec=.*/RuntimeWatchdogSec=30s/' /etc/systemd/system.conf
sudo systemctl daemon-reexec
sudo wdctl

# reboot 10 s after a kernel panic or soft lockup instead of hanging on an error screen
printf "kernel.panic = 10\nkernel.softlockup_panic = 1\n" | sudo tee /etc/sysctl.d/99-reboot-on-freeze.conf
sudo sysctl --system

# phone alert on every boot
(sudo crontab -l 2>/dev/null; echo '@reboot sleep 60 && /usr/local/bin/notify "asian-le-server just booted"') | sudo crontab -
```

- An unexpected "just booted" alert = it froze or lost power. Check the previous boot's last log lines with `journalctl -b -1 -n 60 --no-pager`.
- Next diagnostic step: a RAM test. Reboot holding **Shift**, choose **Memory test (memtest86+)**, let it finish one full pass (1 to 2 hours). Red errors = a bad DDR3 stick.

## Client apps

Every client logs in with server address `https://asianle.cloud` and a personal account (not `admin`). Desktop clients need the chunk-size fix so large files stay under Cloudflare's 100 MB per-request limit; an 811 MB file uploaded fine after it.

### Windows

1. Install: nextcloud.com/install → Nextcloud Files → Windows, or `winget install Nextcloud.NextcloudDesktop`.
2. Log in, Grant access in the browser. Sync folder: `C:\Users\Bui Vu Anh Huy\Nextcloud`. Virtual files: **on** (files download when opened; right-click → Always keep on this device for offline).
3. Chunk fix: quit Nextcloud, press Win+R, run `notepad %APPDATA%\Nextcloud\nextcloud.cfg`, add `maxChunkSize=50000000` under `[General]`, save, restart Nextcloud.
4. Do not delete the hidden `.sync_*.db`, `.sync_*.db-shm`, `.sync_*.db-wal` or `.nextcloudsync.log` files in the sync folder; they are the client's tracking database. The sample files (Nextcloud Manual.pdf, intro.mp4 and so on) are safe to delete.

### Mac

1. Install Nextcloud Desktop for macOS and log in.
2. If it says **"No synchronization configured"**, open the account settings → **Add Folder Sync Connection** → local `~/Nextcloud`, remote `/` → Add.
3. Find it: menu-bar cloud icon → Open local folder, or Cmd+Shift+H in Finder. Drag it into Finder Favorites. With virtual files it shows under Locations; enable it in System Settings → General → Login Items & Extensions → File Providers.
4. Chunk fix: quit Nextcloud, run `open -e ~/Library/Preferences/Nextcloud/nextcloud.cfg`, add `maxChunkSize=50000000` under `[General]`, save, restart.

### Phone

Install the **Nextcloud** app, log in, optionally turn on Auto upload for the camera folder. Files are shown on demand; mark folders "available offline" if needed.

### Upload size limits

| How | Limit |
| --- | --- |
| Logged in (desktop, phone, browser) | Chunked, so effectively free space or quota; 5 to 20 GB files work |
| Guest upload link | Untested; may stop at 100 MB if not chunked. Test with a 200 MB+ file in a private window. |
| Single non-chunked upload | 100 MB (Cloudflare free plan) |
| Downloads | No per-request limit; heavy repeated video sharing could break Cloudflare's free-plan terms |

## Command cheat sheet

### Health check (run these first when anything looks wrong)

```bash
uptime                                              # how long since the last boot
cat /proc/mdstat                                    # [UU] = mirror healthy, [U_] = a drive is out
df -h /mnt/data /mnt/backup                         # free space
free -h                                             # RAM and swap
docker ps --format "table {{.Names}}\t{{.Status}}"  # every container should be Up / healthy
curl -I http://localhost:11000                      # 200 or 302 = Nextcloud answering locally
docker logs --tail 30 cloudflared                   # tunnel problems
```

### Containers

```bash
docker start nextcloud-aio-apache      # if asianle.cloud gives 502 and apache is missing from docker ps
docker restart nextcloud-aio-redis     # clears stuck file locks
docker logs --tail 50 nextcloud-aio-nextcloud
docker logs --tail 50 nextcloud-aio-mastercontainer
```

Prefer **Start containers** on the AIO page; it starts everything in the right order.

### Nextcloud admin (occ)

```bash
OCC="docker exec --user www-data -it nextcloud-aio-nextcloud php occ"
$OCC user:list
$OCC user:resetpassword USERNAME
$OCC twofactorauth:disable USERNAME totp
$OCC security:bruteforce:reset 127.0.0.1
```

### Drives and mirror

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL
sudo smartctl -H /dev/sdX                         # quick PASSED/FAILED
sudo smartctl -a /dev/sdX                         # full detail
sudo mdadm --detail /dev/md0
cat /sys/block/md0/md/mismatch_cnt                # 0 after a check = drives identical
```

Replacing a dead mirror drive: power off, swap the drive, boot, check its name with `lsblk`, then `sudo mdadm /dev/md0 --add /dev/sdX` and watch `cat /proc/mdstat` until \[UU\]. Mirror failure drill (tested Oct 22): `--fail /dev/sdb`, then `--remove` and `--re-add`. Never pull a cable while powered on.

### Logs and freezes

```bash
journalctl -b -1 -n 60 --no-pager                 # last lines before the previous boot ended
journalctl -b -1 -k --no-pager | grep -iE "oom|out of memory|hung|panic|mce|hardware error|thermal"
journalctl -k --since "1 hour ago" | grep -iE "md|ata|sd[ab]"   # disk or cable errors
```

### Shutting down and moving

```bash
sudo shutdown now      # wait for lights and fans to stop before unplugging
```

Carry gently and upright. After a cold car ride, wait 1 to 2 hours indoors before powering on. At the new location the local IP changes; Cloudflare and Tailscale keep working without changes. Set a DHCP reservation on that router.

## Troubleshooting log

Every problem hit during setup, newest first, with the cause and fix.

| Date | Symptom | Cause | Fix |
| --- | --- | --- | --- |
| Oct 25 | Mac client: "No synchronization configured" | Folder step skipped during setup | Add Folder Sync Connection (Client apps → Mac) |
| Oct 24 | Nextcloud offline 4:01 AM onward, AIO says "Daily backup currently running" for 45+ min | First backup after \~100 GB of new data; slow CPU encryption | Not stuck; let it finish, watch `df -h /mnt/backup` grow |
| Oct 23 | Notifications RebuildStarted, Rebuild20 to 80, RebuildFinished | Ubuntu's routine mirror check, about 1.5 hours | None; added the `case` filter to skip progress alerts |
| Oct 23 | SSH to 192.168.5.136 times out, Nextcloud works | Laptop on SaskTel Wi-Fi (172.16.1.x), server on eero (192.168.4.0/22); Tailscale not connected on laptop | Join eero Wi-Fi or connect Tailscale and use `asian-le-server` |
| Oct 23 | Tunnel Down, 0 replicas; server unreachable | Silent freeze at \~1:20 AM, no log entries, plenty of free RAM; likely hardware | Hard reset; added watchdog, panic reboot, boot alert (Phase 9); RAM test pending |
| Oct 23 | 4 ntfy alerts at every boot | `-M test` still in smartd.conf | `sudo sed -i 's/ -M test//' /etc/smartd.conf`, restart smartmontools |
| Oct 23 | Files appear locked | Temporary files lock app (Office edits, desktop client locks) | Expected; ⋯ → Unlock file, or `docker restart nextcloud-aio-redis` for stuck locks |
| Oct 22 | AIO link from Nextcloud: "refused to connect" | Link points to the local IP; laptop was on another network | Join eero Wi-Fi or swap the IP for `asian-le-server` on Tailscale |
| Oct 22 | asianle.cloud 502 Bad Gateway, Cloudflare shows Host error | `nextcloud-aio-apache` stopped (the AIO login page tells you to stop it) | `docker start nextcloud-aio-apache` or Start containers on the AIO page; open AIO from inside Nextcloud instead |
| Oct 22 | `sendmail: not found` on mdadm test | Default `MAILADDR root` line, no mail system | Commented out MAILADDR |
| Oct 22 | ntfy test arrived without sound | Phone notification settings | iPhone: Settings → Notifications → ntfy, allow Sounds and Banners |
| Oct 22 | No ntfy alerts after changing topic | Script still posted to the old topic | sed the topic in `/usr/local/bin/notify` |
| Oct 22 | Cloudflare "Public hostname" screen missing | Cloudflare renamed it | Tunnel → Routes → Add route → Published application |
| Oct 22 | Old Linux Mint stuck at a blinking dash | BIOS booting a drive without GRUB, or GRUB installed in UEFI mode on a legacy-only BIOS | Fresh Ubuntu Server install; HP 250 GB first in boot order |

## Open items

- [ ] Run the RAM test (memtest86+, one full pass) to explain the Oct 23 freeze
- [ ] Confirm the watchdog and panic-reboot settings are active (`sudo wdctl`)
- [ ] Family accounts, a `family` group, and a shared **Family** folder via the Team folders app, with quotas (50 to 100 GB each)
- [ ] Two-factor login (Two-Factor TOTP Provider app) on admin and personal accounts; save backup codes
- [ ] DHCP reservation for 192.168.5.136 in the eero app (Settings → Network settings → Reservations & port forwarding)
- [ ] Decide on one home Wi-Fi: turn off the SaskTel modem's Wi-Fi so everything uses the eero
- [ ] Test a guest upload link with a 200 MB+ file in a private window
- [ ] Offsite backup: rclone `/mnt/backup` to Backblaze B2 nightly (Borg data is already encrypted)
- [ ] Before moving to the office: check the office's upload speed and whether its firewall allows outbound tunnels
- [ ] Optional: a UPS (APC 600 to 850 VA) with `nut` for clean shutdowns; more used DDR3 RAM if Immich is added
- [ ] Optional: replace the CMOS coin battery if the BIOS clock or settings reset

## Asking Claude for help

Paste the block below at the start of a new chat, then describe the problem and attach the health-check output from the cheat sheet. For a full rebuild, share this whole doc instead. Never paste the tunnel token, passwords, the Borg password or the ntfy topic.

```markdown
My home server, for context:
- HP ProLiant (2010, legacy BIOS), i3-540, 6 GB RAM, Ubuntu Server LTS, host asian-le-server, user asianlerestaurant
- Drives: HP 250 GB = OS; 2x WD 500 GB (14+ years old) = mdadm RAID 1 /dev/md0 at /mnt/data (write-intent bitmap); Toshiba 1 TB = /mnt/backup
- Nextcloud AIO (reverse proxy mode, APACHE_PORT=11000 bound to 127.0.0.1, data dir /mnt/data/ncdata), Euro-Office + Imaginary
- Public access: Cloudflare Tunnel (cloudflared container, --network host) asianle.cloud -> http://localhost:11000; free plan, 100 MB request limit, desktop clients use maxChunkSize=50000000
- Admin access: Tailscale (asian-le-server), Cockpit :9090, AIO page :8080 (open via Nextcloud admin settings)
- Backups: AIO BorgBackup to /mnt/backup daily at 10:00 UTC (4 AM local) with auto updates; no offsite copy yet
- Alerts: /usr/local/bin/notify posts to ntfy; used by mdadm PROGRAM and smartd -M exec
- Freeze protection: iTCO_wdt watchdog 30 s, kernel.panic=10, @reboot ntfy alert
- Network: server on eero router (192.168.4.0/22, IP 192.168.5.136) behind a SaskTel modem
- History: one silent hardware-like freeze on Oct 23, 2026; RAM test pending

Problem: <describe it>
Output: <paste uptime, cat /proc/mdstat, docker ps, and any error>
```

Update this doc when something changes: new drives, a new domain or tunnel, a move to the office, or a new fix worth remembering.

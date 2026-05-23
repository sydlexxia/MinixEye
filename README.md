# minix_z83_postinstall.sh

Post-install configuration script for **Linux Mint** running on a
**Minix Z83-4** mini PC, converting a stock desktop install into a
headless motion-surveillance node / IPCam.

The goal is to minimise eMMC write amplification (no local video
storage), forward all logs off-box, and end with a lean, headless
system running motionEye.

---

## Target hardware

| Component    | Value                                              |
|--------------|----------------------------------------------------|
| Device       | Minix Neo Z83-4                                    |
| CPU          | Intel Atom x5-Z8350                                |
| Storage      | 32 GB internal eMMC                                |
| WiFi         | Broadcom BCM43455                                  |
| OS           | Linux Mint (Debian/Ubuntu base)                    |
| Role         | Headless IPC running MotionEye as a front-end      |

---

## Prerequisites
-----------------------------------------------------------------------

:warning:  ***IMPORTANT!*** - A BIOS update is required before installing 
Linux on the Neo Z83-4. Boot into BIOS (press F11 at startup) and note 
your current BIOS revision and build version — you'll need these to 
select the correct update file. Download your update here:

   https://theminixforum.com/index.php?forums/neo-z83-4-series.124/

-----------------------------------------------------------------------


The script will **refuse to run** unless these are already done:

1. `sudo apt update` has been run within the last 24 hours.
2. `sudo apt upgrade` has been run and zero packages are pending.
3. The machine has internet connectivity (verified against
   `1.1.1.1` / `8.8.8.8`).
4. `findmnt` (from `util-linux`) is available.

A full apt upgrade is intentionally **not** performed by this script —
package upgrades should be reviewed by the operator. Run them
manually first, review the output, then run this script.

> **Strongly recommended:** install and enable `openssh-server`
> before running this script. Section 10 (headless mode) refuses to
> disable the desktop unless SSH is reachable on port 22.

---

## Usage

```sh
sudo ./minix_z83_postinstall.sh [OPTIONS]
```

### Options

| Flag            | Behaviour                                        |
|-----------------|--------------------------------------------------|
| `-h`, `--help`  | Show help and exit                               |
| `-d`, `--debug` | Print every command before it executes           |
| `-y`, `--yes`   | Assume "yes" for all per-section confirmations   |
| `--force`       | Skip the up-front change summary acknowledgement |

`--yes` alone still requires a one-time `yes` at the summary banner.
`--yes --force` together run the script unattended.

### A run is always logged

Every run is `tee`'d to:

```
/root/minix-postinstall-YYYYMMDD-HHMMSS.log
```

The path is printed at the start and end of every run. Keep these
logs — Section 3 turns journald volatile, so the tee log may be your
only post-mortem record.

---

## What it does (in execution order)

| #  | Section           | Purpose                                                                          | Reboot? |
|----|-------------------|----------------------------------------------------------------------------------|---------|
| 1  | Packages          | apt-get install of build deps + utilities                                        | no      |
| 2  | Additional tweaks | Empty placeholder; edit in-script for hostname, sysctl, etc.                     | varies  |
| 3  | eMMC preservation | Swap off, noatime, tmpfs, volatile journal, logrotate, write-back tuning         | partial |
| 4  | Audio             | PipeWire buffer override                                                         | no      |
| 5  | WiFi              | BCM43455 firmware file                                                           | no      |
| 6  | CPU governor      | `performance` governor, persisted                                                | yes     |
| 7  | GRUB              | `intel_pstate=active` kernel param                                               | yes     |
| 8  | Logging           | rsyslog to remote host and/or USB; optionally disable local logging              | no      | 
| 9  | motionEye         | pip bootstrap → `pip install motioneye` → `motioneye_init` → verify port 8765    | no      |
| 10 | Headless          | Disable LightDM and desktop services; set `multi-user.target`                    | yes     |

Section 10 deliberately runs **last** so any earlier failure leaves
you with a recoverable GUI.

---

## Configuration

### Package list (Section 1)

Edit the `PACKAGES` array at the top of the script. Defaults pull in
everything motionEye needs to build:

```
ca-certificates curl python3 python3-dev gcc
libjpeg62-dev libcurl4-openssl-dev libssl-dev
locate bpytop rsyslog
```

### Additional tweaks (Section 2)

Open the script and edit the marked block in `additional_tweaks()`.
Suggested uses: hostname, sysctl rules, `ufw` setup, timezone.

### Remote / USB logging (Section 8)

The logging section is interactive by default, but every value can be
pre-set via environment variable so a `--yes` run is fully unattended:

| Variable            | Default        | Notes                                                                 | 
|---------------------|----------------|-----------------------------------------------------------------------|
| `LOG_REMOTE_HOST`   | *(none)*       | IP/host of the remote rsyslog receiver. Empty = no remote forwarding. |
| `LOG_REMOTE_PORT`   | `514`          | UDP/TCP port                                                          |
| `LOG_REMOTE_PROTO`  | `tcp`          | `tcp` (reliable) or `udp`                                             |
| `LOG_USB_DEVICE`    | *(none)*       | Block device path, e.g. `/dev/sdb1`. Empty = no USB logging.          |
| `LOG_USB_MOUNT`     | `/mnt/usb-log` | Mount point for the USB stick                                         |
| `LOG_DISABLE_LOCAL` | *(prompted)*   | Set to `true` to silence local logging without prompting              |

Example unattended run:

```sh
sudo \
  LOG_REMOTE_HOST=192.168.1.50 \
  LOG_REMOTE_PROTO=tcp \
  LOG_DISABLE_LOCAL=true \
  ./minix_z83_postinstall.sh --yes --force
```

---

## Safety features

The script is built to fail loudly and rollback rather than leave a
half-configured (or unbootable) system.

- **Pre-flight hard stops** — exits before any change if apt is
  stale, upgrades are pending, internet is down, or `findmnt` is
  absent.
- **Single up-front summary** — every destructive change is listed
  in one banner; you must type the full word `yes` to proceed.
- **fstab validation** — every `/etc/fstab` edit is wrapped in a
  snapshot + `findmnt --verify` check. On failure the snapshot is
  restored automatically and the script aborts.
- **GRUB self-healing** — if `update-grub` fails after the patch,
  `/etc/default/grub.bak` is restored and `update-grub` is re-run.
  A second failure aborts with a `DO NOT REBOOT` warning.
- **SSH gate** — `disable_desktop` refuses to run unless `sshd` is
  active and listening on `:22`. Can be manually overridden.
- **Order of operations** — motionEye is installed and verified
  while the GUI is still up; only then is the desktop disabled.
- **Self-logging** — full transcript saved to `/root/minix-postinstall-*.log`.
- **`ERR` trap** — on any failure, the failing line number, command,
  and transcript path are printed before exit.

---

## After the script runs

### Verify motionEye

The script polls `http://<local-ip>:8765/` up to 10 times before
finishing. To re-check manually:

```sh
curl -I http://<local-ip>:8765/
systemctl status motioneye
journalctl -u motioneye -n 50
```

> Once Section 3 has run, `journalctl` reads from a volatile journal
> that does not survive reboot. For persistent diagnostics, rely on
> remote rsyslog and/or the `/root/minix-postinstall-*.log` file.

### Confirm remote logging is reaching its destination

```sh
logger "minix postinstall test message"
```

The message should appear on the configured rsyslog receiver or in
`<usb_mount>/syslog.log`.

### Reboot

A reboot is required for sections **3b/3c**, **6**, **7**, and
**10**. Run:

```sh
sudo reboot
```

After reboot the box should come up to a text console (Ctrl-Alt-F1
on the attached display), with motionEye listening on `:8765`.

---

## Re-running and idempotency

The script is designed to be safely re-runnable. Each section
checks current state before applying changes:

- Already-set fstab options are skipped.
- Existing config files trigger an overwrite prompt.
- Already-disabled services are skipped silently.
- Already-installed pip is detected and the bootstrap is skipped.

You can re-run after a partial failure without redoing completed
work — but always read the transcript first to confirm what state
the box is in.

---

## Recovery / rollback hints

If something goes wrong, these are the relevant artefacts:

| File                                            | What it is                                                          |
|-------------------------------------------------|---------------------------------------------------------------------|
| `/etc/fstab.bak`                                | Pre-edit fstab backup (Section 3, 8)                                |
| `/etc/default/grub.bak`                         | Pre-edit grub backup (Section 7)                                    |
| `/etc/rsyslog.d/50-default.conf.disabled`       | Renamed local-file logging rules; rename back to `.conf` to restore |
| `/root/minix-postinstall-*.log`                 | Full run transcript                                                 |

To restore the desktop manually after Section 10:

```sh
sudo systemctl set-default graphical.target
sudo systemctl enable --now lightdm
sudo reboot
```

The decision to disable swap is to preserve the integrity of the eMMCblk device.
This can of course be enabled if desired (not recomended).  
To restore swap:

```sh
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## Files written or modified

| Path                                          | Section | Purpose                             |
|-----------------------------------------------|---------|-------------------------------------|
| `/etc/fstab`                                  | 3, 8    | Mount-options, tmpfs, USB log mount |
| `/etc/sysctl.d/99-swappiness.conf`            | 3a      | `vm.swappiness=0`                   |
| `/etc/sysctl.d/99-no-coredumps.conf`          | 3f      | `kernel.core_pattern=/dev/null`     |
| `/etc/sysctl.d/99-emmc-writeback.conf`        | 3g      | Dirty-page write-back tuning        |
| `/etc/systemd/journald.conf.d/99-emmc.conf`   | 3d, 8   | Volatile journal, forward-to-syslog |
| `/etc/systemd/coredump.conf.d/99-disable.conf`| 3f      | Disable systemd-coredump            |
| `/etc/logrotate.d/99-emmc`                    | 3e      | Aggressive log rotation             |
| `/etc/pipewire/buffer.conf`                   | 4       | Audio buffer overrides              |
| `/lib/firmware/brcm/brcmfmac43455-sdio.txt`   | 5       | BCM43455 firmware                   |
| `/etc/default/cpufrequtils`                   | 6       | `GOVERNOR="performance"`            |
| `/etc/default/grub`                           | 7       | `intel_pstate=active`               |
| `/etc/rsyslog.d/10-remote-forward.conf`       | 8       | Remote syslog forwarding            |
| `/etc/rsyslog.d/20-usb-log.conf`              | 8       | USB log destination                 |
| `/etc/pip.conf`                               | 9       | `break-system-packages=true`        |
| `/root/minix-postinstall-*.log`               | —       | Run transcript                      |

---

## Known limitations

- Targets Linux Mint specifically; should work on Debian/Ubuntu but
  is untested elsewhere.
- `findmnt --verify` catches structural fstab errors but cannot
  prove a mount will actually succeed — physical media issues
  surface only at mount time. Section 8b uses `nofail` on the USB
  mount to keep boot resilient.
- `motioneye_init` provisions a systemd unit; if the install method
  ever changes upstream, Section 9 may need an update.
- No support yet for `--dry-run` — every section either applies its
  change or skips. Use a fresh install / snapshot when testing.

---

## See also

- Main script: [`minix_z83_postinstall.sh`](minix_z83_postinstall.sh)
- Project README: [`README.MD`](README.MD)

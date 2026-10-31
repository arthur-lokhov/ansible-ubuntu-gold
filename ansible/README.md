# Ubuntu 26.04 LTS — expert install as an Ansible project

This directory turns [`docs/expert-install.md`](../docs/expert-install.md) into a
repeatable Ansible build.  It is **not** a wrapper around the Ubuntu installer:
a LiveCD boots, `debootstrap` creates the target, LUKS2/LVM/XFS are assembled,
and every configuration file from the document is written explicitly.

> **Destructive operation.**  The install playbook can wipe the target disk.
> It will only repartition when you pass `-e disk_confirm_wipe=true`; without
> that flag it reuses an existing three-partition layout.  Read the whole file
> before the first run.

## What is here

```
ansible/
├── ansible.cfg
├── inventory/
│   ├── hosts.yml                    # livecd + workstation, both connection=local
│   └── group_vars/all/
│       ├── main.yml                 # all non-secret settings
│       ├── packages.yml             # base package manifest
│       └── secrets.yml.example      # copy -> secrets.yml -> ansible-vault
├── playbooks/
│   ├── install.yml                  # stage 1, runs on the LiveCD
│   └── post-install.yml             # stage 2, runs on the installed system
├── roles/
│   ├── preflight/                   # stage 0: inventory, UEFI/disk/secrets checks
│   ├── disk/                        # stage 1: GPT, LUKS2, LVM, XFS, mounts
│   ├── debootstrap/                 # stage 2: minimal Ubuntu
│   ├── chroot_mounts/               # stage 2: /dev /proc /sys /run + DNS
│   ├── apt_repo/                    # stage 3.1-3.3: DEB822, TLS, Snap pin, policy
│   ├── packages/                    # stage 3.4: one package transaction
│   ├── base_config/                 # stage 4: environment, network, fstab, user
│   ├── boot/                        # stage 5: GRUB, MOK, Plymouth
│   ├── finalize/                    # stage 6: cleanup, unmount, next steps
│   ├── nvidia/                      # stage 7.1
│   ├── libvirt/                     # stage 7.2
│   ├── containers/                  # stage 7.3
│   └── restic/                      # optional backup role
├── docs/MAPPING.md                  # document section -> role/task map
└── artifacts/                       # hardware snapshots fetched from the LiveCD
```

No Galaxy collections or external roles are required.  Everything uses
`ansible-core` and built-in modules.

## Requirements

* Ubuntu 26.04 LTS LiveUSB, booted in **UEFI** mode (Secure Boot may stay on).
* A target disk of at least `disk_min_size_gb` (default 560 GiB) for the default
  LVM layout.
* `ansible-core` in the Live environment.  The LiveCD does not ship it:

  ```bash
  sudo apt update
  sudo apt install -y ansible-core python3-apt git
  ```

  A virtualenv with `pip install ansible-core` works too; the inventory uses
  `ansible_connection: local`, so no SSH server is required.
* The project copied to the Live environment (USB stick, `scp`, or a git clone).

The hardware profile from the document (i5-11400, AX210, RTX 3070) is encoded in
the defaults.  On different hardware change `target_disk`, the firmware package
list, and `debootstrap_arch` if the CPU does not support x86-64-v3.

## Quick start

### 1. Boot the LiveCD and put the project there

```bash
# optional but convenient: SSH into the Live environment
sudo apt update && sudo apt install -y openssh-server
sudo passwd ubuntu
sudo systemctl start ssh
ip -br addr
```

Then, on the build machine:

```bash
cd ansible
```

### 2. Create the secrets file

```bash
cp inventory/group_vars/all/secrets.yml.example \
   inventory/group_vars/all/secrets.yml
$EDITOR inventory/group_vars/all/secrets.yml
ansible-vault encrypt inventory/group_vars/all/secrets.yml
```

The file contains:

| Key | Used for |
| --- | --- |
| `vault_luks_passphrase` | LUKS2 container.  Minimum 8 characters. |
| `vault_user_password` | Password for `workstation_user`. |
| `vault_wifi_psk` | Wi-Fi PSK written to `wpa_supplicant`. |
| `vault_mok_password` | One-time password for MOK Manager. |
| `vault_restic_password` | Only when `restic_enabled=true`. |

The Wi-Fi **SSID** is not secret and lives in `main.yml` as `wifi_ssid`; change
the `CHANGE_ME_SSID` placeholder there.

### 3. Adjust the non-secret settings

At minimum review:

```yaml
target_disk: /dev/nvme0n1
workstation_hostname: ustation
workstation_user: devops
wifi_ssid: "CHANGE_ME_SSID"
timezone: Europe/Moscow
```

Everything else has a documented default in `inventory/group_vars/all/main.yml`.
Interface names `wifi_interface` / `ethernet_interface` are auto-detected from
`/sys/class/net` when left empty; override them if the installed names differ.

### 4. Run the install from the LiveCD

```bash
# first run on a blank or repartitionable disk
ansible-playbook playbooks/install.yml \
  -e disk_confirm_wipe=true \
  --ask-vault-pass
```

What happens, in order:

1. UEFI, root, LiveCD-medium, disk existence/size, secrets and interface checks.
2. `parted` creates GPT: 1 GiB ESP, 1 GiB `/boot`, remainder LUKS2.
3. `cryptsetup luksFormat` (AES-XTS-512, Argon2, 5 s calibration), LVM `vg0`,
   XFS volumes, ext4 `/boot`, FAT32 ESP.
4. `debootstrap --arch=amd64v3 --variant=minbase resolute /mnt`.
5. `/dev`, `/proc`, `/sys`, `/run` are `--rbind`-mounted into the chroot.
6. APT is switched to HTTPS DEB822, `snapd` is pinned to `-10`, the base APT
   policy is written, then the full package manifest is installed.
7. `/etc/environment`, keyboard, `systemd-networkd`, `wpa_supplicant`,
   `systemd-resolved`, sysctl, fstab/crypttab, ZRAM, `doas`, locales, hostname,
   user, services.
8. GRUB (with `GRUB_DEFAULT=saved`, no custom theme), MOK key, the plain
   Plymouth `text` theme, `update-initramfs -k all`.
9. `apt clean`, `apt autoremove`, recursive unmount, LUKS close.

The playbook **does not reboot**.  Re-run it without
`-e disk_confirm_wipe=true` to resume after a failure; existing partitions,
LUKS header, LVM volumes and filesystems are reused.

### 5. First boot

1. Remove the USB stick and reboot.
2. In **MOK Manager** choose `Enroll MOK` → `Continue` → `Yes`, enter the MOK
   password, then `Reboot`.
3. Type the LUKS passphrase in the **US console layout** (first layout in
   `/etc/default/keyboard`).
4. Log in as `workstation_user` (root login is locked).

Verification from the document:

```bash
doas id                                         # uid=0(root)
ip a
networkctl status wlp3s0
ping -c 3 1.1.1.1
sysctl net.ipv4.tcp_congestion_control          # bbr
sysctl net.core.default_qdisc                   # fq
zramctl                                         # /dev/zram0
```

### 6. Post-install (optional, after the first boot)

`opendoas` is not a supported Ansible become plugin, so the post-install
playbook expects to run as root.  The project must be present on the installed
system (copy it over, or clone it), together with `ansible-core`:

```bash
# on the installed workstation, as the daily user
doas apt install -y ansible-core
# python3-apt is already in the base manifest; it is a hard requirement for
# the ansible.builtin.apt module used by the post-install roles.

doas env HOME=/root ansible-playbook \
  playbooks/post-install.yml --ask-vault-pass
```

Or, if `ansible-playbook` is only in your user's virtualenv:

```bash
doas env HOME=/root PATH="$PATH" ansible-playbook \
  playbooks/post-install.yml --ask-vault-pass
```

Tags let you run one part at a time:

```bash
doas env HOME=/root ansible-playbook playbooks/post-install.yml --ask-vault-pass --tags nvidia
doas env HOME=/root ansible-playbook playbooks/post-install.yml --ask-vault-pass --tags libvirt
doas env HOME=/root ansible-playbook playbooks/post-install.yml --ask-vault-pass --tags containers
doas env HOME=/root ansible-playbook playbooks/post-install.yml --ask-vault-pass --tags restic -e restic_enabled=true
```

`nvidia` installs `nvidia-driver-610` from Ubuntu (Canonical's 615 branch with
the Linux 7.0 build fixes), configures `/etc/dkms/framework.conf` so DKMS signs
every module build with the machine-owned MOK key, and reports whether a reboot
is required.  It never reboots automatically.

## Important variables

| Variable | Default | Meaning |
| --- | --- | --- |
| `target_disk` | `/dev/nvme0n1` | Whole disk that receives the GPT layout. |
| `disk_confirm_wipe` | `false` | Must be `true` to repartition/wipe. |
| `disk_min_size_gb` | `560` | Size guard for the default LVM layout. |
| `efi_size_mib` / `boot_size_mib` | `1024` | ESP and `/boot` size. |
| `lvm_volumes` | root/var_log/libvirt/containers/home | Names, sizes, mount points, options.  Last item must be `100%FREE`. |
| `debootstrap_arch` | `amd64v3` | Set to `amd64` for pre-Haswell CPUs. |
| `ubuntu_suite` | `resolute` | Ubuntu 26.04 codename. |
| `apt_mirror_https` | `https://ru.archive.ubuntu.com/ubuntu/` | Change for another country. |
| `workstation_hostname` | `ustation` | Written to `/etc/hostname` and `/etc/hosts`. |
| `workstation_user` | `devops` | Daily account; always added to `wheel`. |
| `wifi_enabled` | `true` | Set `false` for a wired-only machine. |
| `wifi_interface` | auto | Override if predictable names differ between LiveCD and target. |
| `plymouth_theme` | `text` | Base-package theme; prints boot status instead of a logo.  Must exist inside the target. |
| `secure_boot_manage_mok` | `true` | Generate and import the MOK key. |
| `nvidia_driver_package` | `nvidia-driver-610` | Transitional package to the 615 branch. |
| `nerdctl_version` | `2.4.2` | nerdctl-full release. |
| `container_smoke_test` | `false` | Run `nerdctl run --rm hello-world`. |
| `restic_enabled` | `false` | Enable the optional backup role. |

## What cannot be automated

These steps are intentionally left to a human:

* **MOK Manager enrollment.**  Secure Boot requires a physical confirmation at
  the next boot.  The playbook imports the key request, nothing more.
* **Reboot.**  An unattended reboot of a freshly installed encrypted machine is
  a good way to lose it.  The final task prints the exact next steps.
* **LUKS passphrase entry.**  It is typed on the console, not stored in the
  target.  The playbook only stores it in the vault long enough to format and
  open the container.
* **Wi-Fi password correctness.**  The playbook cannot verify association
  before the first boot; check `wpa_cli -i <iface> status` afterwards.
* **Disk layout choices.**  `cfdisk` in the document is interactive; the
  playbook uses `parted` and requires the explicit `disk_confirm_wipe` flag.

## Document deviations and why

The source document contains a few internal contradictions.  The playbook makes
a deliberate choice for each one; see `docs/MAPPING.md` for the full table.

1. **Package list.**  The prose excludes `manpages-posix-dev`, `qrencode`,
   `xfsdump`, `python3-rich`, `linux-perf` and `linux-tools-7.0.0-42-generic`.  
   The manifest follows the prose and replaces the pinned tools package with
   `linux-tools-generic-hwe-26.04`.  `bpftool` stays.  The two theme packages
   from the document are not installed: `grub-theme-breeze` is unnecessary
   because GRUB uses its plain menu, and the `text` Plymouth theme ships in the
   base `plymouth` package, so `plymouth-themes` is not needed.
2. **ZRAM.**  The document installs `zram-tools` but writes
   `/etc/systemd/zram-generator.conf`, which is consumed by
   `systemd-zram-generator`.  Only the generator is installed; `zram-tools`
   (with its competing `zramswap.service`) is deliberately removed.
3. **fstab.**  The document generates fstab with a shell script.  Ansible writes
   the same layout idempotently from UUID facts, and installs the script as
   `/usr/local/sbin/gen-fstab` for future maintenance.
4. **Bind mounts.**  `--rbind` is used for `/dev`, `/sys`, `/run` so that
   `/dev/pts`, `/dev/shm` and `/sys/firmware/efi/efivars` are visible to
   `grub-install` and `mokutil`.
5. **Time synchronisation.**  `timedatectl set-ntp true` needs a running systemd.
   The playbook enables `systemd-timesyncd` offline; NTP is active on boot.
6. **Post-install privileges.**  The document uses `doas`; Ansible has no
   built-in `doas` become plugin, so `post-install.yml` runs as root.
7. **MOK generation.**  `mokutil --generate-key` does not exist in current
   versions; the playbook uses the `openssl` command from stage 7.1.

## Operational caveats

* **`--check` is not supported for the install playbook.**  Most tasks are
  `command` calls that partition disks, format filesystems and enter a chroot;
  Ansible cannot simulate them meaningfully.  Use `--syntax-check` on a
  controller and test on a scratch disk instead.  `--check` is only useful for
  the template-only parts of `post-install.yml`, and even there the package and
  service tasks are skipped.
* **LUKS unlock latency.**  `luks_iter_time: 5000` makes Argon2 spend about
  five seconds calibrating, which is paid on every unlock.  It is a deliberate
  trade-off against offline passphrase guessing.  Lower it to 2000–3000 before
  the first format if boot time matters more; changing it later requires
  `cryptsetup luksChangeKey` and is outside this playbook.
* **Unattended updates vs. the NVIDIA DKMS module.**  `unattended-upgrades` is
  enabled and may install a new kernel.  DKMS rebuilds the module only when the
  matching `linux-headers` package is present and `/etc/dkms/framework.conf`
  points at the enrolled MOK key (the `nvidia` role enforces both).  The new
  module still needs a reboot before it is loaded.  Inspect with
  `dkms status`; to avoid surprise kernel jumps, pin `linux-generic-hwe-26.04`
  or add it to `Unattended-Upgrade::Package-Blacklist`.  Set
  `unattended_upgrades_enable: false` before the install for fully manual
  updates.
* **No firewall by default.**  `nftables` is in the manifest, but the document
  defines no ruleset and no role writes one.  The installed system therefore
  has no netfilter policy; write `/etc/nftables.conf` and enable
  `nftables.service` if you need a firewall.
* **`nerdctl-full` shadows distribution binaries.**  The upstream archive
  installs `nerdctl`, `buildkitd`, `runc`, `containerd` helpers and CNI
  plugins under `/usr/local`, which precedes `/usr/bin` in `PATH`.  The systemd
  unit still starts the distribution `containerd`; version skew between
  `/usr/local/bin/runc` and `/usr/bin/containerd` is the usual cause of
  confusing container errors.
* **The LiveCD-medium guard is best-effort.**  It locates the boot medium via
  `/run/live/medium` or `/cdrom` and compares its parent disk with
  `target_disk`.  A `toram` boot may leave no mounted medium, so the check
  cannot fire; the explicit `disk_confirm_wipe=true` remains the real
  safeguard.

## Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `does not contain the expected three partitions` | First run: add `-e disk_confirm_wipe=true`.  Later runs: check `lsblk`. |
| `is also the disk the running Live environment was booted from` | `target_disk` points at the LiveUSB.  Unplug it or change `target_disk`; do not bypass the guard. |
| `disk is only N GiB` | Shrink `lvm_volumes` or lower `disk_min_size_gb` deliberately. |
| `debootstrap` fails with unsupported architecture | Set `-e debootstrap_arch=amd64`. |
| `apt-get install` cannot find a package | The 26.04 archive may differ from the document; edit `inventory/group_vars/all/packages.yml`.  The `packages` role dry-runs the transaction first, so the error names the missing package.  Re-run the full install playbook: the disk role remounts the existing layout and does not repartition while `disk_confirm_wipe` is false. |
| `snapd` has a candidate | Check `/mnt/etc/apt/preferences.d/nosnap.pref` and `apt-cache policy snapd` in the chroot. |
| `grub-install` cannot access EFI variables | Ensure `/sys/firmware/efi/efivars` is visible; the chroot role uses `--rbind /sys`. |
| MOK import fails with "cannot read password" | Run it manually after boot: `doas mokutil --import /var/lib/shim-signed/mok/MOK.der`. |
| `nvidia-smi` fails after install | Reboot and complete MOK enrollment; check `dkms status` and `/etc/dkms/framework.conf`. |
| `doas` denies the user | The user must be in `wheel`; check `id` and `/etc/doas.conf`. |
| `nerdctl` cannot connect | `systemctl status containerd buildkit`; check `/run/containerd/containerd.sock`. |
| No Wi-Fi after boot | `journalctl -u systemd-networkd -u wpa_supplicant@<iface> -b`; `wpa_cli -i <iface> status`. |

## Security notes

* Root login is locked; `wheel` + `doas` is the only privilege path.
* Secrets belong in the vault, never in `main.yml` or the shell history.
* `disk_confirm_wipe` is a deliberate second factor against accidental wipes.
* The target stores no LUKS passphrase; only the LUKS header and key slots do.
* Rootful containers are a documented trade-off; do not run untrusted images.
* Review `packages.yml` before changing it: `--no-install-recommends` plus
  `Install-Recommends=false` means missing Recommended packages are silent.

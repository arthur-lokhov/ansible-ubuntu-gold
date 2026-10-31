# Mapping: document → Ansible

`docs/expert-install.md` is the specification.  This table shows where each
section is implemented, so a change in the document has an obvious target.

## Stage 0 — hardware inventory

| Document | Implementation |
| --- | --- |
| 0.3 hardware snapshot | `roles/preflight/tasks/main.yml` → `hw-info.txt`, fetched to `artifacts/<host>-hw-info.txt` |
| 0.4 hardware-driven package choice | `inventory/group_vars/all/packages.yml` (firmware and driver comments) |
| 0.5 firmware strategy | `packages.yml`, last block |
| 0.6 systemd/S6 separation | not implemented in this playbook (S6/Qtile are stage 8 in the document and outside `expert-install.md`) |

## Stage 1 — disk, LUKS, LVM

| Document | Implementation |
| --- | --- |
| UEFI check | `roles/preflight` |
| LiveCD medium guard | `roles/preflight` (parent disk of `/run/live/medium` or `/cdrom`) |
| GPT layout | `roles/disk` → `parted` tasks |
| LUKS2 parameters | `roles/disk` → `cryptsetup luksFormat` (`luks_*` variables) |
| LVM volumes | `roles/disk` → `lvcreate`; layout in `lvm_volumes` |
| XFS/ext4/vfat | `roles/disk` → `mkfs.*` tasks |
| mount order | `roles/disk` → root first, then `/boot`, ESP, LVM volumes |

The interactive `cfdisk` is replaced by `parted`; `disk_confirm_wipe=true` is
the equivalent of the confirmation inside `cfdisk`.

## Stage 2 — debootstrap

| Document | Implementation |
| --- | --- |
| mount target tree | `roles/disk` |
| `debootstrap --arch=amd64v3` | `roles/debootstrap` |
| bind `/dev`, `/dev/pts`, `/proc`, `/sys`, `/run` | `roles/chroot_mounts` (`--rbind` for nested mounts) |
| `[chroot]` PS1 | not needed: Ansible never keeps an interactive chroot shell |

## Stage 3 — APT

| Document | Implementation |
| --- | --- |
| 3.1 ca-certificates first | `roles/apt_repo` first two tasks |
| 3.1 DEB822 HTTPS source | `roles/apt_repo/templates/ubuntu.sources.j2` |
| 3.1 TLS verification file | `roles/apt_repo/templates/99-https-verify.j2` |
| 3.2 Snap pin | `roles/apt_repo/templates/nosnap.pref.j2` + assertion on `apt-cache policy snapd` |
| 3.3 base policy | `roles/apt_repo/templates/00-base-policy.j2` |
| 3.4 package transaction | `roles/packages` + `inventory/group_vars/all/packages.yml` |

## Stage 4 — base configuration

| Document | Implementation |
| --- | --- |
| 4.1 `/etc/environment` | `roles/base_config/templates/environment.j2` |
| 4.2 keyboard | `keyboard.j2` + `/etc/default/keyboard` |
| 4.3 networkd + wpa_supplicant | `roles/base_config` network tasks and templates |
| 4.4 sysctl | `99-network-performance.conf.j2` |
| 4.5 crypttab | `crypttab.j2` (UUID from `roles/disk`) |
| 4.5 fstab | `fstab.j2`; maintenance script `gen-fstab.j2` |
| 4.6 ZRAM | `zram-generator.conf.j2` |
| 4.7 opendoas | `doas.conf.j2`, `wheel` creation |
| 4.8 locale/timezone | `locale.gen.j2`, `localtime` symlink, `systemd-timesyncd` |
| 4.9 hostname/hosts | `hosts.j2`, `/etc/hostname` |
| 4.10 root lock + user | `passwd -l root`, `useradd`, `chpasswd`, group checks |

## Stage 5 — boot

| Document | Implementation |
| --- | --- |
| 5.1 `/etc/default/grub` | `roles/boot/templates/grub.j2` |
| 5.1 `grub-install` / `update-grub` | `roles/boot` |
| 5.2 MOK key + import | `roles/boot` openssl + `mokutil` |
| 5.3 Plymouth (`text`) | `update-alternatives --install/--set default.plymouth` |
| initramfs | `update-initramfs -u -k all` |

## Stage 6 — finalize

| Document | Implementation |
| --- | --- |
| `apt clean`, `autoremove` | `roles/finalize` |
| `umount -R /mnt` | `roles/finalize` |
| `cryptsetup close` | `roles/finalize` |
| reboot / MOK Manager | **manual**; printed by `roles/finalize` |

## Stage 7 — post-install

| Document | Implementation |
| --- | --- |
| 7.1 NVIDIA + MOK | `roles/nvidia` |
| 7.2 libvirt/KVM | `roles/libvirt` |
| 7.3 rootful nerdctl/containerd/BuildKit | `roles/containers` |
| restic (from `all-my-work/rustic.md`) | `roles/restic`, disabled by default |

## Deliberate differences

| Topic | Document | Playbook | Reason |
| --- | --- | --- | --- |
| Partitioning | interactive `cfdisk` | `parted` + `disk_confirm_wipe=true` | Ansible cannot drive an interactive TUI; the flag preserves the confirmation. |
| fstab | shell script run once | Jinja template + installed `gen-fstab` | Idempotent task, same layout and options; script remains for maintenance. |
| Chroot mounts | `--bind` | `--rbind` for `/dev`, `/sys`, `/run` | `/dev/pts`, `/dev/shm`, `/sys/firmware/efi/efivars` must be visible to apt, grub-install and mokutil. |
| NTP | `timedatectl set-ntp true` | `systemctl enable systemd-timesyncd` | No PID 1 inside the chroot; enabling the unit is the offline equivalent. |
| ZRAM | `zram-tools` + `zram-generator.conf` | `systemd-zram-generator` only; `zram-tools` is purged if present | `zram-tools` ships a competing `zramswap.service` and ignores `zram-generator.conf`. |
| GRUB theme | Breeze theme from `grub-theme-breeze` | plain GRUB menu, no `GRUB_THEME` | A themed `/boot` adds a package and a copy step without improving a build workstation; the menu also survives theme package changes. |
| Plymouth | `spinner` | `text` | Console-style theme that prints boot status messages instead of a logo; it ships in the base `plymouth` package. |
| Package list | prose and final command disagree | manifest follows the prose, replaces the pinned tools package, drops both theme packages | `grub-theme-breeze` is unused and `text.plymouth` is provided by `plymouth`. |
| MOK | `mokutil --generate-key` (stage 5.2) / `openssl` (stage 7.1) | `openssl` | `--generate-key` does not exist in current `mokutil`; stage 7.1 already corrects this. |
| Post-install privilege | `doas` | playbook runs as root | No built-in `doas` become plugin in `ansible-core`. |
| Reboot | manual | not automated | MOK Manager is a physical confirmation; an unattended reboot is unsafe. |

## Files written into the target

```
/etc/environment
/etc/default/keyboard
/etc/default/grub
/etc/systemd/network/20-wired.network         (if a wired NIC exists)
/etc/systemd/network/25-wireless.network      (if wifi_enabled)
/etc/wpa_supplicant/wpa_supplicant-<if>.conf  (mode 0600)
/etc/resolv.conf -> /run/systemd/resolve/stub-resolv.conf
/etc/sysctl.d/99-network-performance.conf
/etc/systemd/zram-generator.conf
/etc/doas.conf                                (mode 0400)
/etc/locale.gen
/etc/localtime -> /usr/share/zoneinfo/<tz>
/etc/hostname
/etc/hosts
/etc/fstab
/etc/crypttab
/etc/apt/sources.list.d/ubuntu.sources
/etc/apt/apt.conf.d/00-base-policy
/etc/apt/apt.conf.d/20auto-upgrades
/etc/apt/apt.conf.d/99-https-verify
/etc/apt/preferences.d/nosnap.pref
/etc/needrestart/conf.d/50-ansible.conf
/usr/local/sbin/gen-fstab
/etc/alternatives/default.plymouth -> /usr/share/plymouth/themes/text/text.plymouth
/var/lib/shim-signed/mok/MOK.priv, MOK.der
```

Post-install additionally writes:

```
/etc/dkms/framework.conf                     (mok_signing_key / mok_certificate)
/etc/containerd/config.toml
/etc/buildkit/buildkitd.toml
/etc/systemd/system/buildkit.service
/etc/restic/password                          (only when restic_enabled)
/usr/local/bin/backup.sh                      (only when restic_enabled)
/home/<user>/.bashrc                          (nerdctl aliases)
```

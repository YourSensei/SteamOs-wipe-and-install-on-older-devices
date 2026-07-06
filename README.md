Copy and paste is possible

#!/bin/bash
# -*- mode: sh; indent-tabs-mode: nil; sh-basic-offset: 2; -*-
# vim: et sts=2 sw=2
#
# A collection of functions to create, repair, or modify a SteamOS installation.
# This makes a number of assumptions about the target device and will be
# destructive if you have modified the expected partition layout.
#

set -eu

die() { echo >&2 "!! $*"; exit 1; }
readvar() { IFS= read -r -d '' "$1" || true; }

OWD="$(cd -- "$(dirname -- "$0")" && pwd)"

DOPARTVERIFY=1

# Autodetect the target disk unless the caller already exported an override (e.g. DISK=/dev/sda ./repair_device.sh).
# Prefers an nvme device if one is present (matches original stock behavior), otherwise falls back to the first
# non-removable SATA/ATA disk (sdX). DISK_SUFFIX is "p" for nvme (partitions are nvme0n1p1) and empty for sdX
# (partitions are sda1).
detect_disk()
{
  local nvme sata
  nvme="$(lsblk -dn -o NAME,TYPE 2>/dev/null | awk '$2=="disk" && $1 ~ /^nvme/ {print $1; exit}')"
  if [[ -n $nvme ]]; then
    echo "/dev/$nvme" p
    return 0
  fi

  sata="$(lsblk -dn -o NAME,TYPE,RM 2>/dev/null | awk '$2=="disk" && $3==0 && $1 ~ /^sd/ {print $1; exit}')"
  if [[ -n $sata ]]; then
    echo "/dev/$sata" ""
    return 0
  fi

  return 1
}

if [[ -z ${DISK-} ]]; then
  if read -r _detected_disk _detected_suffix < <(detect_disk); then
    DISK="$_detected_disk"
    DISK_SUFFIX="$_detected_suffix"
  else
    DISK=/dev/nvme0n1
    DISK_SUFFIX=p
  fi
fi
DISK_SUFFIX="${DISK_SUFFIX-}"

# True if $DISK is an NVMe device - used to gate nvme-only tooling (sanitize, Jupiter BIOS/controller updates) so
# this script also works against older SATA SSDs (e.g. non-Jupiter hardware like a Dell Latitude).
is_nvme() { [[ $DISK == /dev/nvme* ]]; }

# If this exists, use the jupiter-biosupdate binary from this directory, and set JUPITER_BIOS_DIR to this directory when
# invoking it.  Used for including a newer bios payload than the base image.
VENDORED_BIOS_UPDATE=/home/deck/jupiter-bios
# If this exists, use the jupiter-controller-update binary from this directory, and set
# JUPITER_CONTROLLER_UPDATE_FIRMWARE_DIR to this directory when invoking it.  Used for including a newer controller
# payload than the base image.
VENDORED_CONTROLLER_UPDATE=/home/deck/jupiter-controller-fw

# Partition table, sfdisk format, %%DISKPART%% filled in
#
PART_SIZE_ESP="256"
PART_SIZE_EFI="64"
PART_SIZE_ROOT="5120" # This should match the size from the input disk build
PART_SIZE_VAR="256"
PART_SIZE_HOME="100" # For the stub .img file we're making this can be tiny, OS expands to fill physical disk on first
                     # boot.  We make sure to specify the inode ratio explicitly when formatting.

# Total size + 1MiB padding at beginning/end for GPT structures.
DISK_SIZE=$(( 2 + PART_SIZE_HOME + PART_SIZE_ESP + 2 * ( PART_SIZE_EFI + PART_SIZE_ROOT + PART_SIZE_VAR ) ))
# Alignment: Using general sizes like MiB and no explicit start offset points causes sfdisk to align to MiB boundaries
#            by default (e.g. first partition will start at 1MiB). See `man sfdisk`.

# Sector size: Most physical SSD/NVMe/etc use logical 512 sectors*. GPT partition tables aren't portable between varying
#              sector sizes, so this .img cannot be used directly on a 4k-logical-sector device (a quick search suggests
#              this is most likely with certain VM/cloud/network disks)
#

#              Since we use 1MiB alignment, it should be possible to fixup this partition table for other sector sizes
#              without physically moving any partitions at imaging time:
#
#                  dd if=output.img of=/target/disk
#
#                  # sfdisk will default to 512 for a local file, dumping the table correctly, then translate it to the
#                  # target device's sector size upon re-writing:
#
#                  sfdisk -d < output.img | sfdisk /target/disk
#
#              Alternatively, use `losetup --sector-size` to remount the image at a different size, and use the above
#              steps to regenerate the table.  If this comes up often in practice we could output a "partitions4096.gpt"
#              style file alongside the disk image that could be `dd`'d on top for weird VM setups.
#
#              *Note: logical sectors != physical sectors != optimal I/O alignment.  Logical sectors being the unit the
#               OS addresses the disk by, and what GPT tables use as their basic written-to-disk unit.  Most everything
#               is 512 or (rarely) 4096.

TARGET_SECTOR_SIZE=512 # Passed to `losetup` to emulate, affects the sector-offsets sfdisk ends up writing.
readvar PARTITION_TABLE <<END_PARTITION_TABLE
  label: gpt
  %%DISKPART%%1: name="esp",      size=         ${PART_SIZE_ESP}MiB,  type=C12A7328-F81F-11D2-BA4B-00A0C93EC93B
  %%DISKPART%%2: name="efi-A",    size=         ${PART_SIZE_EFI}MiB,  type=EBD0A0A2-B9E5-4433-87C0-68B6B72699C7
  %%DISKPART%%3: name="efi-B",    size=         ${PART_SIZE_EFI}MiB,  type=EBD0A0A2-B9E5-4433-87C0-68B6B72699C7
  %%DISKPART%%4: name="rootfs-A", size=         ${PART_SIZE_ROOT}MiB, type=4F68BCE3-E8CD-4DB1-96E7-FBCAF984B709
  %%DISKPART%%5: name="rootfs-B", size=         ${PART_SIZE_ROOT}MiB, type=4F68BCE3-E8CD-4DB1-96E7-FBCAF984B709
  %%DISKPART%%6: name="var-A",    size=         ${PART_SIZE_VAR}MiB,  type=4D21B016-B534-45C2-A9FB-5C16E091FD2D
  %%DISKPART%%7: name="var-B",    size=         ${PART_SIZE_VAR}MiB,  type=4D21B016-B534-45C2-A9FB-5C16E091FD2D
  %%DISKPART%%8: name="home",     size=         ${PART_SIZE_HOME}MiB, type=933AC7E1-2EB4-4F13-B844-0E14E2AEF915
END_PARTITION_TABLE

# Partition numbers on ideal target device, by index
FS_ESP=1
FS_EFI_A=2
FS_EFI_B=3
FS_ROOT_A=4
FS_ROOT_B=5
FS_VAR_A=6
FS_VAR_B=7
FS_HOME=8

diskpart() { echo "$DISK$DISK_SUFFIX$1"; }

##
## Util colors and such
##

err() {
  echo >&2
  eerr "Imaging error occured, see above and restart process."
  sleep infinity
}
trap err ERR

_sh_c_colors=0
[[ -n $TERM && -t 1 && ${TERM,,} != dumb ]] && _sh_c_colors="$(tput colors 2>/dev/null || echo 0)"
sh_c() { [[ $_sh_c_colors -le 0 ]] || ( IFS=\; && echo -n $'\e['"${*:-0}m"; ); }

sh_quote() { echo "${@@Q}"; }
estat()    { echo >&2 "$(sh_c 32 1)::$(sh_c) $*"; }
emsg()     { echo >&2 "$(sh_c 34 1)::$(sh_c) $*"; }
ewarn()    { echo >&2 "$(sh_c 33 1);;$(sh_c) $*"; }
einfo()    { echo >&2 "$(sh_c 30 1)::$(sh_c) $*"; }
eerr()     { echo >&2 "$(sh_c 31 1)!!$(sh_c) $*"; }
die() { local msg="$*"; [[ -n $msg ]] || msg="script terminated"; eerr "$msg"; exit 1; }
showcmd() { showcmd_unquoted "${@@Q}"; }
showcmd_unquoted() { echo >&2 "$(sh_c 30 1)+$(sh_c) $*"; }
cmd() { showcmd "$@"; "$@"; }

# Helper to format
fmt_ext4()  { [[ $# -eq 2 && -n $1 && -n $2 ]] || die; cmd sudo mkfs.ext4 -F -L "$1" "$2"; }
fmt_fat32() { [[ $# -eq 2 && -n $1 && -n $2 ]] || die; cmd sudo mkfs.vfat -n"$1" "$2"; }

##
## Prompt mechanics - currently using Zenity
##

# Give the user a choice between Proceed, or Cancel (which exits this script)
#  $1 Title
#  $2 Text
#
prompt_step()
{
  title="$1"
  msg="$2"
  unconditional="${3-}"
  if [[ ! ${unconditional-} && ${NOPROMPT:-} ]]; then
    echo -e "$msg"
    return 0
  fi
  zenity --title "$title" --question --ok-label "Proceed" --cancel-label "Cancel" --no-wrap --text "$msg"
  [[ $? = 0 ]] || exit 1
}

prompt_reboot()
{
  local msg=$1
  local mode="reboot"
  [[ ${POWEROFF:-} ]] && mode="shutdown"

  prompt_step "Action Successful" "${msg}\n\nChoose Proceed to $mode now, or Cancel to stay in the repair image." "${REBOOTPROMPT:-}"
  [[ $? = 0 ]] || exit 1
  if [[ ${POWEROFF:-} ]]; then
    cmd systemctl poweroff
  else
    cmd systemctl reboot
  fi
}

##
## Repair functions
##

# verify partition on target disk - at least make sure the type and partlabel match what we expect.
#   $1 device
#   $2 expected type
#   $3 expected partlabel
#
verifypart()
{
  [[ $DOPARTVERIFY = 1 ]] || return 0
  TYPE="$(blkid -o value -s TYPE "$1" )"
  PARTLABEL="$(blkid -o value -s PARTLABEL "$1" )"
  if [[ ! $TYPE = "$2" ]]; then
    eerr "Device $1 is type $TYPE but expected $2 - cannot proceed. You may try full recovery."
    sleep infinity ; exit 1
  fi

  if [[ ! $PARTLABEL = $3 ]] ; then 
    eerr "Device $1 has label $PARTLABEL but expected $3 - cannot proceed. You may try full recovery."
    sleep infinity ; exit 2
  fi
}

# Replace the device rootfs (btrfs version). Source must be frozen before calling.
#   $1 source device
#   $2 target device
#
imageroot()
{
  local srcroot="$1"
  local newroot="$2"
  # copy then randomize target UUID - careful here! Duplicating btrfs ids is a problem
  cmd dd if="$srcroot" of="$newroot" bs=128M status=progress oflag=sync
  cmd btrfstune -f -u "$newroot"
  cmd btrfs check "$newroot"
}

# Set up boot configuration in the target partition set
#   $1 partset name
finalize_part()
{
  estat "Finalizing install part $1"
  cmd steamos-chroot --no-overlay --disk "$DISK" --partset "$1" -- mkdir /efi/SteamOS
  cmd steamos-chroot --no-overlay --disk "$DISK" --partset "$1" -- mkdir -p /esp/SteamOS/conf
  cmd steamos-chroot --no-overlay --disk "$DISK" --partset "$1" -- steamos-partsets /efi/SteamOS/partsets
  cmd steamos-chroot --no-overlay --disk "$DISK" --partset "$1" -- steamos-bootconf create --image "$1" --conf-dir /esp/SteamOS/conf --efi-dir /efi --set title "$1"
  cmd steamos-chroot --no-overlay --disk "$DISK" --partset "$1" -- grub-mkimage
  cmd steamos-chroot --no-overlay --disk "$DISK" --partset "$1" -- update-grub
  if [[ -e $OWD/steamos-branch ]]; then
    # This is a hack for main image installers only that opts the new device into main by default
    # shellcheck disable=SC2016 # intended
    cmd steamos-chroot --no-overlay --disk "$DISK" --partset "$1" -- bash -c 'mkdir /var/lib && printf "%s\n" "$1" > /var/lib/steamos-branch' -- "$(cat -- "$OWD"/steamos-branch)"
 fi
}

##
## Main
##

onexit=()
exithandler() {
  for func in "${onexit[@]}"; do
    "$func"
  done
}
trap exithandler EXIT

# Check existence of target disk
if [[ ! -e "$DISK" ]]; then
  eerr "$DISK does not exist -- no supported drive (nvme or sata) detected?"
  sleep infinity
  exit 1
fi

# Desktop environments (e.g. the SteamOS/Plasma recovery session) can auto-mount any recognizable filesystem on
# $DISK under /run/media/... as soon as it's visible, or a partition on it may be active swap. sfdisk refuses to
# write a new partition table while anything on the disk is busy, so clear all of that first.
unmount_target_disk()
{
  local part busy=0

  estat "Making sure nothing on ${DISK} is mounted or in use as swap"

  # swapoff any swap partition living on the target disk
  while read -r swapdev _; do
    [[ $swapdev == "${DISK}"* ]] && cmd swapoff "$swapdev"
  done < <(swapon --show=NAME --noheadings 2>/dev/null)

  # unmount any partitions of the target disk, in case the desktop environment auto-mounted them
  for part in "${DISK}${DISK_SUFFIX}"*; do
    [[ -e $part ]] || continue
    [[ $part == "$DISK" ]] && continue
    if findmnt -S "$part" &>/dev/null; then
      cmd umount "$part" || busy=1
    fi
  done

  if [[ $busy = 1 ]]; then
    ewarn "Could not unmount everything on ${DISK} -- repartitioning may still fail."
  fi
}

# Reinstall a fresh SteamOS copy.
#
repair_steps()
{
  if [[ $writePartitionTable = 1 ]]; then
    unmount_target_disk
    estat "Write known partition table"
    echo "$PARTITION_TABLE" | sfdisk "$DISK"

  elif [[ $writeOS = 1 || $writeHome = 1 ]]; then

    # verify some partition settings to make sure we are ok to proceed with partial repairs
    # in the case we just wrote the partition table, we know we are good and the partitions
    # are unlabelled anyway
    verifypart "$(diskpart $FS_ESP)" vfat esp
    verifypart "$(diskpart $FS_EFI_A)" vfat efi-A
    verifypart "$(diskpart $FS_EFI_B)" vfat efi-B
    verifypart "$(diskpart $FS_VAR_A)" ext4 var-A
    verifypart "$(diskpart $FS_VAR_B)" ext4 var-B
    verifypart "$(diskpart $FS_HOME)" ext4 home
  fi

  # clear the var partition (user data), but also if we are reinstalling the OS
  # a fresh system partition has problems with overlay otherwise
  if [[ $writeOS = 1 || $writeHome = 1 ]]; then
    estat "Creating var partitions"
    fmt_ext4  var  "$(diskpart $FS_VAR_A)"
    fmt_ext4  var  "$(diskpart $FS_VAR_B)"
  fi

  # Create boot partitions
  if [[ $writeOS = 1 ]]; then
    # Set up ESP/EFI boot partitions
    estat "Creating boot partitions"
    fmt_fat32 esp  "$(diskpart $FS_ESP)"
    fmt_fat32 efi  "$(diskpart $FS_EFI_A)"
    fmt_fat32 efi  "$(diskpart $FS_EFI_B)"
  fi

  if [[ $writeHome = 1 ]]; then
    estat "Creating home partition..."
    cmd sudo mkfs.ext4 -F -O casefold -T huge -L home "$(diskpart $FS_HOME)"
    estat "Remove the reserved blocks on the home partition..."
    tune2fs -m 0 "$(diskpart $FS_HOME)"
  fi

  # Stage a BIOS update for next reboot if updating OS. OOBE images like this one don't auto-update the bios on boot.
  # jupiter-biosupdate is Jupiter/Deck-specific hardware, so this is skipped entirely on non-nvme (e.g. SATA) targets.
  if [[ $writeOS = 1 ]] && is_nvme; then
    estat "Staging a BIOS update for next boot if necessary"
    # If we included a VENDORED_BIOS_UPDATE directory above, use the newer payload there and point JUPITER_BIOS_DIR to
    # it.  Directory should contain both a newer tool and newer firmware.
    biostool=/usr/bin/jupiter-biosupdate
    if [[ -n $VENDORED_BIOS_UPDATE && -d $VENDORED_BIOS_UPDATE ]]; then
      biostool="$VENDORED_BIOS_UPDATE"/jupiter-biosupdate
      export JUPITER_BIOS_DIR="$VENDORED_BIOS_UPDATE"
    fi

    # This is cursed, but, we want to stage the capsule in the onboard nvme, which we are booting next
    fix_esp() {
      if [[ -n $mounted_esp ]]; then
        cmd umount -l /esp
        cmd umount -l /boot/efi
        mounted_esp=
      fi
    }
    onexit+=(fix_esp)
    einfo "Mounting new ESP/EFI on /esp /boot/efi for BIOS staging"
    cmd mount "$(diskpart $FS_ESP)" /esp
    cmd mount "$(diskpart $FS_EFI_A)" /boot/efi
    mounted_esp=1

    if [[ ${FORCEBIOS:-} ]]; then
      "$biostool" --force || "$biostool"
    else
      "$biostool"
    fi

    fix_esp
  fi

  # Perform a controller update if updating OS.  OOBE images like this one don't auto-update controllers on boot.
  # jupiter-controller-update is Jupiter/Deck-specific hardware, so this is skipped entirely on non-nvme targets.
  if [[ $writeOS = 1 ]] && is_nvme; then
    estat "Updating controller firmware if necessary"
    controller_tool="/usr/bin/jupiter-controller-update"
    # If we included a VENDORED_CONTROLLER_UPDATE directory above, use the newer payload and point
    # JUPITER_CONTROLLER_UPDATE_FIRMWARE_DIR to it.  Directory should contain both a newer tool and newer firmware.
    if [[ -n $VENDORED_CONTROLLER_UPDATE && -d $VENDORED_CONTROLLER_UPDATE ]]; then
      controller_tool="$VENDORED_CONTROLLER_UPDATE"/jupiter-controller-update
      export JUPITER_CONTROLLER_UPDATE_FIRMWARE_DIR="$VENDORED_CONTROLLER_UPDATE"
    fi

    JUPITER_CONTROLLER_UPDATE_IN_OOBE=1 "$controller_tool"
  fi

  if [[ $writeOS = 1 ]]; then
    # Find rootfs
    rootdevice="$(findmnt -n -o source / )"
    if [[ -z $rootdevice || ! -e $rootdevice ]]; then
      eerr "Could not find USB installer root -- usb hub issue?"
      sleep infinity
      exit 1
    fi

    # Freeze our rootfs
    estat "Freezing rootfs"
    unfreeze() { fsfreeze -u / || true; }
    onexit+=(unfreeze)
    cmd fsfreeze -f /

    estat "Imaging OS partition A"
    imageroot "$rootdevice" "$(diskpart $FS_ROOT_A)"

    estat "Imaging OS partition B"
    imageroot "$rootdevice" "$(diskpart $FS_ROOT_B)"

    unfreeze

    estat "Finalizing boot configurations"
    finalize_part A
    finalize_part B
    estat "Finalizing EFI system partition"
    cmd steamos-chroot --no-overlay --disk "$DISK" --partset A -- steamcl-install --flags restricted --force-extra-removable
  fi
}

# drop into the primary OS partset on the device
#
chroot_primary()
{
  partset=$( steamos-chroot --no-overlay --disk "$DISK" --partset "A" -- steamos-bootconf selected-image )

  estat "Dropping into a chroot on the $partset partition set."
  estat "You can make any needed changes here, and exit when done."

  # FIXME etc overlay dir might not exist on a fresh install and this will fail

  cmd steamos-chroot --disk "$DISK" --partset "$partset"
}

# return sanitize state (and echo the current percentage complete)
# 0 : ready to sanitize
# 1 : sanitize in progress (echo the current percentage)
# 2 : drive does not support sanitize
#
get_sanitize_progress()
{
  status=$(nvme sanitize-log "${DISK}" | grep "(SSTAT)" | grep -oEi "(0x)?[[:xdigit:]]+$") || return 2
  [[ $(( status % 8 )) -eq 2 ]] || return 0

  progress=$(nvme sanitize-log "${DISK}" | grep "(SPROG)" | grep -oEi "(0x)?[[:xdigit:]]+$") || return 2
  echo "sanitize progress: $(( ( progress * 100 )/ 65535 ))%"
  return 1
}


# Secure-erase a SATA/older SSD without depending on hdparm, since read-only recovery images often don't ship it
# and installing packages at runtime isn't possible on an immutable image.
#
# Strategy, in order of preference:
#   1. `blkdiscard --secure` - issues a hardware-level secure TRIM/erase via the kernel block layer (BLKSECDISCARD
#      ioctl). blkdiscard ships as part of util-linux, which is an essential base package on virtually every distro
#      (including SteamOS/Arch), so this needs no vendored binary. Equivalent in effect to ATA Security Erase on
#      drives that support it.
#   2. `blkdiscard` (non-secure) - plain TRIM. Not a guaranteed data-recovery-proof erase on all SSDs (depends on
#      the drive supporting deterministic read-after-trim), but much faster than a full overwrite and works on
#      basically all SSDs.
#   3. `dd` full-disk overwrite - last resort if the controller/drive doesn't support discard at all. This is NOT
#      a true secure erase on SSDs with wear-leveling (spare/remapped blocks won't be touched), but it's the only
#      option with zero external dependencies beyond coreutils.
#
sata_secure_erase()
{
  echo
  echo "Warning!"
  echo
  echo "This action irrevocably clears *all* user data from ${DISK}"
  echo "Pausing five seconds in case you didn't mean to do this..."
  sleep 5

  if command -v blkdiscard &>/dev/null; then
    estat "Attempting hardware secure erase (blkdiscard --secure) on ${DISK}..."
    if blkdiscard --secure "${DISK}" 2>/dev/null; then
      echo "... sanitize done (secure discard)."
      return 0
    fi
    ewarn "${DISK} does not support secure discard -- trying a regular discard/TRIM instead."

    estat "Attempting discard/TRIM on ${DISK}..."
    if blkdiscard "${DISK}" 2>/dev/null; then
      echo "... sanitize done (discard/TRIM)."
      return 0
    fi
    ewarn "${DISK} does not support discard/TRIM either -- falling back to a full overwrite."
  else
    ewarn "blkdiscard not found -- falling back to a full overwrite."
  fi

  estat "Overwriting ${DISK} with zeros (this can take a long time, please wait)..."
  ewarn "Note: on SSDs this does not guarantee remapped/spare blocks are erased. If possible, prefer a device"
  ewarn "that supports blkdiscard --secure instead."
  cmd dd if=/dev/zero of="${DISK}" bs=128M status=progress oflag=direct || true
  echo "... sanitize done (overwrite)."
}

# call nvme sanitize (or, on non-nvme/SATA disks, a discard/dd-based secure erase), blockwise, and wait for
# it to complete.
#
sanitize_all()
{
  if ! is_nvme; then
    sata_secure_erase
    return $?
  fi

  sres=0
  get_sanitize_progress || sres=$?
  case $sres in
    0)
      echo
      echo "Warning!"
      echo
      echo "This action irrevocably clears *all* user data from ${DISK}"
      echo "Pausing five seconds in case you didn't mean to do this..."
      sleep 5
      echo "Ok, let's go. Sanitizing ${DISK}:"
      nvme sanitize -a 2 "${DISK}"
      echo "Sanitize action started."
      ;;
    1) echo "An NVME sanitize action is already in progress."
      ;;
    2) # use NVME secure-format since this device does not appear to support sanitize
      nvme format "${DISK}" -n 1 -s 1 -r
      return 0
      ;;
    *) echo "Unexpected result from sanitize-log"
      return $sres
      ;;
  esac

  while ! get_sanitize_progress ; do
    sleep 5
  done

  echo "... sanitize done."
}

# print quick list of targets
#
help()
{
  readvar HELPMSG << EOD
This tool can be used to reinstall or repair your SteamOS installation

Possible targets:
    all : permanently destroy all data on the device, and (re)install SteamOS.
    system : repair/reinstall SteamOS on the device's system partitions, preserving user data partitions.
    home : reformat the devices /home and /var partitions, removing games and user data from the device.
    chroot : chroot into to the primary SteamOS partition set.
    sanitize : perform a drive sanitize/secure-erase operation (NVME sanitize, or ATA secure erase on SATA disks).
EOD
  emsg "$HELPMSG"
  if [[ "$EUID" -ne 0 ]]; then
    eerr "Please run as root."
    exit 1
  fi
}

[[ "$EUID" -ne 0 ]] && help

# Desktop automounters (udisks2-driven, used by Plasma/GNOME file managers) will grab newly-created or
# newly-visible partitions on $DISK the instant they appear -- including ones this script just created -- and
# mount them under /run/media/.... That collides with this script's own temporary mounts (steamos-chroot etc.)
# and leaves them "busy" on umount. Stop the automounter for the duration of any disk-touching operation and
# always restore it afterward, even on failure.
AUTOMOUNT_WAS_ACTIVE=0

disable_automount()
{
  if systemctl is-active --quiet udisks2 2>/dev/null; then
    estat "Temporarily stopping the automounter (udisks2) to avoid interference"
    if systemctl stop udisks2 2>/dev/null; then
      AUTOMOUNT_WAS_ACTIVE=1
    else
      ewarn "Could not stop udisks2 -- desktop automount may interfere with this operation."
    fi
  fi
}

restore_automount()
{
  if [[ $AUTOMOUNT_WAS_ACTIVE = 1 ]]; then
    estat "Restarting the automounter (udisks2)"
    systemctl start udisks2 2>/dev/null || ewarn "Could not restart udisks2 -- you may need to restart it manually."
    AUTOMOUNT_WAS_ACTIVE=0
  fi
}

trap restore_automount EXIT

writePartitionTable=0
writeOS=0
writeHome=0

case "${1-help}" in
all)
  prompt_step "Wipe Device & Install SteamOS" "This action will wipe and (re)install SteamOS on this device.\nThis will permanently destroy all data on your device.\n\nThis cannot be undone.\n\nChoose Proceed only if you wish to wipe and reinstall this device."
  writePartitionTable=1
  writeOS=1
  writeHome=1
  disable_automount
  sanitize_all
  repair_steps
  prompt_reboot "Reimaging complete."
  ;;
system)
  prompt_step "Repair SteamOS" "This action will repair the SteamOS installation on the device, while attempting to preserve your games and personal content.\nSystem customizations may be lost.\n\nChoose Proceed to reinstall SteamOS on your device."
  writeOS=1
  disable_automount
  repair_steps
  prompt_reboot "SteamOS reinstall complete."
  ;;
home)
  prompt_step "Delete local user data" "This action will reformat the home partitions on your device.\nThis will destroy downloaded games and all personal content, including system configuration.\n\nThis action cannot be undone.\n\nChoose Proceed to reformat all user home partitions."
  writeHome=1
  disable_automount
  repair_steps
  prompt_reboot "User partitions have been reformatted."
  ;;
chroot)
  disable_automount
  chroot_primary
  ;;
sanitize)
  prompt_step "Clear and sanitize disk" "This action will kick off a sanitize/secure-erase on the primary drive, irrevocably deleting all user data.\n\nThis action cannot be undone.\n\nChoose Proceed only if you want to remove all data from the current device's primary drive."
  disable_automount
  sanitize_all
  ;;
*)
  help
  ;;
esac 


```
#Gentoo Mirror
#Name               Protocol    IPv4/v6	    URL
#Ionos SE (1&1 )    http        IPv4 + IPv6 http://eu.mirror.ionos.com//linux/distributions/gentoo/gentoo/
#                   https       IPv4 + IPv6 https://eu.mirror.ionos.com//linux/distributions/gentoo/gentoo/
#                   rsync       IPv4 + IPv6 rsync://eu.mirror.ionos.com/gentoo/



#Drive to install to /dev/vd?, /dev/sd?, #/dev/nvme?n?, /dev/mmcblk?
#/dev/vda
#/dev/vda1, vfat,  EFI System, boot 2.5GB
#/dev/vda2, LUKS,  LVM2,LV=Root,LV=Swap

#This us for the rolling release stage-3 desktop openrc
#In Gentoo "current" is rolling releaseand "stable" is milestone release
#https://eu.mirror.ionos.com/linux/distributions/gentoo/gentoo/releases/amd64/autobuilds/current-stage3-amd64-desktop-openrc/stage3-amd64-desktop-openrc-20260614T170130Z.tar.xz

#Open Konsole
sudo -i
passwd
useradd -m -G users,wheel uwe
psswd uwe
su - uwe
#let's start sshd
sudo rc-service sshd start
#get our current ip to ssh into the installation
ip a

#now either continue in the console or ssh into the machine
sudo -i
#optional
wipefs -a /dev/vda
#for NVME SSDs
nvme sanitize /dev/nvme0 -a 0x02
#for SATA SSDs, eMMC, SD Cards
blkdiscard -vfz /dev/vda

#prepare the disk with parted
parted /dev/vda print
parted /dev/vda mklabel gpt
parted /dev/vda mkpart primary fat32 0% 2.5GB
parted /dev/vda name 1 esp
parted /dev/vda set 1 esp on
parted /dev/vda set 1 boot on
parted /dev/vda mkpart primary 2.5GB 100%
parted /dev/vda name 2 LUKS-crypt
parted /dev/vda print

#let's take care of the filesystems
# EFI is combined /boot and EFI so mount point is /boot
mkfs.fat -F 32 -n EFI /dev/vda1
cryptsetup luksFormat /dev/vda2
cryptsetup luksOpen /dev/vda2 crypt
cryptsetup refresh --persistent --allow-discards crypt
vgcreate system /dev/mapper/crypt
#I like swap to be 2.5 * RAM Size
lvcreate --name swap -L 40G system
lvcreate --name root -l 100%free system
mkfs.btrfs -f -L rootfs /dev/mapper/system-root
mkswap /dev/system/swap -L swapfs
mount -v -t btrfs -o ssd,compress=zstd:11,subvol=/ /dev/system/root /mnt/gentoo
btrfs subvolume create /mnt/gentoo/@
btrfs subvolume create /mnt/gentoo/@home
btrfs subvolume create /mnt/gentoo/@root
btrfs subvolume create /mnt/gentoo/@var@log
btrfs subvolume create /mnt/gentoo/@snapshots
btrfs subvolume create /mnt/gentoo/@home/.snapshots

#check if all is done correctly
btrfs subvolume list /mnt/gentoo
ID 256 gen 10 top level 5 path @
ID 257 gen 10 top level 5 path @home
ID 258 gen 10 top level 5 path @root
ID 259 gen 10 top level 5 path @var@log
ID 260 gen 10 top level 5 path @snapshots
ID 261 gen 10 top level 257 path @home/.snapshots

#set @ as the default subvol
btrfs subvolume set-default /mnt/gentoo/@

#check if it's set correctly
btrfs subvolume get-default /mnt/gentoo
ID 256 gen 10 top level 5 path @

umount /mnt/gentoo
#Time to mount everythin at it's place
mount -t btrfs -o compress=zstd:11,ssd,noatime,subvol=/@ /dev/system/root /mnt/gentoo
chmod 755 /mnt/gentoo
mount -m -t btrfs -o compress=zstd:11,ssd,noatime,subvol=/@root /dev/system/root /mnt/gentoo/root
mount -m -t btrfs -o compress=zstd:11,ssd,noatime,subvol=/@var@log /dev/system/root /mnt/gentoo/root/var/log
mount -m -t btrfs -o compress=zstd:11,ssd,noatime,subvol=/@snapshots /dev/system/root /mnt/gentoo/root/.snapshots
mount -m -t btrfs -o compress=zstd:11,ssd,noatime,subvol=/@home /dev/system/root /mnt/gentoo/home
mount -m -t vfat /dev/vda1 /mnt/gentoo/boot
swapon /dev/system/swap

#let's bootstrap our new gentoo system
cd /mnt/gentoo
curl -O https://eu.mirror.ionos.com/linux/distributions/gentoo/gentoo/releases/amd64/autobuilds/current-stage3-amd64-desktop-openrc/stage3-amd64-desktop-openrc-20260614T170130Z.tar.xz
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner -C /mnt/gentoo

#let's set some make stuff in /etc/portage/make.conf
cat <<'EOF' >  /etc/portage/make.conf
# Please consult /usr/share/portage/config/make.conf.example for a more
# detailed example.
COMMON_FLAGS="-march=native -O2 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"
RUSTFLAGS="${RUSTFLAGS} -C target-cpu=native"
#conf for 8c/16t with 32 GB RAM
MAKEOPTS="-j8 -l17"

# NOTE: This stage was built with the bindist USE flag enabled

# This sets the language of build output to English.
# Please keep this setting intact when reporting bugs.
LC_MESSAGES=C.UTF-8

GENTOO_MIRRORS="https://eu.mirror.ionos.com/linux/distributions/gentoo/gentoo/"

EOF

#let's get the DNS config into the chroot before we start it
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
exit

#time to chroot into our virgin system
sudo arch-chroot /mnt/gentoo
source /etc/profile
export PS1="(chroot) ${PS1}"

#let's check our mounts
lsblk -o NAME,FSTYPE,UUID,PARTUUID,RO,RM,SIZE,STATE,OWNER,GROUP,MODE,TYPE,MOUNTPOINT,LABEL,MODEL
NAME              FSTYPE      UUID                                   PARTUUID                             RO RM   SIZE STATE   OWNER GROUP MODE       TYPE  MOUNTPOINT LABEL                MODEL
vda                                                                                                        0  0   200G         root  disk  brw-rw---- disk
|-vda1            vfat        958B-3DE5                              1aa197e0-0a87-4009-ad58-f33b256fd4f6  0  0   1.9G         root  disk  brw-rw---- part  /boot      EFI
`-vda2            crypto_LUKS 1d643b2b-6093-4029-9add-abf842013588   fd4d1ed0-fb8b-4c60-aef2-782f12cda54c  0  0 198.1G         root  disk  brw-rw---- part
  `-crypt         LVM2_member XUAO8R-EIye-BbyP-ohiz-Hpv0-wl50-rSJdc5                                       0  0 198.1G running root  disk  brw-rw---- crypt
    |-system-swap swap        292abd12-ee02-4b33-abe0-674f59711190                                         0  0    40G running root  disk  brw-rw---- lvm              swapfs
    `-system-root btrfs       4881a6ba-fb34-4102-9bde-3a6f0ab34eae                                         0  0 158.1G running root  disk  brw-rw---- lvm   /home      rootfs


#bring everything up to date
#Get it up to date
emerge-webrsync

#Optional: change the gentoo mirror
#emerge --ask --verbose --oneshot app-portage/mirrorselect
#mirrorselect -i -o >> /etc/portage/make.conf

mkdir /etc/portage/repos.conf
cp /usr/share/portage/config/repos.conf /etc/portage/repos.conf/gentoo.conf

#edit sync-uri = in /etc/portage/repos.conf/gentoo.conf
sync-uri = rsync://eu.mirror.ionos.com/gentoo-portage
emerge --sync
eselect news list
eselect news read
#let's select the system profile
eselect profile list
eselect profile set 7

#we want a binary package host that allows for x86-64-3

mkdir -p /etc/portage/binrepos.conf
cat << 'EOF' > /etc/portage/binrepos.conf/gentoo.conf
[gentoo]
priority = 9959
# NOTE: Must adjust <arch> and <variant> as appropriate!
# sync-uri = https://distfiles.gentoo.org/releases/<arch>/binpackages/<variant>
# x86-64 example sync-uri
sync-uri = https://eu.mirror.ionos.com//linux/distributions/gentoo/gentoo/releases/amd64/binpackages/23.0/x86-64/

# Introduced in portage-3.0.74 for per-repo verification choices
verify-signature = true
# Default value with >=portage-3.0.77
location = /var/cache/binhost/gentoo

[gentoo-x86-64-v3]
priority = 9999
sync-uri = https://eu.mirror.ionos.com//linux/distributions/gentoo/gentoo/releases/amd64/binpackages/23.0/x86-64-v3/
# Introduced in portage-3.0.74 for per-repo verification choices
verify-signature = true
# Default value with >=portage-3.0.77
location = /var/cache/binhost/gentoo-x86-64-v3
EOF


#tell portage that we want binary packages

cat <<EOF >> /etc/portage/make.conf
# Appending getbinpkg to the list of values within the FEATURES variable
EMERGE_DEFAULT_OPTS="${EMERGE_DEFAULT_OPTS} --getbinpkg"
BINPKG_FORMAT="gpkg"
EOF

#setup keyrings
getuto

emerge --ask --oneshot app-portage/cpuid2cpuflags
echo "*/* $(cpuid2cpuflags)" > /etc/portage/package.use/00cpu-flags

#Make sure that the system uses the right GPU (virgl, d3d12, amdgpu radeonsi, intel, nouveau, nvidia)
echo "*/* VIDEO_CARDS: -* virgl" > /etc/portage/package.use/00video_cards

echo 'ACCEPT_LICENSE="-* @FREE @BINARY-REDISTRIBUTABLE"' >> /etc/portage/make.conf

#Updating the @world set
#use binary packages
emerge --ask --verbose --update --deep --newuse --getbinpkg @world

#or build from source
emerge --ask --verbose --update --deep --changed-use --getbinpkg=n @world

emerge --ask --pretend --depclean
emerge --ask --depclean

#Set root password
passwd root
#Create main user
useradd -m -G wheel,kvm,users,audio -s /bin/bash uwe
passwd uwe

#install and configure doas
emerge --ask app-admin/doas
echo "permit persist :wheel" > /etc/doas.conf
chown -c root:root /etc/doas.conf
chmod -c 0400 /etc/doas.conf

#Set hostname
echo gentoodiablo > /etc/hostname

#Let's set the Timezone
ln -sf ../usr/share/zoneinfo/Europe/Berlin /etc/localtime

#Configure locales
cat << 'EOF' > /etc/locale.gen
en_US.UTF-8 UTF-8
de_DE.UTF-8 UTF-8
EOF

locale-gen
eselect locale list
eselect locale set 4
env-update && source /etc/profile && export PS1="(chroot) ${PS1}"

cat << 'EOF' > /etc/conf.d/keymaps
# Use keymap to specify the default console keymap.  There is a complete tree
# of keymaps in /usr/share/keymaps to choose from.
keymap="de-latin1-nodeadkeys"

# Should we first load the 'windowkeys' console keymap?  Most x86 users will
# say "yes" here.  Note that non-x86 users should leave it as "no".
# Loading this keymap will enable VT switching (like ALT+Left/Right)
# using the special windows keys on the linux console.
windowkeys="YES"

# The maps to load for extended keyboards.  Most users will leave this as is.
extended_keymaps=""
#extended_keymaps="backspace keypad euro2"

# Tell dumpkeys(1) to interpret character action codes to be
# from the specified character set.
# This only matters if you set unicode="yes" in /etc/rc.conf.
# For a list of valid sets, run `dumpkeys --help`
dumpkeys_charset=""

# Some fonts map AltGr-E to the currency symbol instead of the Euro.
# To fix this, set to "yes"
fix_euro="NO"
EOF

#Generate Machine ID
dbus-uuidgen --ensure=/etc/machine-id

#install NetworkManager

echo 'USE="${USE} networkmanager"' >> /etc/portage/make.conf

emerge --ask net-misc/networkmanager
rc-update add NetworkManager default

#syslog
emerge --ask app-admin/syslog-ng
emerge --ask app-admin/logrotate
rc-update add syslog-ng default

#Cron
emerge --ask sys-process/cronie
rc-update add cronie default

#NTP
emerge --ask net-misc/chrony
rc-update add chronyd default

#File indexing
emerge --ask sys-apps/mlocate

#enable sshd
rc-update add sshd default

#bash
emerge --ask app-shells/bash-completion

emerge --ask --getbinpkg=n sys-fs/cryptsetup
emerge --ask sys-fs/btrfs-progs
emerge --ask sys-fs/dosfstools
emerge --ask sys-fs/xfsprogs
emerge --ask sys-fs/e2fsprogs
emerge --ask sys-fs/ntfs3g
emerge --ask --getbinpkg=n sys-fs/zfs
emerge --ask sys-fs/f2fs-tools
emerge --ask sys-fs/mdadm
emerge --ask --getbinpkg=n dev-python/zstandard
emerge --ask sys-block/io-scheduler-udev-rules
emerge --ask app-portage/gentoolkit

cat << 'EOF' >> /etc/portage/package.use/lvm2
#Enable support for the LVM daemon and related tools
sys-fs/lvm2 lvm
EOF

emerge --ask sys-fs/lvm2
rc-update add lvm boot

#Let's build a fstab
emerge --ask sys-fs/genfstab
genfstab -U / >> /etc/fstab

echo "crypt /dev/vda2 none luks,discard" > /etc/crypttab
#better use the UUID=
echo "crypt UUID=1d643b2b-6093-4029-9add-abf842013588   none    luks,discard" > /etc/crypttab


emerge --ask sys-kernel/linux-firmware
emerge --ask sys-firmware/sof-firmware

cat << 'EOF' >> /etc/portage/package.use/installkernel
sys-kernel/installkernel -dracut ugrd grub
EOF

echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf

mkdir -p /etc/kernel
echo "quiet splash" > /etc/kernel/cmdline
mkdir -p /etc/cmdline.d
ln -s /etc/kernel/cmdline /etc/cmdline.d/00-installkernel.conf

#make sure that external kmods get auto rebuild
echo 'USE="${USE} dist-kernel"' >> /etc/portage/make.conf

#we want to build our own kernel
emerge --ask sys-kernel/gentoo-kernel

#Just use the gentoo default kernel binary
emerge --ask sys-kernel/gentoo-kernel-bin


#Once configured, installkernel can be re-emerged and will pull UGRD
emerge -ask sys-kernel/installkernel

#To force an initramfs rebuild, emerge --config can be used on dist-kernel packages:
emerge --config sys-kernel/gentoo-kernel
#or
emerge --config sys-kernel/gentoo-kernel-bin

emerge --ask --getbinpkg=n sys-boot/grub
grub-install --efi-directory=/boot
grub-mkconfig -o /boot/grub/grub.cfg
emerge --ask --getbinpkg=n sys-kernel/installkernel[grub]

#let's add some software
echo 'USE="${USE} elogind -systemd"' >> /etc/portage/make.conf
echo 'USE="${USE} lzma zstd curl"' >> /etc/portage/make.conf
emerge --ask --getbinpkg=n dev-vcs/git
emerge --ask --getbinpkg=n sys-process/btop
emerge --ask --getbinpkg=n app-misc/fastfetch
emerge --ask --getbinpkg=n app-misc/tmux

#KDE
emerge --ask kde-plasma/plasma-meta kde-apps/kde-apps-meta
emerge --ask gui-libs/display-manager-init

#world update
emerge --ask --verbose --update --deep --changed-use --getbinpkg=n @world
rc-update add elogind boot
emerge --ask x11-misc/sddm
usermod -a -G video sddm
echo "sys-auth/seatd server" > /etc/portage/package.use/seatd
emerge --oneshot sys-auth/seatd
rc-update add seatd default
rc-service seatd start
usermod -aG seat sddm
usermod -aG seat uwe

cat << 'EOF' > /etc/conf.d/display-manager
CHECKVT=7
DISPLAYMANAGER="sddm"
EOF

#do some cleanup
eclean-dist
eclean-pkg
rm /stage3-*.tar.*

#we're done with the chroot
exit
rc-service elogind start
rc-update add display-manager default
rc-service display-manager start
emerge --ask kde-plasma/sddm-kcm

cat << 'EOF' > /etc/sddm.conf
[Users]
MaximumUid=60000
MinimumUid=1000
EOF
#reboot time
sudo reboot

Note: because someone asked about the VM config, here you go
---
```
```
<domain type="kvm">
  <name>Gentoo2</name>
  <uuid>d33ee995-4d99-4763-a888-e4f0166dc495</uuid>
  <metadata>
    <libosinfo:libosinfo xmlns:libosinfo="http://libosinfo.org/xmlns/libvirt/domain/1.0">
      <libosinfo:os id="http://almalinux.org/almalinux/10"/>
    </libosinfo:libosinfo>
  </metadata>
  <memory unit="KiB">16777216</memory>
  <currentMemory unit="KiB">16777216</currentMemory>
  <memoryBacking>
    <source type="memfd"/>
    <access mode="shared"/>
  </memoryBacking>
  <vcpu placement="static">16</vcpu>
  <os firmware="efi">
    <type arch="x86_64" machine="pc-q35-10.0">hvm</type>
    <firmware>
      <feature enabled="no" name="enrolled-keys"/>
      <feature enabled="no" name="secure-boot"/>
    </firmware>
    <loader readonly="yes" type="pflash" format="raw">/usr/share/OVMF/OVMF_CODE_4M.fd</loader>
    <nvram template="/usr/share/OVMF/OVMF_VARS_4M.fd" templateFormat="raw" format="raw">/var/lib/libvirt/qemu/nvram/Gentoo2_VARS.fd</nvram>
    <bootmenu enable="yes"/>
  </os>
  <features>
    <acpi/>
    <apic/>
    <vmport state="off"/>
  </features>
  <cpu mode="host-passthrough" check="none" migratable="on"/>
  <clock offset="utc">
    <timer name="rtc" tickpolicy="catchup"/>
    <timer name="pit" tickpolicy="delay"/>
    <timer name="hpet" present="no"/>
  </clock>
  <on_poweroff>destroy</on_poweroff>
  <on_reboot>restart</on_reboot>
  <on_crash>destroy</on_crash>
  <pm>
    <suspend-to-mem enabled="no"/>
    <suspend-to-disk enabled="no"/>
  </pm>
  <devices>
    <emulator>/usr/bin/qemu-system-x86_64</emulator>
    <disk type="file" device="cdrom">
      <driver name="qemu" type="raw"/>
      <source file="/srv/mergerfs/pool1/data/Workspace/ISOs/livegui-amd64-20260510T170106Z.iso"/>
      <target dev="sda" bus="sata"/>
      <readonly/>
      <boot order="1"/>
      <address type="drive" controller="0" bus="0" target="0" unit="0"/>
    </disk>
    <disk type="block" device="disk">
      <driver name="qemu" type="raw" cache="directsync" io="native" discard="unmap"/>
      <source dev="/dev/VMs/gentoo_delme"/>
      <target dev="vda" bus="virtio"/>
      <boot order="2"/>
      <address type="pci" domain="0x0000" bus="0x07" slot="0x00" function="0x0"/>
    </disk>
    <controller type="usb" index="0" model="qemu-xhci" ports="15">
      <address type="pci" domain="0x0000" bus="0x02" slot="0x00" function="0x0"/>
    </controller>
    <controller type="pci" index="0" model="pcie-root"/>
    <controller type="pci" index="1" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="1" port="0x10"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x0" multifunction="on"/>
    </controller>
    <controller type="pci" index="2" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="2" port="0x11"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x1"/>
    </controller>
    <controller type="pci" index="3" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="3" port="0x12"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x2"/>
    </controller>
    <controller type="pci" index="4" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="4" port="0x13"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x3"/>
    </controller>
    <controller type="pci" index="5" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="5" port="0x14"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x4"/>
    </controller>
    <controller type="pci" index="6" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="6" port="0x15"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x5"/>
    </controller>
    <controller type="pci" index="7" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="7" port="0x16"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x6"/>
    </controller>
    <controller type="pci" index="8" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="8" port="0x17"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x02" function="0x7"/>
    </controller>
    <controller type="pci" index="9" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="9" port="0x18"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x0" multifunction="on"/>
    </controller>
    <controller type="pci" index="10" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="10" port="0x19"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x1"/>
    </controller>
    <controller type="pci" index="11" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="11" port="0x1a"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x2"/>
    </controller>
    <controller type="pci" index="12" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="12" port="0x1b"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x3"/>
    </controller>
    <controller type="pci" index="13" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="13" port="0x1c"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x4"/>
    </controller>
    <controller type="pci" index="14" model="pcie-root-port">
      <model name="pcie-root-port"/>
      <target chassis="14" port="0x1d"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x03" function="0x5"/>
    </controller>
    <controller type="sata" index="0">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1f" function="0x2"/>
    </controller>
    <controller type="virtio-serial" index="0">
      <address type="pci" domain="0x0000" bus="0x03" slot="0x00" function="0x0"/>
    </controller>
    <controller type="scsi" index="0" model="virtio-scsi">
      <address type="pci" domain="0x0000" bus="0x04" slot="0x00" function="0x0"/>
    </controller>
    <interface type="network">
      <mac address="52:54:00:55:68:b9"/>
      <source network="bridged-network"/>
      <model type="virtio"/>
      <address type="pci" domain="0x0000" bus="0x01" slot="0x00" function="0x0"/>
    </interface>
    <serial type="pty">
      <target type="isa-serial" port="0">
        <model name="isa-serial"/>
      </target>
    </serial>
    <console type="pty">
      <target type="serial" port="0"/>
    </console>
    <channel type="unix">
      <target type="virtio" name="org.qemu.guest_agent.0"/>
      <address type="virtio-serial" controller="0" bus="0" port="1"/>
    </channel>
    <channel type="spicevmc">
      <target type="virtio" name="com.redhat.spice.0"/>
      <address type="virtio-serial" controller="0" bus="0" port="2"/>
    </channel>
    <input type="tablet" bus="usb">
      <address type="usb" bus="0" port="1"/>
    </input>
    <input type="mouse" bus="ps2"/>
    <input type="keyboard" bus="ps2"/>
    <tpm model="tpm-crb">
      <backend type="emulator" version="2.0"/>
    </tpm>
    <graphics type="spice" autoport="yes">
      <listen type="address"/>
    </graphics>
    <sound model="ich9">
      <address type="pci" domain="0x0000" bus="0x00" slot="0x1b" function="0x0"/>
    </sound>
    <audio id="1" type="spice"/>
    <video>
      <model type="virtio" heads="1" primary="yes"/>
      <address type="pci" domain="0x0000" bus="0x00" slot="0x01" function="0x0"/>
    </video>
    <redirdev bus="usb" type="spicevmc">
      <address type="usb" bus="0" port="2"/>
    </redirdev>
    <redirdev bus="usb" type="spicevmc">
      <address type="usb" bus="0" port="3"/>
    </redirdev>
    <watchdog model="itco" action="reset"/>
    <memballoon model="virtio">
      <address type="pci" domain="0x0000" bus="0x05" slot="0x00" function="0x0"/>
    </memballoon>
    <rng model="virtio">
      <backend model="random">/dev/urandom</backend>
      <address type="pci" domain="0x0000" bus="0x06" slot="0x00" function="0x0"/>
    </rng>
  </devices>
</domain>
```
```
---
```

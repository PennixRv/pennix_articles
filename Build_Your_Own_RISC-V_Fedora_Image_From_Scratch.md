# Build Your Own RISC-V Fedora Image From Scratch

Fedora is one of the most popular Linux distributions: it is sponsored by Red Hat, but its development is community-driven. While the default version of Fedora ships with the GNOME desktop environment (it is probably the ideal choice if you want to use a vanilla version of the latter), there are many alternative spins available, which allows us to try a variety of desktop environments such as XFCE or KDE Plasma. In few easy steps it is even possible to build and try a custom Fedora live image.

In this tutorial we see how to create a custom live image of Fedora using lorax, mock and a kickstart file.

**In this tutorial you will learn:**

- How to prepare the mock build environment
- How to create a custom live image using a kickstart file
- How to build a custom image using lorax

| Category    | Requirements, Conventions or Software Version Used           |
| :---------- | :----------------------------------------------------------- |
| System      | Fedora                                                       |
| Software    | mock, lorax, git, pykickstart and a text editor              |
| Other       | Root privileges, being familiar with Kickstart               |
| Conventions | # – requires given [linux-commands](https://linuxconfig.org/linux-commands) to be executed with root privileges either directly as a root user or by use of `sudo` command $ – requires given [linux-commands](https://linuxconfig.org/linux-commands) to be executed as a regular non-privileged user |

## Installing mock and setting up the build environment

In order to create our custom Fedora live image, the first thing we need to do is to install mock, a program which is used to build source RPMs inside a chroot environment:

```shell
sudo dnf install mock
```

In order to use mock without becoming root, we must add our own user to the “mock” group. We can do that using the `usermod` utility (in this case “pennix” is my username):

```shell
sudo usermod -aG mock doc
```

To make the change effective we need to logout and login again, or, as a temporary alternative (will be effective only in the current shell), we could use the `newgrp` utility:

```
$ newgrp mock
```

To verify changes have been applied:

```
$ groups
```

The “mock” group should be reported in the command output:

```
mock wheel doc
```

At this point we can initialize the build environment:

```
$ mock -r /etc/mock/fedora-38-x86_64.cfg --init
```

The `-r` option takes a chroot configuration as argument. Configurations for a choice of systems are distributed together with the mock package, and can be found under the `/etc/mock` directory. In this case, since we want to base the live image on the latest Fedora release (38 at the moment of writing), we pointed mock to the “fedora-38-x86_64.cfg” file. The command will take a while to finish. Once the mock environment is initialized we can install the packages we need inside of it. To do that we use the `--install` option:

```
$ mock -r fedora-38-x86_64 --install lorax anaconda git pykickstart vim
```

Git is needed to clone the repository containing the Fedora [Kickstart](https://linuxconfig.org/how-to-perform-unattended-linux-installations-with-kickstart) file we will use as a base for our custom configuration; the “pykickstart” package contains the “ksflatten” utility which is used to “flatten” kickstart files (produce a single output following all the `%include` instructions in kickstart “source” files). Vim is what we will use to write the custom configuration (any editor will do, of course). These packages are not strictly required inside the chroot, since these steps can be performed in the host system and the final kickstart file can be copied inside the chroot later. In this tutorial, however, we will perform all the work inside the chroot. To enter the build environment, we run:

```
$ mock -r fedora-38-x86_64 --shell --isolation=simple --enable-network
```

A modified the shell prompt should confirm we entered the chroot environment:

```
<mock-chroot> sh-5.2#
```

## Creating the kickstart file

At this point, we need to clone the repository containing the kickstart files used by the Fedora project to build the official distribution images:

```bash
<mock-chroot> sh-5.2# git clone https://pagure.io/fedora-kickstarts -b f38
```

With the command above we clone the repository in a directory called fedora-kickstart. By using -b f38  we specify we want to clone the “f38” branch of the repository, since we are dealing with that versions of Fedora (the “master” branch always points to “rawhide”, the testbed for new Fedora releases). What we need to do now, is to create a kickstart file containing the instructions which will produce the custom live image. For the sake of this tutorial we will create a stripped-down version of Fedora Workstation, excluding a bunch of GNOME applications from the default package set. The most convenient thing to do is to start by including the official fedora-live-workstation.ks kickstart file into our own. Here is its content, we can save it as “fedora-live-minimal-workstation.ks”:

```
%include fedora-live-workstation.ks

%packages
# Package groups excluded from @workstation-product-environment
-@guest-desktop-agents
-@libreoffice
-@multimedia
# Packages excluded from @workstation-product
-rhythmbox
-unoconv
# Packages excluded from @gnome-desktop
-gnome-boxes
-gnome-connections
-gnome-text-editor
-baobab
-cheese
-gnome-clocks
-gnome-logs
-gnome-maps
-gnome-photos
-gnome-remote-desktop
-gnome-weather
-orca
-rygel
-totem
%end
```

In order to obtain a valid output, we use the ksflatten utility. We pass our kickstart file as argument to the -c option, and the output file as argument of -o:

```bash
$ ksflatten -c fedora-live-minimal-workstation.ks -o ks.cfg
```

Once our kickstart file is ready, we can build the live image.

## Creating the live image

At this point we can create our custom live image:

```bash
livemedia-creator --ks ks.cfg --no-virt --resultdir /var/lmc --project Fedora-minimal-workstation-Live --make-iso --volid Fedora-minimal-workstation-Live --iso-only --iso-name Fedora-minimal-workstation-Live.iso --releasever 38 --macboot
```

Let’s take a look at the options we used:

- –ks: is used to point to the kickstart configuration file used to build the live system
- –novirt: used to specify we don’t want to run the anaconda installer into a virtualized qemu environment, but on the host
- –resultdir: specifies the directory where to create the image
- –project: the argument passed to this option is what will replace the @PRODUCT@ placeholder in the bootloader configuration files
- –make-iso: specifies we want to build a live ISO
- –volid: specifies the volume id
- –iso-only: clean all creation artifacts
- –iso-name: the resulting image will be renamed after the argument passed to this option
- –releasever: the argument passed to this option will substitute the @VERSION@ placeholder in the bootloader configuration files
- –macboot: needed to make the iso bootable on UEFI based Mac systems

At the end of the process we should find the `Fedora-minimal-workstation-Live.iso` file under the `/var/lmc` directory. To copy the image from the chroot directory to our home, first we exit the chroot environment:

```bash
<mock-chroot> sh-5.2# exit
```

Than, we run:

```bash
cp /var/lib/mock/fedora-38-x86_64/root/var/lmc/Fedora-minimal-workstation-Live.iso "$HOME"
```

Only the packages included in our custom live image will be included if we decide to finalize an actual installation of the system. We can now clean the build environment:

```bash
mock -r fedora-38-x86_64 --clean
```

## Conclusions

There are many spins of Fedora available, each one providing a different working environment. In this tutorial we saw how to create a custom live image using a kickstart file and tools like mock and lorax.

```kickstart
Fedora-Developer-38-20230519.n.0-qemu
# Kickstart file for Fedora RISC-V (riscv64) workstation 38 with XFCE desktop and lightdm

#repo --name="koji-override-0" --baseurl=https://fedora.riscv.rocks/repos-dist/f38/latest/riscv64/

#repo --name="test-fedora-origin" --baseurl=https://fedora.riscv.rocks/repos-dist/f38/latest/riscv64/

install
text
#reboot
lang en_US.UTF-8
keyboard us
# short hostname still allows DHCP to assign domain name
network --bootproto dhcp --device=link --hostname=fedora-riscv
rootpw --plaintext fedora_rocks!
firewall --enabled --ssh
timezone --utc Asia/Shanghai
selinux --disabled
services --enabled=sshd,NetworkManager,chronyd,haveged,lightdm --disabled=lm_sensors,libvirtd,gdm

bootloader --location=none --extlinux

#zerombr
clearpart --all --initlabel --disklabel=msdos
part swap --asprimary --label=swap --size=32
part /boot/efi --fstype=vfat --size=128 --label=EFI
part /boot --fstype=ext4 --size=512 --label=boot
part / --fstype=ext4 --size=12288 --label=rootfs

# Halt the system once configuration has finished.
poweroff

%packages --ignoremissing
@core
@buildsys-build
@base-x
@hardware-support
@rpm-development-tools
@c-development
@development-tools
@xfce-desktop

# missing packages of xfce-desktop
-abrt-desktop
-dnfdragora-updater
-gnome-abrt
-python3-yui


kernel
kernel-core
kernel-devel
kernel-modules
kernel-modules-extra
linux-firmware
opensbi-unstable
extlinux-bootloader
uboot-tools
uboot-images-riscv64
# Remove this in %post
dracut-config-generic
-dracut-config-rescue

openssh
openssh-server
glibc-langpack-en
glibc-static
lsof
nano
openrdate
chrony
systemd-udev
vim-minimal
neovim
screen
hostname
bind-utils
htop
tmux
strace
pciutils
nfs-utils
ethtool
rsync
hdparm
git
tig
mercurial
breezy
moreutils
rpmdevtools
fedpkg
mailx
mutt
patchutils
ninja-build
cmake
extra-cmake-modules
elfutils
gdisk
util-linux
gparted
parted
fpaste
vim-common
hexedit
koji-builder
mc
evemu
lftp
mtr
traceroute
wget
tar
aria2
incron
emacs
vim
neofetch
bash-completion
zsh
tcsh
nvme-cli
pv
dtc
axel
bc
bison
elfutils-devel
flex
m4
net-tools
openssl-devel
perl-devel
perl-generators
pesign
xterm
#fluxbox
elinks
lynx
#awesome
midori
dillo
epiphany
#i3
#sway
pcmanfm
entr
cowsay
ack
the_silver_searcher
tldr
ncdu
colordiff
prettyping
qemu-guest-agent
iptables-services
autoconf
autoconf-archive
automake
gettext
nnn
gdb
libtool
texinfo
policycoreutils
policycoreutils-python-utils
setools-console
coreutils
setroubleshoot-server
audit
selinux-policy
selinux-policy-targeted
execstack
stress-ng
python3-pyelftools
inxi
# Below packages are needed for creating disk images via koji-builder
livecd-tools
python-imgcreate-sysdeps
python3-imgcreate
python3-pyparted
isomd5sum
python3-isomd5sum
pykickstart
python3-kickstart
python3-ordered-set
appliance-tools
pycdio
qemu-img
nbdkit
nbd
# end of creating disk image packages list
dosfstools
btrfs-progs
e2fsprogs
f2fs-tools
jfsutils
mtd-utils
ntfsprogs
udftools
xfsprogs
kpartx
libguestfs-tools-c
rpkg
binwalk
bloaty
bpftool
kernel-tools
perf
python3-perf
libgpiod
libgpiod-c++
libgpiod-devel
libgpiod-utils
python3-libgpiod
i2c-tools
i2c-tools-eepromer
i2c-tools-perl
libi2c
libi2c-devel
python3-i2c-tools
spi-tools
# Add gcc packages
cpp
gcc
gcc-c++
gcc-gdb-plugin
gcc-gfortran
gcc-go
gcc-plugin-devel
libatomic
libatomic-static
libgcc
libgfortran
libgfortran-static
libgo
libgo-devel
libgo-static
libgomp
libstdc++
libstdc++-devel
libstdc++-static
gcc-gdc
libgphobos
libgphobos-static
pax-utils
gcc-gnat
libgnat
libgnat-devel
libgnat-static
usbutils
haveged
# end of gcc packages
# Add dejavu fonts
dejavu-fonts-all
dejavu-lgc-sans-fonts
dejavu-lgc-sans-mono-fonts
dejavu-lgc-serif-fonts
dejavu-sans-fonts
dejavu-sans-mono-fonts
dejavu-serif-fonts
# end of dejavu fonts

-grubby
grubby-deprecated

# No longer in @core since 2018-10, but needed for livesys script
initscripts
chkconfig

# Lets resize / on first boot
#dracut-modules-growroot

dnscrypt-proxy
meson
cloud-utils-growpart
iperf3
sysstat
fio
memtester
fuse-sshfs
zstd
xz
NetworkManager-tui
cheat
ddrescue
glances
python3-psutil

# Add dependencies (BR) for kernel, gcc, gdb, binutils, rpm, util-linux, glibc,
# bash and coreutils
audit-libs-devel
bzip2-devel
dblatex
dbus-devel
dejagnu
docbook5-style-xsl
dwarves
expat-devel
fakechroot
file-devel
gd-devel
gettext-devel
glibc-all-langpacks
ima-evm-utils-devel
isl-devel
libacl-devel
libarchive-devel
libattr-devel
libbabeltrace-devel
libcap-devel
libcap-ng-devel
libdb-devel
libpng-devel
libselinux-devel
libuser-devel
libutempter-devel
libzstd-devel
lua-devel
ncurses-devel
pam-devel
pcre2-devel
perl
popt-devel
python3-devel
python3-langtable
python3-sphinx
readline-devel
rpm-devel
sharutils
source-highlight-devel
systemd-devel
texinfo-tex
texlive-collection-latex
texlive-collection-latexrecommended
zlib-static
# end of dependencies (BR)

dos2unix
fwts
acpica-tools
glib
glib2
dkms
expect
openssl
gnutls-utils
iw
info
wireless-tools
wireless-regdb
jq
sysfsutils
golang
golang-bin
golang-src
bmap-tools
flashrom

glib-devel
glib2-devel
json-c-devel
libbsd-devel
zlib-devel
xz-devel
brotli-devel
libuv-devel
libnghttp2-devel
libicu-devel
libX11-devel
libXtst-devel
libXt-devel
libXrender-devel
libXrandr-devel
libXi-devel
libXext-devel
cups-devel
fontconfig-devel
alsa-lib-devel
freetype-devel
libdwarf-devel
gnulib-devel
xorg-x11-apps
avahi
%end

%post

echo 'nameserver 114.114.114.114' >> /etc/resolv.conf

pushd /tmp

/usr/bin/wget -P boot "http://openkoji.iscas.ac.cn/pub/dl/riscv/Allwinner/Nezha_D1/kernel-cross/kernel-cur.tar"
/usr/bin/wget -P boot "http://openkoji.iscas.ac.cn/pub/dl/riscv/riscv64/grubriscv64.efi"

pushd boot
/usr/bin/tar  xpf kernel-cur.tar
popd

mkdir -p /boot/efi/EFI/fedora
mv -vf boot/grubriscv64.efi /boot/efi/EFI/fedora/
mv -vf boot/lib/modules/* /lib/modules/
mv -vf boot/boot-5.4.img /boot
mv -vf boot/*.dtb /boot
mv -vf boot/uEnv.txt /boot
mv -vf boot/extlinux.conf /boot/extlinux/extlinux.conf
mv -vf boot/*-5.4.61* /boot

rm -r boot

ROOT_UUID=`grep ' / ' /etc/fstab | awk '{printf $1}' | sed -e 's/^UUID=//g'`

sed -i -e "s/@@ROOTUUID@@/${ROOT_UUID}/g"   \
    /boot/extlinux/extlinux.conf

popd

%end

%post
# Disable default repositories (not riscv64 in upstream)
dnf config-manager --set-disabled rawhide updates updates-testing fedora fedora-modular fedora-cisco-openh264 updates-modular updates-testing-modular rawhide-modular

dnf -y remove dracut-config-generic

# systemd on no-SMP boots (i.e. single core) sometimes timeout waiting for storage
# devices. After entering emergency prompt all disk are mounted.
# For more information see:
# https://www.suse.com/support/kb/doc/?id=7018491
# https://www.freedesktop.org/software/systemd/man/systemd.mount.html
# https://github.com/systemd/systemd/issues/3446
# We modify /etc/fstab to give more time for device detection (the problematic part)
# and mounting processes. This should help on systems where boot takes longer.
sed -i 's|noatime|noatime,x-systemd.device-timeout=300s,x-systemd.mount-timeout=300s|g' /etc/fstab

# remove unnecessary entry in /etc/fstab

sed -i '/swap/d' /etc/fstab
sed -i '/efi/d' /etc/fstab

# Fedora 31
# https://fedoraproject.org/wiki/Changes/DisableRootPasswordLoginInSshd
cat > /etc/rc.d/init.d/livesys << EOF
#!/bin/bash
#
# live: Init script for live image
#
# chkconfig: 345 00 99
# description: Init script for live image.
### BEGIN INIT INFO
# X-Start-Before: display-manager chronyd
### END INIT INFO

. /etc/rc.d/init.d/functions

useradd -c "Fedora RISCV User" riscv
echo fedora_rocks! | passwd --stdin riscv > /dev/null
usermod -aG wheel riscv > /dev/null

exit 0
EOF

chmod 755 /etc/rc.d/init.d/livesys
/sbin/restorecon /etc/rc.d/init.d/livesys
/sbin/chkconfig --add livesys

# Create Fedora RISC-V repo
cat << EOF > /etc/yum.repos.d/fedora-riscv.repo
[fedora-riscv]
name=Fedora RISC-V
baseurl=http://fedora.riscv.rocks/repos-dist/rawhide/latest/riscv64/
#baseurl=https://dl.fedoraproject.org/pub/alt/risc-v/repo/fedora/rawhide/latest/riscv64/
#baseurl=https://mirror.math.princeton.edu/pub/alt/risc-v/repo/fedora/rawhide/latest/riscv64/
enabled=1
gpgcheck=0

[fedora-riscv-debuginfo]
name=Fedora RISC-V - Debug
baseurl=http://fedora.riscv.rocks/repos-dist/rawhide/latest/riscv64/debug/
#baseurl=https://dl.fedoraproject.org/pub/alt/risc-v/repo/fedora/rawhide/latest/riscv64/debug/
#baseurl=https://mirror.math.princeton.edu/pub/alt/risc-v/repo/fedora/rawhide/latest/riscv64/debug/
enabled=0
gpgcheck=0

[fedora-riscv-source]
name=Fedora RISC-V - Source
baseurl=http://fedora.riscv.rocks/repos-dist/rawhide/latest/src/
#baseurl=https://dl.fedoraproject.org/pub/alt/risc-v/repo/fedora/rawhide/latest/src/
#baseurl=https://mirror.math.princeton.edu/pub/alt/risc-v/repo/fedora/rawhide/latest/src/
enabled=0
gpgcheck=0
EOF

# Create Fedora RISC-V Koji repo
cat << EOF > /etc/yum.repos.d/fedora-riscv-koji.repo
[fedora-riscv-koji]
name=Fedora RISC-V Koji
baseurl=http://fedora.riscv.rocks/repos/rawhide/latest/riscv64/
enabled=0
gpgcheck=0
EOF

# systemd starts serial consoles on /dev/ttyS0 and /dev/hvc0.  The
# only problem is they are the same serial console.  Mask one.
systemctl mask serial-getty@hvc0.service

# setup login message
cat << EOF | tee /etc/issue /etc/issue.net
Welcome to the Fedora/RISC-V disk image
https://fedoraproject.org/wiki/Architectures/RISC-V

Build date: $(date --utc)

Kernel \r on an \m (\l)

The root password is 'fedora_rocks!'.
root password logins are disabled in SSH starting Fedora 31.
User 'riscv' with password 'fedora_rocks!' in 'wheel' group is provided.

To install new packages use 'dnf install ...'

To upgrade disk image use 'dnf upgrade --best'

If DNS isn’t working, try editing ‘/etc/yum.repos.d/fedora-riscv.repo’.

For updates and latest information read:
https://fedoraproject.org/wiki/Architectures/RISC-V

Fedora/RISC-V
-------------
Koji:               http://fedora.riscv.rocks/koji/
SCM:                http://fedora.riscv.rocks:3000/
Distribution rep.:  http://fedora.riscv.rocks/repos-dist/
Koji internal rep.: http://fedora.riscv.rocks/repos/
EOF

# Remove machine-id on pre generated images
rm -f /etc/machine-id
touch /etc/machine-id

# remove random seed, the newly installed instance should make it's own
rm -f /var/lib/systemd/random-seed

%end


%post --nochroot

/usr/sbin/sfdisk -l /dev/loop0

echo 'Deleting the first partition of Image...'
/usr/sbin/sfdisk --delete /dev/loop0 1
/usr/sbin/sfdisk -l /dev/loop0

echo 'Reorder the partitions of Image...'
/usr/sbin/sfdisk -r /dev/loop0
/usr/sbin/sfdisk -l /dev/loop0

pushd /tmp
/usr/bin/wget -P boot "http://openkoji.iscas.ac.cn/pub/dl/riscv/Allwinner/Nezha_D1/FW/boot0_sdcard_sun20iw1p1.bin"
/usr/bin/wget -P boot "http://openkoji.iscas.ac.cn/pub/dl/riscv/Allwinner/Nezha_D1/FW/boot_package.fex"

dd if=boot/boot0_sdcard_sun20iw1p1.bin of=/dev/loop0 seek=16 bs=512
dd if=boot/boot_package.fex of=/dev/loop0 seek=32800 bs=512

popd

%end

# EOF
```


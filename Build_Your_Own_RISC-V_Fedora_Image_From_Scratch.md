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

```bash
sudo dnf install mock
```

In order to use mock without becoming root, we must add our own user to the “mock” group. We can do that using the `usermod` utility (in this case “pennix” is my username):

```bash
sudo usermod -aG mock doc
```

To make the change effective we need to logout and login again, or, as a temporary alternative (will be effective only in the current shell), we could use the `newgrp` utility:

```bash
newgrp mock
```

To verify changes have been applied:

```bash
groups
```

The “mock” group should be reported in the command output:

```bash
mock wheel doc
```

At this point we can initialize the build environment:

```bash
mock -r /etc/mock/fedora-38-x86_64.cfg --init
```

The `-r` option takes a chroot configuration as argument. Configurations for a choice of systems are distributed together with the mock package, and can be found under the `/etc/mock` directory. In this case, since we want to base the live image on the latest Fedora release (38 at the moment of writing), we pointed mock to the “fedora-38-x86_64.cfg” file. The command will take a while to finish. Once the mock environment is initialized we can install the packages we need inside of it. To do that we use the `--install` option:

```bash
mock -r fedora-38-x86_64 --install lorax anaconda git pykickstart vim
```

Git is needed to clone the repository containing the Fedora [Kickstart](https://linuxconfig.org/how-to-perform-unattended-linux-installations-with-kickstart) file we will use as a base for our custom configuration; the “pykickstart” package contains the “ksflatten” utility which is used to “flatten” kickstart files (produce a single output following all the `%include` instructions in kickstart “source” files). Vim is what we will use to write the custom configuration (any editor will do, of course). These packages are not strictly required inside the chroot, since these steps can be performed in the host system and the final kickstart file can be copied inside the chroot later. In this tutorial, however, we will perform all the work inside the chroot. To enter the build environment, we run:

```bash
mock -r fedora-38-x86_64 --shell --isolation=simple --enable-network
```

A modified the shell prompt should confirm we entered the chroot environment:

```bash
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

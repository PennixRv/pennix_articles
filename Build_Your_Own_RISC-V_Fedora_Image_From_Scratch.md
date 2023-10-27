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



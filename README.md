<p align="center">
   <img alt="Arch Linux Logo" src="image/archlinux-logo.png">
</p>

# Arch Linux Installation Guide

> Author: Gustavo Leite `<contact[at]gustavoleite.me>` | Created: November 2019

![GitHub](https://img.shields.io/github/license/leiteg/arch-linux-install)

This is the script I follow for installing Arch Linux on my own machines. For
that matter, it is heavily skewed towards *my* needs. I do not claim that this
script is the most appropriate for every scenario nor do I hold responsibility
for any mistakes contained here. Use with caution!

A fuller and more in-depth tutorial can be found in the [Arch Linux Wiki][1].
If you would like to suggest improvements or report any errors, I'll be glad to
hear from you! Reach out to me via email or open an issue on [Github][11].

This guide is distributed under the [MIT License](LICENSE). Enjoy! 🤠

## Table of Contents

1. [Pre Installation](#1-pre-installation)
   1. [Keyboard](#11-keyboard)
   2. [Internet](#12-internet)
   3. [System Clock](#13-system-clock)
   4. [Disk Partitioning](#14-disk-partitioning)
   5. [Partition Formatting](#15-partition-formatting)
   6. [Mounting](#16-mounting)
2. [Installation](#2-installation)
3. [Configuration](#3-configuration)
   1. [Filesystem Tab](#31-filesystem-tab)
   2. [Change Root](#32-change-root)
   3. [Timezone](#33-timezone)
   4. [Localization](#34-localization)
   5. [Network](#35-network)
   6. [Bootloader](#36-bootloader)
4. [Post Installation](#4-post-installation)
   1. [User Creation](#41-user-creation)
   2. [Package Manager](#42-package-manager)
   3. [Graphical User Interface](#43-graphical-user-interface)
   4. [Timesyncd](#44-timesyncd)
   5. [Programs](#45-programs)
   6. [Other Things](#46-other-things)

## 1. Pre Installation

Some preparation is needed before effectively installing the operating system,
namely:

1. Set the keyboard layout
2. Connect to the internet
3. Update system clock
4. Partition the disk
5. Format the partitions
6. Mount the partitions

### 1.1. Keyboard

After booting into the Live USB and being prompted by the command line, load
the correct keyboard layout, for example:

```bash
# List all layouts
ls /usr/share/kbd/keymaps/**/*.map.gz

# Load a specific layout
loadkeys br-abnt2
```

### 1.2. Internet

Now, certify that you are connected to the internet. If you are using a cabled
connection you should be good to go, whereas if you use WiFi, you can connect
using `wifi-menu`:

```bash
wifi-menu # Select the network and login
```

It is possible to test the connection by pinging https://archlinux.org.

```bash
ping archlinux.org # You should see the packets arriving
```

### 1.3. System Clock

Enable the *Network Time Protocol* (NTP) so that the time is updated
automatically:

```bash
timedatectl set-ntp true

# You can check the status to see if it is actually active
timedatectl status
```

### 1.4. Disk Partitioning

Next, it is necessary to partition the disk. There are many programs to do this
(`fdisk`, `gdisk`, `cfdisk`, etc) in both command line and text user interfaces
flavors.

You will need at least two partitions: one for the operating system; and
anoother for the boot record. I will assume that your are installing the system
on drive `/dev/sda`, the OS will reside on partition `/dev/sda2` and the EFI
boot partition on `/dev/sda1`. These can vary on your system.

> **Note:** Just to be clear: by deleting an existing partition you will
> essentially loose all their contents. Do no delete partitions that already
> have data.

> **Tip:** If you are starting from a brand new disk, I suggest you put the boot
> partition first. If you eventually decide to resize the primary partition,
> your disk will be less fragmented.

```bash
# If you never done this before, I recommend `cfdisk`
cfdisk /dev/sda
```

Below is a table describing how the partitioning should be made:

   | Partition | Size    | Type             |
   |-----------|---------|------------------|
   | Boot      | 500 MiB | EFI System       |
   | Primary   | Any     | Linux Filesystem |

### 1.5. Partition Formatting

After partitioning the disk, you should format the partitions with `mkfs`. The
boot partition should be formatted as `vfat`. The primary partition, however,
can be formatted as: `ext3`, `ext4`, `xfs`, `jfs`, `btrfs`, and others. If you
are unsure, go with `ext4`.

> **Note:** Just to be clear: formatting a partition will delete everything on
> it.

```bash
# Format the boot partition as `vfat`
mkfs.vfat /dev/sda1

# Format the primary partition as `ext4`
mkfs.ext4 /dev/sda2
```

### 1.6. Mounting

After partitioning, mount the disks. If you have any other partitions that you
would like to use with this system, mount them accordingly.

```bash
mount /dev/sda2 /mnt            # The primary partition
mount /dev/sda1 /mnt/boot/efi   # The boot partition
```

## 2. Installation

Before installing the operating system, it is advised to edit the file
`/etc/pacman.d/mirrorlist` and put geographically close mirrors on the top of
the list so they have higher priority.

```bash
# Place mirrors from your country on top
vim /etc/pacman.d/mirrorlist
```

Now we install the operating system. This will download and install the
essential packages. In place of `vim` you could download an editor of your
preference (`pico`, `nano`, `emacs`, etc).

```bash
pacstrap /mnt base linux linux-firmware vim sudo
#         |   |<---------- packages ---------->|
#         |
#         `--> Where to install (Your primary partition is mounted here)
```

## 3. Configuration

Some things have to be configured before we boot into the new system. They are:

1. Generate `/etc/fstab`
2. Change Root
3. Timezone
4. Localization
5. Network
6. Bootloader

### 3.1. Filesystem Tab

Generate the filesystem tab:

> **Note:** It is advised to use the append operator `>>` here to avoid
> overriding any contents `/etc/fstab` already had.

```bash
genfstab -U -p /mnt >> /mnt/etc/fstab
#         |  |
#         |  `--> Avoid printing pseudofs mounts (default)
#         `--> Use UUID labels for source identifiers
```

### 3.2. Change Root

And finally, `chroot` into the new system. This will change the filesystem root
so that any changes you make (i.e. install packages) are being applied to the
system you just installed and not to the the Live USB.

```bash
arch-chroot /mnt bash
#            |    |
#            |    `--> Which shell to use (`sh` is default)
#            `--> New root
```

### 3.3. Timezone

Adjust your timezone and system clock.

```bash
# Symlink the desired region
ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
#                             |       |
#                             |       `--> Substitute for your city
#                             `--> Substitute for your region

# Set the hardware clock (see note below)
hwclock --systohc
#            |
#            `--> Set the hardware clock from the system clock and update
#                 the timestamps in /etc/adjtime.
```

> **Note:** If you use Windows, this will probably cause the clock to be wrong
> when you log into it. This happens because Linux interprets the hardware clock
> as UTC, while Windows use local time. To fix that, it is suggested to edit
> Windows registry to make it use UTC as well ([See How][2]). Or, you can
> configure Linux to use local time, although ill advised.

### 3.4. Localization

Edit the file `/etc/locale.gen` and uncomment the locales you will need.

```bash
# Select locales
vim /etc/locale.gen

# In my case, I will be using English (US) and Portuguese (BR)
# Uncomment the following lines:
#    en_US.UTF-8 UTF-8
#    pt_BR.UTF-8 UTF-8

# Generate locales
locale-gen

# Set default locale
echo LANG=en_US.UTF-8 > /etc/locale.conf
export LANG=en_US.UTF-8
#               |
#               `--> The locale you set in the above two commands will be the
#                    language of your operating system.
```

### 3.5. Network

Assign your machine a host name:

```bash
echo "YOUR_HOSTNAME" > hostname
```

Then add the following entries at the end of the `/etc/hosts` file:

```
# /etc/hosts
127.0.0.1     localhost
::1           localhost
127.0.1.1     YOUR_HOSTNAME.localdomain YOUR_HOSTNAME
```

### 3.6. Bootloader

Now we need to install the bootloader. This piece of software will run after the
BIOS and will allow you to select which operating system you want to boot.

The packages needed here are:

- `grub-efi-x86_64`: [GRUB bootloader][4]
- `os-prober`: Tool to detect other OSes
- `efibootmgr`: Tool to modify UEFI firmware boot manager variables

```bash
# Install the packages
pacman -S grub-efi-x86_64 os-prober efibootmgr
#  |    |
#  |    `--> Install packages
#  `--> This is the Arch Linux package manager`

# Install GRUB on the EFI partition
grub-install --recheck /dev/sda1 --efi-directory=/boot/EFI
#                 |       |
#                 |       `--> Your boot partition
#                 `--> Check for new drives

# Create a config file for GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

> **Note:** There is an alternative to GRUB called [rEFInd][5] (`refind-efi`).
> Beware that rEFInd is not compatible with some filesystems, read the manual
> before installing it.

## 4. Post Installation

Now your system is ready to roll! But we still need to make it suitable to
working and development. These are the steps I usually go through.

### 4.1. User Creation

Although you can create your user *after* booting into the system, I prefer to
do it here. Do not forget to setup passwords for both root and your user.

```bash
useradd -m -G users YOUR_USERNAME
#        |  |   |
#        |  |   |
#        |  |   `--> Group name
#        |  `--> Which group the user belongs
#        `--> Create a /home folder for this user

passwd                # Create root password
passwd YOUR_USERNAME  # Create user password
```

Also, enable your user to use `sudo`. Edit the file `/etc/sudoers`.

```bash
# Enable your user by inserting a line like this:
#    YOUR_USERNAME ALL=(ALL) ALL
visudo /etc/sudoers
```

### 4.2. Package Manager

Uncomment the block `[multilib]`  in `/etc/pacman.conf` to enable the
installation of 32-bit packages in 64-bit systems.

```bash
# Enable multilib
vim /etc/pacman.conf

# Uncomment the following lines:
#   [multilib]
#   Include = /etc/pacman.d/mirrorlist
```

### 4.3. Graphical User Interface

We still need to install a desktop environment. In this case, I'll be using
[GNOME][6] although there are alternatives. A complete list of desktop
environments is available [here][7].

We will need three packages:

- `wayland`: Display server (alternative: `xorg`)
- `gnome`: Desktop environment and window manager (alternatives: `i3-wm`,
  `xfce4`, `kde`, etc)
- `gdm`: Gnome Display Manager (alternatives: `lightdm`)

```bash
# Install GUI
pacman -S wayland gnome gdm

# Enable desktop manager to start with the system
systemctl enable gdm
```

> **Note:** You can actually install multiple desktop environments and choose
> which one to use in the display manager before logging-in.

### 4.4. Timesyncd

Enable time synchronization daemon. This will keep your system clock always
updated.

```bash
systemctl enable --now systemd-timesyncd.service
#                  |
#                  `--> Enable and start the service in one command
```

### 4.5. Programs

Following is a list of essential packages for my development environment, this
may vary from your needs.

- `openssh`: Secure Shell (SSH)
- `git`: Control version system
- `base-devel`: GNU development tools
- `llvm`, `clang`: C/C++ compiler and libraries
- `intel-ucode`: Microcode update for Intel CPUs (For AMD: `amd-ucode`)
- `coreutils`: GNU file, shell and text utilities
- `strace`: Track system calls
- `ltrace`: Tracks runtime library calls
- `gdb`: GNU Debugger
- `tmux`: Terminal multiplexer
- `python-pip`: Python package manager
- `gvim`: Graphical vim (with clipboard support)
- `zsh`: Z-Shell
- `htop`: Interactive process viewer
- `cmake`: build system

These are also other programs that I always install:

- `firefox`: Web browser
- `networkmanager`, `network-manager-applet`: Network configuration and
  management
- `texlive-most`: LaTeX binaries and packages
- `neofetch`: Tool to display system information

```bash
# Development and administration
pacman -S openssh git base-devel llvm clang intel-ucode coreutils \
          strace ltrace gdb tmux zsh python-pip gvim zsh htop \
          cmake

# User programs
pacman -S firefox networkmanager network-manager-applet \
          texlive-most neofetch

# Drivers for NVIDIA
pacman -S nvidia nvidia-libgl

# Fonts
pacman -S ttf-dejavu ttf-fira-code ttf-fira-sans ttf-hack \
          ttf-liberation ttf-roboto noto-fonts noto-fonts-extra \
          noto-fonts-emoji noto-fonts-cjk

# Enable services
systemctl enable --now networkmanager.service
systemctl enable --now sshd.service

# Change my user's default shell to zsh
chsh -s $(which zsh) YOUR_USERNAME
```

> **Note:** Font rendering in Linux is below average, to say the least. Here are
> [some tips][3] to improve it.

### 4.6. Other Things

This section shows some other things I find useful to do. However, I usually
run them *after* booting natively into the new system (and not from a chroot'ed
system).

```bash
# Generate a pair of SSH keys for this machine
ssh-keygen

# Clone dotfiles
git clone https://github.com/leiteg/dotfiles ~/dotfiles

# Install Python packages
pip install yapf jupyter
```

Besides the official Arch Linux repositories, there is the [Arch User
Repository][8] (AUR) which is maintained by users and provides even more
packages.  However, to download packages from AUR, you need to install a
wrapper for `pacman`. There are several in the market, but I would recommend
[yay][9].

## Before you go...

Now your OS is installed and ready! Run `exit` to quit the `arch-root` and then
`reboot`. You should be greeted with the GRUB screen. You can look for further
resources in the [general recommendations][10] page in the Arch Wiki.

[1]: https://wiki.archlinux.org/index.php/Installation_Guide
[2]: https://www.howtogeek.com/323390/how-to-fix-windows-and-linux-showing-different-times-when-dual-booting/
[3]: https://www.reddit.com/r/archlinux/comments/5r5ep8/make_your_arch_fonts_beautiful_easily/
[4]: https://wiki.archlinux.org/index.php/GRUB
[5]: https://wiki.archlinux.org/index.php/rEFInd
[6]: https://wiki.archlinux.org/index.php/GNOME
[7]: https://wiki.archlinux.org/index.php/Desktop_Environment
[8]: https://wiki.archlinux.org/index.php/Arch_User_Repository
[9]: https://github.com/jguer/yay
[10]: https://wiki.archlinux.org/index.php/General_Recommendations
[11]: https://github.com/leiteg/arch-linux-install/issues

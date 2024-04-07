.. title: Repairing a Botched Arch Linux Kernel Update
.. slug: arch-crash-recovery-chroot
.. date: 2024-04-06 22:00:00 UTC-04:00
.. tags: linux
.. description: In which I repair my Arch Linux system after a crash during a kernel update.
.. type: text

I made the jump to Arch Linux on my personal and work computers a few years
ago. Ubuntu was a dependable experience, but a combination of issues with the
Debian way of life (slow progress) and Canonical's stewardship of the project
(e.g., Snaps) convinced me it was time to try another distribution.

It's been a great experience. Pacman is a wonderful package manager, and AUR is
everything I wanted PPAs to be. I get the latest versions of packages
(including the Linux kernel) shortly after they're released. Of course, that
isn't _always_ a good thing, but rolling back to an older version is
straightforward.

I recently stubbed my toe in a way that's unique to Arch - my computer crashed
while I was in the middle of upgrading my kernel. What's great is I was able to
recover with only an hour or so of downtime. What happened?

<!-- TEASER_END -->

So, I took an update:

```sh
# Update installed packages using the `pacman` utility.
# -S: Synchronize (install) a package to the local machine.
# -y: Refresh the database of known-available packages.
# -u: Upgrade all installed packages that are out of date.
sudo pacman -Syu
```

This kicked off a download of the Linux kernel, headers, and Nvidia drivers
(don't judge, this is a work machine that was the only thing that could be
shipped in a few days during the height of pandemic-induced supply chain
disruption). The download and installation succeeded - all that was left were
the post-update hooks, which build modules for the newly installed kernels and
package everything nicely for your bootloader, which also gets updated to be
made aware of the new kernels. However, disaster befell me. Halfway through the
application of post-update hooks, my screen froze and the caps lock light
started flashing. Kernel panic. The only way out of this was to hard-restart my
computer.

I pushed the power button and crossed all my fingers. I got to my bootloader
and selected the kernel I wished to run. I was immediately greeted with an
error saying that my kernel could not be loaded. I tried the other listed
kernel and was similarly unlucky. My data was fine, but my kernel wasn't. Time
to get to recovering.

Thankfully I already had a USB flash drive with a live image of my Arch
distribution. I plugged it into my machine and booted into it. Live CDs (or
Live Images) are in-memory versions of a Linux distribution that are great for
kicking the tires on a distro, or, in my case, recovery. From there I opened a
terminal.

First, I needed to mount my disk so I could perform surgery. My situation was
complicated by the fact that my disk was encrypted. I'd first need to create a
logical volume of the decrypted disk, and then mount that to the filesystem.

```sh
# List block devices.
# -f: Include information about filesystems
$ lsblk -f
nvme0n1
├─nvme0n1p1
│        vfat   FAT32 NO_LABEL       AB24-0292
└─nvme0n1p2
         crypto 1                    7bb11b10-3b40-475d-ba3e-2be64987d859
```

See that `crypto` line? It's got nothing to do with a great way to destroy the
planet and defraud the technologically unaware - that means the partition
`nvme0n1p2` is encrypted. That's my disk.

Now, I need to get it mounted as a logical volume.

```sh
# Use dm-crypt to map the encrypted block device as a logical volume.
# -v: Run this command verbosely, with a more detailed output.
# luksOpen: LUKS is the encryption format I'm using. Open it as such.
# /dev/nvme0n1p2: The partition to map.
# cryptDrive: The name of the volume the device will be mapped to.
sudo cryptsetup -v luksOpen /dev/nvme0n1p2 cryptDrive
```

This prompted me for the password I use for the encryption key. Inputting that
resulted in a successful map operation. My volume was at
`/dev/mapper/cryptDrive`, and I was one step closer to resolution.

Next, I needed to touch the filesystem. This is as straightforward as mounting
the mapped volume.

```sh
# Mount the volume to a path.
# /dev/mapper/cryptDrive: The decrypted disk partition.
# /mnt: The local path to which that disk should be mounted.
sudo mount /dev/mapper/cryptDrive /mnt
```

Another success. Let's verify I can now read the contents of my disk:

```sh
# List the contents of a path.
$ ls /mnt/
bin    dev/  etc/   lib    mnt/  proc/  run/  srv/      sys/  usr/
boot/  efi/  home/  lib64  opt/  root/  sbin  storage/  tmp/  var/
```

Yep, that's me!

Now for the magic. This is the whole reason I'm using a Live CD for my
distribution. I'm going to tell the system "Forget about what you know about
your in-memory root disk. This is the root disk, now."

```sh
# Change the root directory of the command.
# /mnt: The new root directory.
# /bin/bash: The command to run with knowledge of the new root directory.
sudo arch-chroot /mnt /bin/bash
```

_(note: if using a distribution like Manjaro, this would be `manjaro-chroot`)_

And with that, I'm now logged in as `root` into a shell specially configured to
think that my encrypted disk is the disk it's booted off, with some helpful
standard Arch tools at hand.

First, let's try reinstalling the kernel.

```sh
# Reinstall the Linux 6.6 Kernel
$ pacman -S linux66
ldconfig: File /usr/lib/[FILE].so is empty, not checked.
...that repeated for many different files...
error: failed to prepare transaction (invalid or corrupted package)
```

Well, crap. It's hosed. I repeated this for a few other packages I knew were
being upgraded (reminder: always read the output before you accept an upgrade)
and discovered that they were in similar states of disrepair.

Lets do this at scale with all installed packages.

```sh
# Attempt to reinstall all known packages.
# -S: Synchronize (install) a package to the local machine.
# $(...): This is a way to pass the output of one command to another.
# -Q: Query the package database.
# -n: Limit the packages queried to only those that are tracked in the Pacman database.
# -q: Show less information about packages than usual.
pacman -S $(pacman -Qnq)
```

Because we know there to be broken packages, the command will ultimately exit
with an error. What we're after is the complete list of broken packages. Take
note of the names of any packages with accompanying error loglines.

Now, let's fix them.

```sh
# Force installation of the known bad packages.
# -S: Synchronize (install) a package to the local machine.
# --overwrite "*": ignore conflicting files and overwrite all.
# Replace 'package1 package2' with a space-delimited list of broken packages.
pacman -S --overwrite "*" package1 package2
```

This should knock those packages back in line. Let's run a full system upgrade
and force reinstall our kernels to verify it's all good.

```sh
pacman -Syu
# Reinstall all installed kernels.
reinstall-kernels
```

No errors! I crossed my fingers, rebooted my computer... and was able to log in
as expected!

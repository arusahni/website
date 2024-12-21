.. title: Installing a Release Candidate Kernel in Arch Linux
.. slug: arch-rc-kernel
.. date: 2024-12-19 20:33:03 UTC-05:00
.. tags: linux
.. description: In which I install a release candidate Linux kernel to be able to listen to tunes again.
.. type: text

I recently received a shiny new Thinkpad for my shiny new job. I had the option
to select what I wanted from Lenovo (or Apple, but I dislike MacOS) and chose a
machine that wouldn't have the same issues [my other workstation
had](link://slug/arch-crash-recovery-chroot). I made it a point to avoid Intel
chipsets (due to webcam issues) and stayed far away from discrete Nvidia
graphics. I settled on a nice AMD Ryzen-based system. It had a very new
Qualcomm radio, but I saw people online saying that Wi-Fi and Bluetooth worked
fine. I took the plunge and selected that as my workstation.

It arrived, I popped in my live USB, and a few hours later, I had a fully
provisioned Arch Linux system. Wi-Fi worked fine, and I was able to scan for
Bluetooth devices. Success! Or was it?

The next morning, I sat down at my desk, clocked in, and grabbed my Bluetooth
headphones to listen to some morning tunes. I initiated pairing from the
headphones, selected them from my laptop's device list, and... nothing. It
timed out. I tried again, and again, and again, messing with timeouts and using
`bluetoothctl` to manually negotiate things, but no luck.

<!-- TEASER_END -->

I continued my digging. First, I validated that the computer could not pair
with another set of Bluetooth headphones. This let me isolate the problem to
the system. I then started comparing the list of Bluetooth-related packages
against a computer with which the headphones worked — no differences. I did the
same with configs; no differences again. Curious. Worryingly, this was starting
to look like a hardware support issue.

I pulled up my Thinkpad's spec sheet and saw it uses a Qualcomm NCM825 WLAN +
BT chip. Some strategic Googling (pro-tip: use double quotes to ensure your
search matches specific terms) revealed other Linux users across various
distributions and hardware configurations having similar issues with this chip.
While sad to see my hypothesis reinforced, it was reassuring that it wasn't
just me. This meant it was more likely to be solved!

I continued my search and stumbled upon [this post in the Framework computer
forum](https://community.frame.work/t/guide-successful-wi-fi-7-802-11be-on-framework-13-amd-with-qualcomm-qcncm865-and-arch-linux/44723),
where someone had installed it in their computer. They, too, had issues
connecting audio devices via Bluetooth. However, the author was knowledgeable
enough to submit [a patch to the Linux kernel's Bluetooth
subsystem](https://patchwork.kernel.org/project/bluetooth/patch/20241121180742.156230-1-greyxor@protonmail.com/)
that allowlisted the chip for certain wideband audio capabilities. The patch
was approved and merged. Exciting! Unfortunately, it was only in the
`bluetooth-next` repository, meaning it was not in a stable kernel.

When would it be out? I pulled the Git history of the latest kernel
(`6.13-rc3`) and was pleased to see the commit was included. Unfortunately, the
6.13 branch is not scheduled to be stable until late January 2025. I didn't
want to wait that long to validate this was the issue.

Time to install a new kernel.

Arch doesn't make release candidate kernels (referred to as `mainline`)
available for installation via Pacman (likely for good reason). Instead, one
can try building it themselves (which takes a long time and can be finicky to
get right for my distribution), installing an AUR-maintained version (which
would also take a long time), or... installing someone else's prebuilt kernel
(which requires trusting an internet rando). Given the feedback I saw
regarding the prebuilt kernels, I felt okay moving forward with that.

To make those available, open `/etc/pacman.conf` and add a section:

```ini
[miffe]
SigLevel = Optional TrustAll
Server = https://arch.miffe.org/$arch/
```
I then saved it and updated my package databases (`pacman -Syy`), accepting the
prompt to import a new PGP key. I then installed the release candidate kernel
(`pacman -S linux-mainline linux-mainline-headers`) and rebooted into it. I
placed my headphones in pairing mode, selected them from the device list,
and... success! They paired, and sure enough, I was able to play some music.

<s>Now, I could have stopped there and run a pre-release kernel for a month
until a stable version was released, but this is my work system. I'd prefer to
avoid bleeding-edge instability where possible. Since the problem appeared to
be with pairing, now that the headphones were paired with the system, I
wondered if I could just boot into my old kernel. Nothing to lose. I rebooted
into the old version, hit connect, opened Tidal, hit play, and was greeted with
my music.</s> EDIT (2024-12-20): This is incorrect. It still suffered from the
same issue, so I needed to remain on the RC kernel.

Moral of the story: never give up.

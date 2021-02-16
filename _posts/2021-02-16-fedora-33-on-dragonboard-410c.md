---
layout: post
title: "Fedora 33 on Dragonboard 410c"
date: 2021-02-16
---

In this post, I describe how to get Fedora 33 Server running on the Dragonboard
410c board.

A while back we got some Dragonboard 410c boards[0]. The code we developed was
tested on Fedora 33, and I didn't feel like porting it to Debian or Fedora 29,
which work out-of-the-box on this board. Getting Fedora 33 to work on it
seemed much more interesting.

# Step 1 - Compilation!

The first step is to compile and install the kernel. I installed the
cross-compilation tools for armv8 (aarch64). Since I use Fedora 33 on my
development machine, I just ran `dnf install gcc-aarch64-linux-gnu`.

Next, I downloaded the latest kernel version from Linaro[1]. It has been
modified to work on the Dragonboard 410C. The latest version, as of this
time of writing, is aligned with Linux 5.9.9.

For the most part, I have followed the instructions in [2], with minor
modification.

Firstly, download the kernel source code. Note that the following command
automatically checks out branch `debian-qcom-dragonboard410c-20.11`. This
was the latest branch at the time of writing. See [2] to check if there's
a newer version.

```
git clone http://git.linaro.org/landing-teams/working/qualcomm/kernel.git -b debian-qcom-dragonboard410c-20.11 kernel-dragonboard
```

The code has been downloaded into `kernel-dragonboard`. If this takes too long,
you can add `--depth 1` to avoid downloading the git history.

Next, the default configuration has to be modified. Apply the following patch:
```diff
diff --git a/kernel/configs/distro.config b/kernel/configs/distro.config
index f7dc7c00a72d..f84bcec8347b 100644
--- a/kernel/configs/distro.config
+++ b/kernel/configs/distro.config
@@ -189,6 +189,10 @@ CONFIG_EXT3_FS_SECURITY=y
 CONFIG_EXT4_FS=y
 CONFIG_EXT4_FS_POSIX_ACL=y
 CONFIG_EXT4_FS_SECURITY=y
+CONFIG_XFS_FS=m
+CONFIG_XFS_QUOTA=y
+CONFIG_XFS_POSIX_ACL=y
+CONFIG_XFS_ONLINE_SCRUB=y
 CONFIG_TMPFS_POSIX_ACL=y
 CONFIG_AUTOFS4_FS=y
 CONFIG_TMPFS_XATTR=y
```

This adds support for the xfs file system, as it is configured in the stock
Fedora 33 kernel. This is the file system used in the ready-to-use Fedora 33
Server `aarch64` image.

Next, set the following environment variables:
```
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
```

Note that in `CROSS_COMPILE`, if the cross-compiler binaries (e.g.,
`aarch64-linux-gnu-gcc`) are not in your path, you should provide a full path
to them.

Create a kernel configuration from the modified default configuration:
```
make defconfig distro.config
```

Compile the kernel image, the Device Tree Blob, and the kernel modules:
```
make -j$(nproc) Image.gz dtbs modules KERNELRELEASE=5.9.0-qcomlt-arm64
```

Next, install the kernel modules to some location. You will need them later.
```
make modules_install KERNELRELEASE=5.9.0-qcomlt-arm64 INSTALL_MOD_STRIP=1 INSTALL_MOD_PATH=<directory>
```

That's it for compilation. The kernel image is in `arch/arm64/boot/Image.gz`,
the device tree blobs are in `kernel/arch/arm64/boot/dts/qcom`. There are a
bunch there, but it also includes the correct one for the Dragonboard 410c
board. There is no initrd image yet. We still need to create that.

# Step 2 - initrd

I created an initrd image with dracut. I figured this would work best
for Fedora. Running `dracut` works best when ran from the same image and
architecture as the target. So first off, hop over to [3], and download Fedora
33's `aarch64` raw image. This blog post was tested with the Server edition.

Once you have the raw image, it might be best to start by making a copy of
it, as we are going to edit it. Start a Virtual Machine with the image. Make
sure that the virtual machine uses it as its disk, and doesn't create a
copy or overlays it in some way, or converts it to a different format. Using
`virt-manager` with `libvirt` gave me some good results.

Don't forget that the architecture is `aarch64`. You may need to install some
additional packages for it, such as `qemu-system-aarch64`.

Once the VM is up and running, copy the compiled kernel modules to
`/lib/modules/`. I created a tar archive:
```
MODDIR=<INSTALL_MOD_PATH directory given in the make modules_install step>
cd $MODDIR/lib/modules
tar -zcf /tmp/5.9.0-qcomlt-arm64.tar.gz 5.9.0-qcomlt-arm64
```

Do not make the mistake of archiving and extracting the entire tree. There is
some symbolic tree magic that breaks when you do that. And then the VM is
unusable. And then you spend the long seconds it takes to restore the copy
considering your life choices. You made a copy, right?

Once the archive has been copied to the VM (e.g., using scp, or shared
directories), you can extract it:
```
tar -zxf <path to archive> -C /lib/modules/
```

We will need to tell dracut that we need additional modules. This is because
all the device manager modules are built into the kernel in Fedora, but are
built as kernel modules in our kernel's default configuration.

Create the file `/etc/dracut.conf.d/dragonboard.conf` with the following
content:
```
add_drivers+=" dm_mirror dm_region_hash dm_log dm_mod xfs "
```
Note the spaces around the module names, including the first and last one.

You may also want to give root a temporary or empty password. This password
will be used when the initrd image drops into a rescue shell. By default the
root account is locked and has no password. Leaving it like this means that
the rescue shell is effectively inaccessible. `passwd -u root` to set a
password, or `passwd -d root` to clear the password. Run `passwd -l root` to
lock the account again when you are done.

Once ready, run `sudo dracut /tmp/initrd-dragonboard.img`. This will work for
a while. Cross-architecture virtual machines are painfully slow. Once complete,
copy the image back to the host, using `scp` or something.

# Stage 3 - Create Boot Image

To create the boot image, you will need to clone the skales repository for
android bootloaders[4]:
```
git clone https://us.codeaurora.org/quic/kernel/skales
```

This is the one recommended by the Dragonboard 410c documentation. I also
read the code and saw that they match in terms of what the bootloader
expects versus what the image utility creates. Which does not match the
android bootloader header 100%. There are minor differences.

Firstly, you will need to pack the device tree blobs.
```
skales/dtbTool -o dtb-dragonboard-5.9 kernel-dragonboard/arch/arm64/boot/dts/qcom
```

Note that you may have to fiddle with the directory locations. I assume you
cloned skales and the kernel into the same directory.

Once done, we can create a boot image. The following command line creates it in
`linux-5.9.img`:
```
skales/mkbootimg --kernel=kernel/arch/arm64/boot/Image.gz --output=linux-5.9.img --dt=dtb-dragonboard-5.9 --pagesize 2048 --base 0x80000000 --ramdisk=initrd-dragonboard.img --cmdline="root=/dev/mapper/fedora_fedora-root ro rd.lvm.lv=fedora_fedora/root rhgb quiet mdss_mdp.panel=1:dsi:0:qcom,mdss_dsi_hx8394d_720p_video:1:none "
```

You can add the following options to increase debugging and to use the serial
console: `console=ttyMSM0,115200n8 debug earlycon rd.shell`. Setting up the
serial console is described here [5].

You will also need to write the main Fedora 33 image to a micro-SD or USB
stick. Use the one from the VM, where the kernel modules were installed.
Otherwise, the modules may be missing and some things won't work. Once done,
insert the micro-SD or USB stick into the device.

To test the boot image, connect the device to your computer using the micro-USB
port. Power on the device while pressing the volume down (-) button on the
device. It is the middle small black button next to the host USB ports. This
will start the device in *fastboot*.

You can verify the device is started in *fastboot* by running the following:
```
sudo fastboot devices
```

You may need to install the fastboot package for this to work.

You should see output similar to the following:
```
451f3c38	fastboot
```

To test the new boot image without writing it to the device, run
```
sudo fastboot boot linux-5.9.img
```

If this works well, you can repeat the process and flash the image to the
device. It will then boot without *fastboot*:
```
sudo fastboot flash boot linux-5.9.img
```

# Step 4: Troubleshooting

First, note that these instructions are for Fedora 33 Server. The workstation
version would need minor modification. At the very least, the workstation
version uses `brtfs` and not `xfs`. I also haven't managed to get the HDMI
display to work. Everything is done using the serial console.

I ran into a few problems. This is how I got around them:

Firstly, if the kernel refuses to boot, or refuses to see the block devices on
the SD or USB storage devices, verify you are using the correct kernel. That
is, make sure you are using the kernel from [1].

If it fails to find the Device Tree Blob (DTB), verify you have used `dtbTool`
to convert the directory to a file, and passed `--dt` to `mkbootimg`. Verify
also that you are using `mkbootimg` from the `skales` repo, and not the one
provided by Fedora.

If the initrd fails to start the shell, make sure you ran `dracut` from a
matching architecture (e.g., using an `aarch64` VM).

Next, via the rescue shell, you can run `blkid`. It should show you some
partitions on `/dev/mmcblk0p<X>`, and your partitions on `/dev/mmcblk1p<X>`.
In case of a USB, your partitions will show up as `/dev/sda<X>`, or something
similar.

If you don't, and your image is on a USB stick, try again with the computer
disconnected.  That is, disconnect the micro-USB cable from the board. The
USB controller on the board cannot be a device (micro-USB port active) and
host (two USB ports active) at the same time.

Supposing this works, you can also run `lvm_scan`. You should get something
similar to the following:
```
sh-5.0# lvm_scan
Scanning devices mmcblk1p3  for LVM logical volumes fedora_fedora/root
ACTIVE '/dev/fedora_fedora/root' [5.41 GiB] inherit
sh-5.0#
```

`mmcblk1p3` will probably be replaced by the LVM partition on your SD or USB
drives.

If the above works, then your storage is visible. If mounting fails, make sure
you can load the `xfs` kernel module. `lsmod` lists all loaded modules, so you
can verify it is there.

Lastly, `journalctl -xe` is your friend. If something doesn't work, you should
review the logs there to try and understand what failed.

# Conclusion

It wasn't easy. A lot of work has gone into getting this board to work. The
important thing to remember is to use *their* kernel. It has been customised
to work on their board.

Notice that `u-boot` and `grub` are not used here. I didn't need it, and this
way seemed faster. It has its drawbacks. For starters, the size of the boot
image is limited to 64 megabytes.

I'll also note that your mileage may vary. I went through a lot of trial and
error to get this working. It's possible that I forgot to mention one or two
steps.

[0]: https://www.96boards.org/product/dragonboard410c/
[1]: https://git.linaro.org/landing-teams/working/qualcomm/kernel.git
[2]: https://releases.linaro.org/96boards/dragonboard410c/linaro/debian/latest/
[3]: https://getfedora.org/
[4]: https://us.codeaurora.org/quic/kernel/skales
[5]: https://www.96boards.org/documentation/consumer/dragonboard/dragonboard410c/guides/uart-serial-console.md.html


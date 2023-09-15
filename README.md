# Linuxbook
A guide to build the perfect Linux notebook

## Note
Google has restructured and updated their repositories quite a bit so this guide does not work as it is. I am currently working on an updated version to fix this.

## Initial thoughts
Chromebooks are great! At least from a Linux hardware support perspective because traditional devices do have proprietary hardware which might not be as well supported as we all hope. Chromebooks however should be fully compatiple with Linux because Chrome OS is nothing else than Linux (If you want to install a ChromeOS / ChromeOS forked kernel otherwise you might have to deal with hardware issues and potentially port some features). A second very important point is the security of chromebooks which can be harvested to secure your Linux system. In this guide we will turn a Lenovo Yoga C13 Chromebook to a (in my opinion) perfect Linux notebook. In the end we will end up with a fully encrypted system and a Linux distro of our choosing. Please note that this is only tested on a Lenovo Yoga C13. If there are differences on other devices please send me a pull request so that this turns into a more universal usable guide. All patches and resources you need will be contained in this repository.

## Preparing the chromebook
First of you should have Developer Mode enabled. You can do so by powering of the device and then press `ESC + ⟳ + Power`. You are now in recovery mode. Now press `CTRL + D` and hit confirm. Congratulations you are now in Developer Mode.

**Note:** Before enabling Developer Mode make sure that you have backed up all your data as the disk will be erased during the procedure.

The next step however includes a little bit more effort than the previous one. Now it's time to disable write protect. To do so please follow MrChromeboxes guide as it's really well explained. You can find his guide here: https://wiki.mrchromebox.tech/Firmware_Write_Protect#Disabling_WP_on_CR50_Devices_via_CCD . 

**Note:** A Suzy-Q Cable is a very good way to recover your Chromebook if you have made a mistake. I heavily recommend it even though it's not strictly required.

The next thing that needs to be done is to enable our Chromebook to load Chrome OS from other disks and to load UEFI payloads. This can be done by pressing `CTRL + ALT + t` when your device is powered on and you are logged in. This will bring up the Crosh shell which is the default shell in Chrome OS. Now simply type in shell to access a "normal" Linux shell. Now to enable the two options as mentioned before we have to modify the so called GBB Flags. These are a bunch of options or flags that are stored in the RO (read only) part of our firmware. To do so execute the following two commands: `sudo crossystem dev_boot_altfw=1`, `sudo crossystem dev_boot_usb=1`. Now you can reboot your device and see if it worked. By doing so you will notice that two options have appeared at the boot screen. The first one ("Boot from external disk") is used to boot Chrome OS from an external drive. The second one ("Select alternate bootloader → Tianocore") is used to boot a UEFI application like most of your "normal" desktops or notebooks.

Now one last step is required before we can start. It consists of backing up the stock firmware and therefore making faster recovery possible. To do so fire up the Linux shell as before, change to a writable directory and execute `sudo flashrom -r stock.rom`. Now you will get a file named "stock.rom" that you should copy to a safe place (definitely not your chromebook!!!).

## Implementing Verified Boot
One of the aspects that make a Chromebook so secure is Google's Verified Boot feature. It allows for a completely cryptographically verifiable boot process from firmware to rootfs. This is a feature I definitely want in my Linuxbook. Now this took a lot of effort to get right and therefore I will only show you the how and not the why behind of all this. 

### Generating the keys
First of all we need to generate the keys that will be used to verify the firmware and the operating system. This is done using openssl. The following keys need to be generated:
1. root_key - RSA8192, SHA512
2. recovery_key - RSA8192, SHA512
3. recovery_kernel_data_key - RSA8192, SHA512
4. kernel_subkey - RSA4096, SHA256
5. kernel_data_key RSA2048, SHA256
6. firmware_data_key - RSA4096, SHA256
7. dev_firmware_data_key - RSA4096, SHA256

**Note:** The RSA and SHA versions listed here are just my personal preference. In fact every algorithm combination listed in the output of `vbutil_key` can be used. However it's recommendet to start with stronger keys and then move down to weaker ones. Using weaker keys has the benefit of decreased booting times but results in weaker firmware/kernel protection. In the following commands everything written in curly braces needs to be replaced accordingl. Also they should be executed on Chrome OS.

First generate a RSA keypair using the following command: 
`openssl genrsa -F4 -out {outfile.pem} {RSA lenght}`

Then generate a self-signed certificate:
`openssl req -batch -new -x509 -key {infile.pem} -out {outfile.crt}`

Now generate a pre-processed RSA public key:
`dumpRSAPublicKey -cert {infile.crt} > {outfile.keyb}`

Finally wrap the keys:
1. Public: `vbutil_key --pack {outfile.vbpubk} --key {infile.keyb} --version 1 --algorithm {algorithm id}`
2. Private: `vbutil_key --pack {outfile.vbprivk} --key {infile.pem} --algorithm {algorithm id}`

To generate a keyblock use:
`vbutil_keyblock --pack {outfile.keyblock} --flags {flag} --datapubkey {pubkey.vbpubk} --signprivate {signkey.vbprivk}`

The following keyblocks need to be created:
1. recovery_kernel.keyblock (`datapubkey: recovery_kernel_data_key.vbupk, signprivate: recovery_key.vbprivk, flags: 13`)
2. kernel.keyblock (`datapubkey: kernel_data_key.vbpubk, signprivate: kernel_subkey.vbprivk, flags: 5`)
3. dev_kernel.keyblock (`datapubkey: kernel_data_key.vbpubk, signprivate: kernel_subkey.vbprivk, flags: 7`)
4. firmware.keyblock (`datapubkey: firmware_data_key.vbpubk, signprivate: root_key.vbprivk, flags: 5`)
5. dev_firmware.keyblock (`datapubkey: dev_firmware_data_key.vbpubk, signprivate: root_key.vbprivk, flags: 7`)

Now a directory with the following files should exist (Names are important!):
- dev_firmware_data_key.vbprivk
- dev_firmware_data_key.vbpubk
- dev_firmware.keyblock
- dev_kernel.keyblock
- firmware_data_key.vbprivk
- firmware_data_key.vbpubk
- firmware.keyblock
- kernel_data_key.vbprivk
- kernel_data_key.vbpubk
- kernel.keyblock
- kernel_subkey.vbprivk
- kernel_subkey.vbpubk
- recovery_kernel_data_key.vbprivk
- recovery_kernel_data_key.vbpubk
- recovery_kernel.keyblock
- recovery_key.vbprivk
- recovery_key.vbpubk
- root_key.vbprivk
- root_key.vbpubk

The keyblock signatures can be verified by `vbutil_keyblock --verify` 

### Resigning the firmware
Now we need to get the firmware rom by executing: `sudo flashrom -p host -r {outfile.rom}`

Luckily there already exists a script to resign a firmware image. It's on https://chromium/googlesource.com/chromiumos/platform/vboot_reference. The important folder here is the `scripts/imagesigning` subdirectory.

**Note:** You might need to clone it using a already existing linux computer and then transfer it over as git is no installed on chromebooks.

By executing `sign_official_build.sh firmware {infile.rom} {key directory} {signed_outfile.rom}` the `infile.rom` will be resigned and available to flash which can be done by calling `flashrom -p host -w {signed_infile.rom}`.

## Building and signing the kernel
Now all that is left to do is building a kernel signing it and then copy it to the hard drive. When compiling the kernel you can copy the .config of your stock chromebook (.config is accessible through the associated kernel module) and just `make olddefconfig CC=clang`. This should produce a custom kernel of whatever version you want that runs. Once this configuration is running you can start making changes to it. 

**Note:** If you have a Lenovo Yoga C13 you can get the newest working kernel at my linux-morphius repository. This kernel is now also uploaded to the AUR.

Depending on your system and kernel version you might need to use the `amd-rt5683-c13-chromebook.patch` which fixes a compiler error in the audio system. To sign the resulting kernel we need to execute: `vbutil_kernel --pack {outfile.bin} --keyblock {kernel.keyblock} --signprivate {kernel_data_key.vbprivk} --version 1 --vmlinuz {bzImage} --bootloader {bootloader.bin} --config {config.txt}`

**Note:** The mentioned `bootloader.bin` is not used in the current chromebooks. Therefore a file generated with `dd if=/dev/zero of=bootloader.bin bs=1K count=1` can be used. The `config.txt` can be extracted from a running Chrome OS system and adjusted. As a reference a working `config.txt` is provided (It works on my machine aka a Yoga C13 Chromebook :D). If a developer kernel is wanted the `dev` versions of the keyblock and private key should be used.

The resulting kernel file needs to be truncated to `33554432 bytes` and can be verified by `vbutil_kernel --verify`. Copy the kernel partition to an external disk.

## Formatting the harddrive and installing Linux
Now restart the Chromebook and make sure that you end up in the developer mode bootscreen. Boot a Linux distro from an external device by booting from UEFI (I used Ubuntu 18.04 LTS as newer Ubuntu version had some graphical issues). If you don't have the option to do so (for example on Gemini Lake Devices) then proceed by formatting an external harddrive according to the next paragraph and installing a linux distro this way. Booting with the option "Boot from external disk" will enable you to still boot from linux even if no UEFI boot is shipped with your chromebook. This is also a good way to test your kernel before installing it to the main harddrive. Then use the `sudo wipefs -a -f {/dev/device}` to clear all headers. Now run `sudo fdisk {/dev/device}` and type `g` and `w`. Then use `sudo cgpt create {/dev/device}` to finally create the headers. The `cgpt add` tool should be used to partition the hard drive according to this standart: https://chromium.googlesource.com/chromiumos/docs/+/HEAD/disk_format.md#Drive-contents. To get the maximum amount of space for the root partition a 1 block long `state` partition, a 65536 blocks long `kernel` partition and a `root` partition (filling up the remaining space) need to be created.
The kernel partition should have the flags `priotity=1` and `successful=1`. Use `dd` to move the kernel partition. A Linux distro can be installed on the `root` partition (Tested with Ubuntu 18.04 LTS and Voidlinux). Now reboot, return to secure mode and enjoy Linux.

**Note:** Make sure you don't accidently create EFI partitions or partitions for GRUB this might lead to an OS reinstall.

## Fix Audio
To get audio working take a look at https://github.com/eupnea-linux/audio-scripts. Installing this made my audio instantly work on Arch with a 6.2.10-Kernel

## Final thoughts
### Dual Booting
Dual booting is definitely possible with the following approach:
1. Compile a kernel that supports `kexec`
2. Install a tiny linux distro on your root
3. `kexec` a kernel on another partition

This way various possibilities for device encryption are possible.

### Plans
In the future I want to work on a system that allows easy dual booting by providing a custom made distro featuring a boot menu, possibilities for device encryption and mounting of Chrome OS utilities (so that they don't need to be compiled for every distro). Also I want to make the whole process more user friendly to allow more usage of this technique as it results in cheap, solid and secure notebooks with incredible boot times.
With eventually more people getting involved testing of various different models would be required so that a universal application can be crafted that allows easy installation of various operating systems on different devices.

### Using UEFI
If you want to use your chromebook like a normal notebook look at the incredible work MrChromebox (https://mrchromebox.tech/). There is all the information you need to reflash your chromebook and installing UEFI OSes and other things. Also great explanation for unbricking etc. is provided.

## Sources
- https://mrchromebox.tech/
- https://wiki.mrchromebox.tech/
- https://chromium.googlesource.com/
- https://chromium.googlesource.com/chromiumos/docs
- https://link.springer.com/chapter/10.1007/978-1-4842-0070-4_5
- https://www.chromium.org/chromium-os

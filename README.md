# Linuxbook
A guide to build the perfect Linux notebook

## Initial thoughts
Chromebooks are great! At least from a Linux hardware support perspective because traditional devices do have proprietary hardware which might not be as well supported as we all hope. Chromebooks however should be fully compatiple with Linux because Chrome OS is nothing else than Linux. A second very important point is the security of chromebooks which can be harvested to secure your Linux system. In this guide we will turn a Lenovo Yoga C13 Chromebook to a (in my opinion) perfect Linux notebook. In the end we will end up with a fully encrypted system and a Linux distro of our choosing. Please note that this is only tested on a Lenovo Yoga C13. If there are differences on other devices please send me a pull request so that this turns into a more universal usable guide. All patches and resources you need will be contained in this repository.

## Preparing the chromebook
First of you should have Developer Mode enabled. You can do so by powering of the device and then press `ESC + ⟳ + Power`. You are now in recovery mode. Now press `CTRL + D` and hit confirm. Congratulations you are now in Developer Mode.

**Note:** Before enabling Developer Mode make sure that you have backed up all your data as the disk will be erased during the procedure.

The next step however includes a little bit more effort than the previous one. Now it's time to disable write protect. To do so please follow MrChromeboxes guide as it's really well explained. You can find his guide here: https://wiki.mrchromebox.tech/Firmware_Write_Protect#Disabling_WP_on_CR50_Devices_via_CCD . 

**Note:** A Suzy-Q Cable is a very good way to recover your Chromebook if you have made a mistake. I heavily recommend it even though it's not strictly required.

The next thing that needs to be done is to enable our Chromebook to load Chrome OS from other disks and to load UEFI payloads. This can be done by pressing `ALT + t` when your device is powered on and you are logged in. This will bring up the Crosh shell which is the default shell in Chrome OS. Now simply type in shell to access a "normal" Linux shell. Now to enable the two options as mentioned before we have to modify the so called GBB Flags. These are a bunch of options or flags that are stored in the RO (read only) part of our firmware. To do so execute the following two commands: `sudo crossystem dev_boot_altfw=1`, `sudo crossystem dev_boot_usb=1`. Now you can reboot your device and see if it worked. By doing so you will notice that two options have appeared at the boot screen. The first one ("Boot from external disk") is used to boot Chrome OS from an external drive. The second one ("Select alternate bootloader → Tianocore") is used to boot a UEFI application like most of your "normal" desktops or notebooks.

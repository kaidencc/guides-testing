<link rel="stylesheet" type="text/css" href="styles.css">

# Dump APFS devices on Linux
This guide will show you how to dump APFS devices on Linux using Legacy iOS Kit and SSH ramdisk.<br>

## Note
**This guide is still in testing and will be published on the main site at [kaiden.cc](https://kaiden.cc/) once fully tested.**<br>

## Disclaimer
I am **not** responsible for any damage to your devices caused by following this guide. Please proceed with caution and at your own risk.<br>

## Note
When I use angle brackets (`< >`), they indicate placeholders. Do not include the brackets themselves in your input.<br>

## Requirements
- **A Linux system** with USB access<br>
- [Legacy iOS Kit](https://github.com/LukeZGD/Legacy-iOS-Kit) for the SSH ramdisk functionality<br>
- [gaster](https://github.com/0x7ff/gaster) **(AMD users only)** for pwning the device on an Intel PC<br>
- **An iOS device** that can be put into DFU mode<br>
- **gtar** (GNU tar) for creating stable tar archives<br>
- **iproxy** and **usbmuxd** for SSH communication with the device<br>
- **Sufficient disk space** for the dump files (at least 2x your device storage)<br>

## Initial Setup

Download and run Legacy iOS Kit<br>
`git clone https://github.com/LukeZGD/Legacy-iOS-Kit.git && cd Legacy-iOS-Kit`<br>

Plug the device into your computer in DFU mode<br>

**AMD users:** You must first pwn the device using an Intel PC with gaster, then plug it into DFU mode and run `gaster pwn`, after which you can replug it to your main AMD PC<br>

Run Legacy iOS Kit<br>
`./restore.sh`<br>

## Booting SSH Ramdisk

Follow the instructions for initial setup, then navigate to **Other Utilities > SSH ramdisk**<br>

The ramdisk version does not have to match the build currently on your device, as long as you do **not** mount the filesystems during this process<br>

When the ramdisk has booted successfully, leave the Legacy iOS Kit terminal window open (do **not** mount the filesystems), and open a separate new terminal window<br>

## Dumping disk0

Create a new folder for the dump files<br>
`mkdir dump && cd dump`<br>

Dump disk0 (this will take some time)<br>
`ssh root@localhost -p 6414 -o StrictHostKeyChecking=no 'dd if=/dev/rdisk0 bs=4096' | gzip > disk0.dmg.gz`<br>
*Note: When asked for a password, enter "alpine" as the password*<br>

Go back to the Legacy iOS Kit window, and select the option to reboot the device<br>

## Setting up SSH over USB

When the device has fully booted back into SwitchBoard, open a new terminal window and manually restart usbmuxd<br>
`sudo killall -9 usbmuxd 2>/dev/null && sudo usbmuxd -pf`<br>

Leave that terminal running in the background and open another terminal window to run iproxy<br>
`iproxy 2222 22`<br>

Leave that terminal also running in the background and open another new terminal in the dump folder you created earlier<br>

## Preparing gtar

Download gtar and send it to the device (gtar is more stable than the standard BSD tar that comes with iOS)<br>
`curl -LO https://applintrnl.app/tools/dumping/gtar && chmod +x gtar`<br>
`scp -P 2222 -o StrictHostKeyChecking=no gtar root@localhost:/var/mobile/gtar`<br>
*Note: When asked for a password, enter "alpine" as the password*<br>

## Dumping the filesystem

You must additionally dump the full filesystem as a tar archive (this is necessary because the data partition is encrypted when it's not mounted on the device)<br>
`ssh root@localhost -p 2222 -o StrictHostKeyChecking=no '/var/mobile/gtar -cpzvf - /' > fs.tar.gz`<br>
*Note: When asked for a password, enter "alpine" as the password*<br>

## Dumping disk1 and disk2

Now dump disk1 and disk2 partitions<br>
`ssh root@localhost -p 2222 -o StrictHostKeyChecking=no 'dd if=/dev/disk1' | dd of=nand_firmware`<br>
`ssh root@localhost -p 2222 -o StrictHostKeyChecking=no 'dd if=/dev/disk2' | dd of=nand_llb`<br>
*Note: When asked for a password, enter "alpine" as the password*<br>

## Get build train

To get the build train (build name and number), fsck the system volume<br>
`fsck_hfs -f /dev/disk0s1s1`<br>
Once the command has completed, it should say *"The volume <build train> appears to be OK."*<br>
Save the build train somewhere, it will be important if you want to rebuild the system as a disk image.<br>

**Done!** You should now have all the necessary dump files in your dump folder.<br>

## Contact

If you are having issues with this guide or think something needs to be explained clearer, feel free to reach out for support.<br>

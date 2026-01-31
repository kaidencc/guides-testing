<link rel="stylesheet" type="text/css" href="styles.css">

# Dump APFS devices without a ramdisk
This guide will show you how to dump APFS devices directly from SwitchBoard without needing to boot an SSH ramdisk.<br>

## Note
**This guide is still in testing and will be published on the main site at [kaiden.cc](https://kaiden.cc/) once fully tested.**<br>

## Disclaimer
I am **not** responsible for any damage to your devices caused by following this guide. Please proceed with caution and at your own risk.<br>

## Requirements
- **A system** (Linux or macOS) with USB access<br>
- **An iOS device** booted into SwitchBoard<br>
- **gtar** (GNU tar) for creating stable tar archives<br>
- **iproxy** and **usbmuxd** for SSH communication with the device<br>
- **Sufficient disk space** for the dump files<br>

## Setting up SSH over USB

Connect the device while it is in SwitchBoard<br>

**Linux users only:** When the device has fully booted into SwitchBoard, open a terminal and manually restart usbmuxd<br>
`sudo killall -9 usbmuxd 2>/dev/null && sudo usbmuxd -pf`<br>
Leave this terminal in the background.<br>

Open a new terminal window and run iproxy<br>
`iproxy 2222 22`<br>

Leave that terminal also running in the background and open another new terminal<br>

## Preparing gtar

Create a folder for the dump files<br>
`mkdir dump && cd dump`<br>

Download gtar and send it to the device (gtar is more stable than the standard BSD tar that comes with iOS)<br>
`curl -LO https://applintrnl.app/tools/dumping/gtar && chmod +x gtar`<br>
`scp -P 2222 -o StrictHostKeyChecking=no gtar root@localhost:/var/mobile/gtar`<br>
*Note: When asked for a password, enter "alpine" as the password*<br>

## Dumping the filesystem

Dump the full filesystem (this will take some time)<br>
`ssh root@localhost -p 2222 -o StrictHostKeyChecking=no '/var/mobile/gtar -cpzvf - /' > fs.tar.gz`<br>
*Note: When asked for a password, enter "alpine" as the password*<br>

## Dumping firmware partitions

Dump disk1 (NOR)<br>
`ssh root@localhost -p 2222 -o StrictHostKeyChecking=no 'dd if=/dev/disk1 bs=4096' | dd of=nand_firmware`<br>
*Note: When asked for a password, enter "alpine" as the password*<br>

Dump disk2 (LLB)<br>
`ssh root@localhost -p 2222 -o StrictHostKeyChecking=no 'dd if=/dev/disk2 bs=4096' | dd of=nand_llb`<br>
*Note: When asked for a password, enter "alpine" as the password*<br>

**Done!** You should now have all the necessary dump files in your dump folder.<br>

## Contact

If you are having issues with this guide or think something needs to be explained clearer, feel free to reach out for support.<br>

# Tutorial moved: 
[https://dmitrykryukov.com/blog/sophos-ap55-openwrt](https://dmitrykryukov.com/blog/sophos-ap55-openwrt)

# Tested using iMac 2011 and Macbook pro (M1 Pro)

## What you need
* ethernet cable
* screw driver
* CP2102 module USB to TTL serial UART (I bought mine on Aliexpress for 3$)
  
## Step 1. Prepare your Sophos.
Disassemble it and find the UART header. 
![IMG_3812](https://github.com/user-attachments/assets/23fda168-6636-4b02-bca3-191f4a10e13e)

Header pins on the picture from left to right: 
* pin 1 - RX
* pin 2 - TX
* pin 3 - GND
* pin 4 - 3.3v (won't be used)

Your USB-to-TTL adapter should be connected to the header. GND to GND, RX to TX, TX to RX, leave 3.3V unconnected.

1. To make it work on mac you need to install `CP2102` driver from here: https://www.silabs.com/developer-tools/usb-to-uart-bridge-vcp-drivers?tab=downloads.

> I also uplaoded a copy to this repo.

3. Connect your USB-to-TTL adapter to the AP and connect it to the Mac usb port. At this point we don't powerup the AP yet.

4. Command `ls /dev/tty.*` supposed to show the device now: `/dev/tty.SLAB_USBtoUART`

5. Now you can open a separate terminal window and run screen: `screen /dev/tty.SLAB_USBtoUART 115200` 

## Step 2. Configure Ethernet
Connect the Sophos AP 55C to your Mac via Ethernet cable. Assign a Static IP to your Mac's Ethernet adapter:

* Go to System Preferences > Network
* Select your Ethernet interface and change DHCP to manually
* IP Address: `192.168.99.8`
* Subnet Mask: `255.255.255.0`
* Router: Leave blank.

## Step 3. Time to power it up
UART and Ethernet are prepared. Time to power on the AP. 

Then check logs in terminal screen app and when you see this: `Press any key to stop autoboot:` press any key to interrupt booting. You'll see the console `ath>`

## Step 4. Download firmwares (latest I used 23.05.5)
Go here: https://downloads.openwrt.org/releases/23.05.5/targets/ath79/generic/
For me the `AP100C` firmware worked. I used this tutorial (https://forum.openwrt.org/t/howto-installing-openwrt-on-sophos-ap-55-and-ap-100/171914) for reference.

So, download initramfs-kernel and squashfs-sysupgrade versions.
> I also uploaded what I used to this repo

## Step 5. Set Up a TFTP Server
* Make sure you have a folder `/private/tftpboot` 
* also give it 777 permissions: `sudo chmod 777 /private/tftpboot`
* Then load the service: `sudo launchctl load -w /System/Library/LaunchDaemons/tftp.plist`

Then copy `initramfs-kernel.bin` firmware you downloaded recently to the `tftpboot` folder and rename it to `uImage_AP100C`:

* `cp /path/to/openwrt-23.05.5-ath79-generic-sophos_ap100c-initramfs-kernel.bin /private/tftpboot/uImage_AP100C`
* Also give it 777 persmisssions just in case: `chmod 777 /private/tftpboot/uImage_*`
* and test it with these commands:
```
tftp localhost
tftp> get uImage_AP100C
tftp> quit
```

## Step 6. Load firmware to the AP
If all the above was configured correctly. 

We can try to upload the firmware to the AP. The command is running on the AP where we have `ath>` prompt:

* `setenv serverip 192.168.99.8`
* `setenv ipaddr 192.168.99.9`
* `printenv` - to check we saved the env for tftp server address
* `tftpboot` - this one supposed to load the firmware from our tftp server.

You should see progress logs showing the firmware is being loaded into RAM at address `0x81000000`

We need to set the firmware address:
* `setenv fwaddr 0x9f070000`
* `erase $fwaddr +$filesize` - Here we erase the current firmware from the flash (Dangerous step, not sure we can revert this somehow)
* `cp.b $fileaddr $fwaddr $filesize` - and write the uploaded firmware to flash
* `iminfo $fwaddr` - everything should be ok 

## Step 7. Boot
run `boot` command on the AP and it supposed to reboot and boot to the openwrt (you'll see logs).

## Step 8. Change Ethernet again
Set your Mac’s Ethernet back to DHCP auto.
* Go to System Preferences > Network
* Select your Ethernet interface and use DHCP automatic.

> Probably you'll have to restart AP, but not sure about this step

Open a browser and go to http://192.168.1.1 (there is no password by default)
user: root
password: [no password]

## Step 9. Install sysupgrade firmware
If it works fine and you can enter the OpenWRT interface, you can install the firmware permanently.

> Our firmware is in the initramfs mode it will be reset to default everytime we restart the AP.

Your AP is connected to your mac so it doesnt have internet. We need to push the sysupgrade firmware to the AP. 

### Option 1 - try to use scp and uplaod sysupgrade firmware to the AP
`scp /path/to/openwrt-23.05.5-ath79-generic-sophos_ap100c-squashfs-sysupgrade.bin root@192.168.1.1:/tmp`

### Option 2 - Run python server and download firmware right from the AP
Create a folder `firmwares` on Desktop and put firmware there. 
```
cd ~/Desktop/firmwares
python3 -m http.server 8000
```

Then ssh to the AP `ssh root@192.168.1.1` from there you can download the binary.

I don't exactly remember the IP you should use but it supposed to be the IP of you mac where you running the Python server. Play around, use ping and find the right IP.
* `wget http://192.168.1.10:8000/openwrt-23.05.5-ath79-generic-sophos_ap100c-squashfs-sysupgrade.bin -O /tmp/sysupgrade.bin`


Once you have a sysupgrade.bin on your AP you can run this:
`sysupgrade /tmp/sysupgrade.bin`

## Step 10. Finishing 
At this point you should have OpenWRT installed and configured. By default with static IP: `192.168.1.1`

Now you can left is as is and just set a password from the openwrt interface when you go to http://192.168.1.1 or change it in the settings to dhcp so you can connect it to the router and it will give the AP automatic IP.

Alternatively you can do that over ssh. 
`vi /etc/config/network`
```
config interface 'lan'
    option proto 'static'
    option ipaddr '10.11.1.2'  # Replace with desired static IP
    option netmask '255.255.255.0'
    option gateway '10.11.1.1'   # Replace with your router's IP
    option dns '8.8.8.8'         # Replace with preferred DNS server
```
`service network restart`

## Useful links
https://git.openwrt.org/?p=openwrt/openwrt.git;a=commit;h=6f1efb28983758116a8ecaf9c93e1d875bb70af7

https://forum.openwrt.org/t/howto-installing-openwrt-on-sophos-ap-55-and-ap-100/171914

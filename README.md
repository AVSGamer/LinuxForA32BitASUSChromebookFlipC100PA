# Installing Linux on an Asus Chromebook Flip C100PA
Documentation of my journey into learning more about Linux and Repurposing a 10 year old almost obsolete hardware by installing and configuring a different operating system for it.

## Distros I've tried
### Arch Linux 32Bit for ARM Systems like this device:
- Current(July 2026) official instruction from Arch Linux Website works(only as USB Mode) only if you change the main instruction set's used 'http://os.archlinuxarm.org/os/ArchLinuxARM-veyron-latest.tar.gz' to 'http://os.archlinuxarm.org/os/ArchLinuxARM-armv7-chromebook-latest.tar.gz'
- Haven't tried modifying the bios beforehand in conjunction with the original .tar.gz or any tar.gz for that matter.
- Can read all the raw notes in this repo's only .txt file to see stuff I've tried.
- Had to configure the keyboard shortcuts + audio routing + volume changes notifications UI/UX myself and it was messy work.. Got them to work still and it's in the Raw Notes text file.
#### Used the Arch Linux Live USB to install these to the internal storage
### PrawnOS:
- Works booted from internal storage, not tested much, opted to use the next in this list instead.
### PostMarketOS:
- What I ended up using instead, boots from internal storage.
- Works great out of first boot, no need to configure the volume adjustment toast/dunst-notifications, all keys on keyboard working as supposed to be with the device's defaults.
- No need to mess around trying to fix the audio routing, works out-of-the-box or rather out of fresh install.
- No available pre-built image online so I used the website's instruction on building the system image file using the Live USB Arch Linux aforementioned above.
- The above had me running 'install pmbootstrap command' using my Linux's package manager then 'pmbootstrap init' then mostly using defaults by just pressing enter key then selecting the 'community build' instead of the 'edge' or bleeding edge experimental build, then choosing xfce4 as the display manager then the rest are mostly defaults. You don't need to specify to have 'sudo' installed as it's pre-packaged with it and would rather not put any in the extras-to-install asked. Once the image is built use 'pmbootstrap install --no-cgpt --sdcard /dev/mmcblk0' to install into the internal storage. Check if it finishes with 'DONE!', if it doesn't then just keep repeating running that command until it does, at first execution it's slow because it has to download some stuff but subsequent executions will be faster as it only needs to verify that the files are already there and not corrupted to skip trying to download them again.
- if you need to run sudo for the above use --as-root flag right after the pmbootstrap word.

## Once the Operating System or OS is installed and tested to be running from internal storage
For simplicity's sake since this the editor I been using a lot in Linux and we'll be mentioned hereon out, you can use whichever you want:  
```bash
sudo apk add nano
```  
It's also a good habit to read through the entire instruction set such as this first before actually executing/doing them one by one, and because of that I purposely hid the way to save edits in nano somewhere down this guide :D  
Not kidding! That's a life-school-hack to finish exams faster! You skim through every question first identifying every easy ones so you don't spend too much time on difficult questions and end up losing time to answer those easy ones.  
But in this case it's more of a "Is this real or a rick-roll?" or is this guide seemingly off with missing details? Where can it go wrong?  
Install flashrom for the firmware modification if you want to prevent your LinuxOS installation getting bricked by accidentally re-enabling "OS Verification" or someone does it for you ;):  
```bash
sudo apk add flashrom
```  
Install the ChromeOS Futility package:  
```bash
sudo apk add vboot-utils
```

Modify the bios flags to shorten the booting time's default ChromeOS' Developer White Screen where you used to have to press 'ctrl+d' for internal storage boot or 'ctrl+u' for usb boot. And turn that into taking only 2 seconds instead of 30 seconds. Also disabling the spacebar function on that screen to 'Re-enable OS Verification' so that your Linux Installation doesn't get bricked. And lastly forcing the device to be always in dev_mode, totally eliminating the need to press 'ctrl+d' everytime to boot into the Linux OS.

## How To Modify the BIOS Flags
This is specifically an old Chromebook where in those years they used a ground screw as the Firmware/Bios Write-Protection Mechanism, so you have to open the chassis and remove that first.  
In newer models, it seems to be directly attached to the battery's power management board so for that, you might have to remove the battery, I am not sure with this one so better look at the link below:  
https://docs.mrchromebox.tech/docs/firmware/wp/disabling.html <- this should serve as a better source of information on how to do this whole section.  
Seems like in most Chromebooks, the Write-Protect Screw is a screw with an arrow marking pointing to it AND once you unscrew/remove it, underneath is a circular copper with tin/silver dots and you can imagine why that works similar to switch as it closes the circuit serving as the Hardware Level Write-Protection for the Firmware/Bios. Since you've opened it up might as well clean it with a not too strong cleaning air-spray can or use a not too hard bristle brush ensuring the write protect screw's previous placement area is no longer connecting where the divided by half or quarters circular copper is.  
After that comes disabling the Software Level Write-Protect Switch/Flag. 'sudo flashrom --programmer internal --wp-disable --wp-range 0,0' <- this disables the Software Level Firmware Write-Protection of the BIOS.  
Now we can backup our existing installed FW/BIOS then reflash that back in with a command that modifies the BIOS Flags.  
'sudo flashrom -r gbb.bin' <- The -r flag there tells the command 'flashrom' to backup the existing installed BIOS followed by where you want to save it and including the filename with the correct extension of '.bin'  
I personally saved mine on the external live usb installation of ArchLinux32Bit so that even if the device itself get's bricked by this operation, I should be able to log back into the Live USB and restore it.  
So I used this instead 'sudo flashrom -r /usr/veyronfwbak/bios_backup.bin'  
'futility gbb -s --flags=0x9 gbb.bin' <- This modifies the 'gbb.bin' or whatever file you named it as to replace the current BIOS Flags with '0x8' + '0x1' = '0x9' in that backed up file.  
For reference on the flags: https://chromium.googlesource.com/chromiumos/platform/vboot_reference/+/firmware-link-2695.B/scripts/image_signing/set_gbb_flags.sh  
The way those flags identifiers are structed is in a way that you add up all the numbers of the flags you want to use then it gets divided into the different flags to use based on the highest number of a flag that is known/usable downwards. Like ?reductive?, in the case of 0x32: it is 0x20 with 12 leftover so the next flag it enables is 0x10 then 0x2.  
'flashrom -w gbb.bin' <- this flashes/writes your modified BIOS/FW file into the one currently installed, overwriting it. Overwriting is a computer term that just means whatever was in it is now replaced with what is just placed in it.  
Can reboot the system now.  

## The Hard Shutdown Fix
Somehow this specific model has problems fully shutting down. The OS does shutdown but the Disk-Read-LED or light on the side doesn't turn off meaning the device is not really fully turned off yet.  
This can cause heat build up as it is stored in a bag as it is if forgotten to long-press the power button to do a hard shutdown shouldn't be done everytime.  
I've tried ACPID but that doesn't work since this installation have nowhere to hook that into.  
What worked is a SystemD.service file configuration.  
The kernel or very low-level system command to shut the power off or disconnect the battery from the board does work so we put that in the SystemD.service that we will add.  
SystemD is not the only ?system-services-manager? you could be using a different one depending on the display-manager you selected in making your PostMarketOS Imagefile, it could be 'rc-update' or BusyBox-RC or something like that.  
We first need to unlock the emergency interface of the kernel:  
'sudo echo 1 > /proc/sys/kernel/sysrq' <- this survives reboots.  
To test that it works:  
'sudo sync' <- saves your current session/work/some settings.  
'sudo echo o > /proc/sysrq-trigger' <- this will suddenly/abruptly turn off the device without even showing you the shutdown animation and should turn off the Disk-Reading LED.

Instead of routing the XFCE4's GUI Power Commands to 'systemctl poweroff' command like I had to try as suggested by the LLM I was using. That in conjunction with a script that calls those commands and needs a lot of permission manipulation to get all set up and ultimately just not work. We make/use a SystemD Service instead to connect the command to a system process that is called when pressing shutdown or any power options from the GUI.

Make the service file:  
```bash
sudo nano /etc/systemd/system/chromebook-hardware-poweroff.service
```
Inside that we put:  
```
[Unit]
Description=Asus Chromebook Hard Poweroff Trigger
DefaultDependencies=no
Before=umount.target
#Conflicts is what prevents reboots from not fully occuring since in order to reboot, it has to pass through umount.target first
#And this file as indicated by WantedBy at the bottom is executed when umount.target is called by the system.
#shutdown.target (Software Exit Phase)
#then
#umount.target (Sealing of the '/' directory or essentially your entire rootfs or storage partition/drive then dismounting it.)
#then
#final.target (Last phase of the shutting down right before cutting electricity off from the motherboard.)
#then
#poweroff.target (Electricity Kill Switch)
Conflicts=reboot.target

[Service]
#Type is set to only get triggered once when called so it actually just sits as 'inactive' when checked via 'systemctl status' command once enabled.
Type=oneshot
#/bin/sh is the location of the sh program that is responsible for running scripts or '.sh' files
#essentially '/' is your rootfs partition's highest directory then 'bin/sh' is inside that.
#You will notice this when you use 'cd' or change directory a lot in conjunction with 'ls' or 'dir' to see the contents of your current location in the console(user@/:~)
ExecStart=/bin/sh -c "sync && sleep 5 && echo o > /proc/sysrq-trigger"

[Install]
WantedBy=umount.target
```

Double check you got that correctly with no UpperCase/LowerCase/Typo/Misspellings. Might be good to remove the # lines or comment lines too.

Reload the systemctl daemon:  
```bash
sudo systemctl daemon-reload
```  
Enable the service we just created:  
```
sudo systemctl enable chromebook-hardware-poweroff
```


If it doesn't work then you might need to give all users of the device the ability to send the 'o' signal to the kernel emergency interface; /proc/sysrq-trigger.  
Nothing wrong with the above statement since eventually we want any user to be able to fully shutdown the device and not just your selected users.  
'sudo chmod 666 /proc/sysrq-trigger' <- The problem is this doesn't survive reboots. So we make a configuration file to re-apply it every boot-up.  
Make the directory if it doesn't exist yet:  
```bash
sudo mkdir -p /etc/tmpfiles.d
```  
Make the file inside that directory:  
```bash
sudo nano /etc/tmpfiles.d/chromebook-power.conf
```  
Put this in that file:  
```
f /proc/sysrq-trigger 0666 root root - -
```  
Save with ctrl+o then enter then exit via ctrl+x IF you are using nano as your editor, I've heard VIM is good but haven't really tried it and my first experience finding it hard to just even exit the editor, I am flabbergasted.

That should be enough since SystemD runs as 'root' but if it still doesn't work, then maybe using the code below to give ALL users a Passwordless Execution of the command:  
#Edit your sudoers' configuration file, basically managing user's and their access to the sudo(run as root/admin) command:  
```bash
sudo VISUAL=nano visudo
```  
Add this at the very bottom, but this shouldn't really be needed at all: It just makes executing the command 'systemctl poweroff' and 'systemctl halt' from the console run without asking for admin password.
We aren't routing the XFCE4's GUI Power Commands to 'systemctl poweroff'...  
```
ALL ALL=(ALL) NOPASSWD: /usr/bin/systemctl poweroff, /usr/bin/systemctl halt
```

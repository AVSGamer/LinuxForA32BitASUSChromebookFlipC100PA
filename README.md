# ArchLinux32BitForASUSChromebookFlipC100PA
Documentation of my journey into learning more about Linux and Repurposing a 10 year old almost obsolete hardware by installing and configuring a different operating system for it.

## Arch Linux ARM Installation & Customization Guide for Asus Chromebook Flip C100PA (Veyron Minnie)
This repository contains a comprehensive guide, optimized configuration files, and utility scripts to install, configure, and maintain a lightweight Arch Linux ARM (32-bit) environment on the 10-year-old Asus Chromebook Flip C100PA (pp. 1, 11).
The setup uses the LXDE desktop environment and the Openbox window manager (p. 3) to optimize performance on the device's 16GB internal storage and older ARM processor (pp. 11, 13).
------------------------------
## 📋 Table of Contents

   1. Prerequisites & Entering Developer Mode
   2. Enabling External Storage Booting
   3. Partitioning the Installation Media
   4. Base OS Installation & Core Configuration
   5. Desktop Environment & Display Manager
   6. System Optimization Scripts
   * Volume Management (vol_change.sh)
      * Hardware Audio Fix (chromebook-audio-fix.sh)
      * Audio Initialization (chromebook-audio-init.sh)
      * Power Button Dunst Wrapper (power_dunst.sh)
      * Multi-User Administration Scripts
   7. Hardware & Performance Tweaks
   8. Browser Performance Tweaks
   9. Local Network Screen Streaming
   10. To-Do / Future Roadmap

------------------------------
## 1. Prerequisites & Entering Developer Mode

(!WARNING)
Enabling developer mode will factory reset your system. Back up all crucial local ChromeOS data before proceeding (p. 1).


   1. Turn off the Chromebook (p. 1).
   2. Invoke Recovery Mode by holding down ESC + Refresh keys and poking the Power button (p. 1).
   3. At the Recovery screen, press Ctrl + D (there is no visible prompt) (p. 1).
   4. Confirm switching to developer mode by pressing Enter (p. 1). The reset process will take approximately 15–20 minutes (p. 1).
   5. Note: Once enabled, you must press Ctrl + D at every boot splash screen or wait 30 seconds to bypass the warning (p. 1).

------------------------------
## 2. Enabling External Storage Booting

   1. Boot into ChromeOS (p. 1).
   2. Open the Chrome OS developer shell (Crosh) by pressing Ctrl + Alt + T (p. 1).
   3. Open a proper bash environment (p. 1):
   
   shell
   
   4. Elevate to root privileges (p. 1):
   
   sudo su
   
   5. Enable booting from unsigned USB and external operating systems (p. 1):
   
   crossystem dev_boot_usb=1 dev_boot_signed_only=0
   
   6. Reboot the system to apply your changes (p. 1).

------------------------------
## 3. Partitioning the Installation Media
These commands target a USB Flash Drive mapped to /dev/sda (p. 1).

(!TIP)
If installing to a microSD Card, adjust your device target throughout the instructions to /dev/mmcblk1 (p. 1).


   1. Drop back into your root shell (sudo su) and unmount existing automated ChromeOS mounts (p. 1):
   
   umount /dev/sda*
   
   2. Initialize an empty GPT partition table via fdisk (p. 1):
   
   fdisk /dev/sda
   
   * At the interactive fdisk prompt, type g to write a new empty GPT layout (p. 1).
      * Type w to save changes and exit (p. 1).
   3. Create the specialized ChromeOS-compatible kernels and rootfs layouts using cgpt (p. 1):
   
   cgpt create /dev/sda
   cgpt add -i 1 -t kernel -b 8192 -s 32768 -l Kernel -S 1 -T 5 -P 10 /dev/sda
   
   4. Check your partition table blocks using cgpt show /dev/sda to find the start sector of your Sec GPT table (p. 1). Assuming standard boundaries, calculate and scale your rootfs partition (p. 1):
   
   # Replace 15633375 with your exact Sec GPT table start index if it differs
   cgpt add -i 2 -t data -b 40960 -s `expr 15633375 - 40960` -l Root /dev/sda
   
   
------------------------------
## 4. Base OS Installation & Core Configuration

   1. Force the kernel to refresh partition maps and format the new system root partition to ext4 (p. 2):
   
   partx -a /dev/sda
   mkfs.ext4 /dev/sda2
   
   2. Download and extract the formal Arch Linux ARM tarball structure (p. 2):
   
   cd /tmp
   curl -LO http://archlinuxarm.org
   mkdir root
   mount /dev/sda2 root
   tar -xf ArchLinuxARM-armv7-chromebook-latest.tar.gz -C root
   
   3. Flash your embedded kernel partition directly onto partition 1 (p. 2):
   
   dd if=root/boot/vmlinux.kpart of=/dev/sda1
   umount root
   sync
   
   4. Reboot the machine (p. 2). When the developer splash warning appears, press Ctrl + U to bypass local eMMC and boot to your external drive (p. 2).
   5. Log in as root (Default Password: root) (p. 2).
   6. Connect your phone via USB and enable USB Tethering to pass temporary network connectivity down to the Chromebook (p. 2).
   7. Populate pacman authentication rings and fetch primary system configurations (p. 2):
   
   pacman-key --init
   pacman-key --populate archlinuxarm
   pacman -S firmware-veyron
   
   8. Reboot the system (p. 2). You can now connect to local networks natively using wifi-menu (p. 2).

------------------------------
## 5. Desktop Environment & Display Manager## Base Package Installation
Install the display system, core window tooling, lightweight desktop paneling, background communication tools, and the display manager (p. 3):

pacman -Syu xorg-server xorg-xinit lxde lightdm lightdm-gtk-greeter sudo sakura xterm dbus acpi
systemctl enable lightdm
systemctl enable --now dbus

## Local User Permissions & Sudo Implementation
Configure user structures and elevate administrators (p. 3):

usermod -aG wheel alarm

Run visudo and uncomment the global execution parameters for wheel-bound actors (p. 3):

%wheel ALL=(ALL:ALL) ALL

## System Locale Fixes
To prevent visual crashes across virtual terminals, generate appropriate character schemas (p. 3):

sed -i '/en_US.UTF-8 UTF-8/s/^# //g' /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" | tee /etc/locale.conf
echo "export LANG=en_US.UTF-8" >> ~/.bashrc
localectl set-locale LANG=en_US.UTF-8

------------------------------
## 6. System Optimization Scripts## Volume Management (vol_change.sh)
Create this script at /usr/local/bin/vol_change.sh to handle responsive volume configurations using WirePlumber and notify standard Dunst listeners (p. 7).

#!/usr/bin/env bash
export XDG_RUNTIME_DIR="/run/user/$(id -u)"
if [ "$1" == "toggle" ]; then
    wpctl set-mute @DEFAULT_AUDIO_SINK@ toggleelse
    wpctl set-volume -l 1.0 @DEFAULT_AUDIO_SINK@ "$1"fi

MUTE_STATE=$(wpctl get-volume @DEFAULT_AUDIO_SINK@ 2>/dev/null)
VOLUME_RAW=$(echo "$MUTE_STATE" | awk '{print $2}')
if [ -z "$VOLUME_RAW" ]; then
    VOLUME=0else
    VOLUME=$(echo "$VOLUME_RAW" | awk '{print int($1 * 100 + 0.5)}')fi

HP_JACK_STATE=$(amixer -c 0 cget numid=67 2>/dev/null | grep ": values=" | awk -F= '{print $2}')if [[ "$HP_JACK_STATE" == "on" ]]; then
    DISPLAY_NAME="Headphones"else
    DISPLAY_NAME="Speakers"fi
if [[ "$MUTE_STATE" == *"[MUTED]"* ]]; then
    notify-send -a 'Volume' -h string:x-dunst-stack-tag:volume "$DISPLAY_NAME: Muted"else
    notify-send -a 'Volume' -h string:x-dunst-stack-tag:volume -h int:value:"$VOLUME" "$DISPLAY_NAME: $VOLUME%"fi

## Hardware Audio Fix (chromebook-audio-fix.sh)
Create this script at /usr/local/bin/chromebook-audio-fix.sh to force raw hardware registers into standard ALSA paths (p. 8).

#!/usr/bin/env bash
sleep 3

amixer -c 0 cset numid=47 3 >/dev/null 2>&1
amixer -c 0 cset numid=48 3 >/dev/null 2>&1
amixer -c 0 cset name='Speaker Playback Switch' on,on >/dev/null 2>&1
amixer -c 0 cset name='Headphone Playback Switch' on,on >/dev/null 2>&1

amixer -c 0 cset name='Headphone Left Switch' on >/dev/null 2>&1
amixer -c 0 cset name='Headphone Right Switch' on >/dev/null 2>&1
amixer -c 0 cset name='Speaker Left Switch' on >/dev/null 2>&1
amixer -c 0 cset name='Speaker Right Switch' on >/dev/null 2>&1
amixer -c 0 cset name='LINMOD Mux' 'Left and Right' >/dev/null 2>&1

amixer -c 0 cset name='Left Speaker Mixer Left DAC Switch' on >/dev/null 2>&1
amixer -c 0 cset name='Right Speaker Mixer Right DAC Switch' on >/dev/null 2>&1
amixer -c 0 cset name='Left Speaker Mixer Right DAC Switch' off >/dev/null 2>&1
amixer -c 0 cset name='Right Speaker Mixer Left DAC Switch' off >/dev/null 2>&1

amixer -c 0 cset name='Left Headphone Mixer Left DAC Switch' on >/dev/null 2>&1
amixer -c 0 cset name='Right Headphone Mixer Right DAC Switch' on >/dev/null 2>&1
amixer -c 0 cset name='Left Headphone Mixer Right DAC Switch' off >/dev/null 2>&1
amixer -c 0 cset name='Right Headphone Mixer Left DAC Switch' off >/dev/null 2>&1

## Audio Initialization (chromebook-audio-init.sh)
Create this script at /usr/local/bin/chromebook-audio-init.sh to initialize the sound card pins on startup (p. 9).

#!/usr/bin/env bash
amixer -c 0 cset numid=77 on >/dev/null 2>&1
amixer -c 0 cset numid=29 0,0 >/dev/null 2>&1
amixer -c 0 cset numid=49 3 >/dev/null 2>&1
amixer -c 0 cset numid=123 on >/dev/null 2>&1
amixer -c 0 cset numid=124 off >/dev/null 2>&1
amixer -c 0 cset numid=54 35,35 >/dev/null 2>&1

## Power Button Dunst Wrapper (power_dunst.sh)
Create this script at /usr/local/bin/power_dunst.sh to intercept the physical chassis power switch and prevent accidental hard shutdowns (pp. 6, 9).

#!/bin/bash
export XDG_RUNTIME_DIR="/run/user/$(id -u)"

ACTION=$(dunstify -u critical \
 -t 8000 \
 -A "default,Shut Down" \
 "Power Button Pressed" "Tap notification to Confirm Shut Down.")
if [ "$ACTION" = "default" ] || [ "$ACTION" = "2" ]; then
    powerofffi

## Multi-User Administration Scripts## create_account.sh
Deploy this utility at /usr/local/bin/create_account.sh to streamline the onboarding of new standard or administrator profiles (p. 2).

#!/bin/bashif [ "$EUID" -ne 0 ]; then
    echo "This script needs admin privileges to create accounts."
    echo "Please enter your password below to proceed."
    exec sudo "$0" "$@"fi

clear
echo "=========================================="
echo "   Chromebook New User Account Creator    "
echo "=========================================="
echo ""
while true; do
    read -p "Enter a name for the new account (lowercase letters only, no spaces): " USERNAME
    USERNAME=$(echo "$USERNAME" | tr -d ' ' | tr 'A-Z' 'a-z')
    if [ -z "$USERNAME" ]; then
        echo "The account name cannot be blank. Please try again."
        echo ""
    elif id "$USERNAME" &>/dev/null; then
        echo "An account with the name '$USERNAME' already exists. Please choose a different name."
        echo ""
    else
        break
    fidone

echo ""
echo "------------------------------------------"
echo "        Account Permission Level          "
echo "------------------------------------------"
echo "Should this user be an Administrator?"
echo ""
echo " [N] Standard User (Recommended for guests/kids)"
echo "     - CANNOT install or remove software."
echo "     - CANNOT alter core system settings."
echo "     - Helps save storage space on your 16GB internal storage."
echo ""
echo " [Y] Administrator"
echo "     - CAN install new applications."
echo "     - CAN update the system and change settings."
echo "     - Full control over the entire Chromebook."
echo "------------------------------------------"
while true; do
    read -p "Make this user an administrator? (y/n): " CHOICE
    CHOICE=$(echo "$CHOICE" | tr 'A-Z' 'a-z')
    if [[ "$CHOICE" == "y" || "$CHOICE" == "yes" ]]; then
        GROUPS="audio,video,wheel"
        IS_ADMIN=true
        break
    elif [[ "$CHOICE" == "n" || "$CHOICE" == "no" || -z "$CHOICE" ]]; then
        GROUPS="audio,video"
        IS_ADMIN=false
        break
    else
        echo "Invalid input. Please type 'y' for Yes or 'n' for No."
    fidone

echo ""
echo "Creating account for '$USERNAME'..."
useradd -m -G "$GROUPS" "$USERNAME"
if [ $? -eq 0 ]; then
    echo "Account tracking folder created successfully."
    echo ""
    echo "------------------------------------------"
    echo "Now, let's set a password for $USERNAME."
    echo "Note: The letters will NOT show on screen as you type them for safety."
    echo "------------------------------------------"
    while true; do
        passwd "$USERNAME"
        if [ $? -eq 0 ]; then
            echo ""
            echo "Success! The password has been set."
            break
        else
            echo "Passwords did not match or were too weak. Let's try again."
            echo ""
        fi
    done
    echo ""
    echo "=========================================="
    echo " Done! The account '$USERNAME' is ready."
    if [ "$IS_ADMIN" = true ]; then
        echo " Privilege Level: Administrator (Can install apps)"
    else
        echo " Privilege Level: Standard User (Restricted access)"
    fi
    echo "=========================================="else
    echo "An unexpected error occurred while creating the account."fi

## delete_user.sh
Deploy this utility at /usr/local/bin/delete_user.sh to safely list and erase user profiles (p. 14).

#!/bin/bashif [ "$EUID" -ne 0 ]; then
    echo "This script needs admin privileges to delete accounts."
    echo "Please enter your password below to proceed."
    exec sudo "$0" "$@"fi

clear
echo "=========================================="
echo "     Chromebook User Account Remover      "
echo "=========================================="
echo ""
echo "Fetching a list of accounts that can be deleted..."
echo "Press Alt+C to cancel this process anytime..."
echo "------------------------------------------"

USER_LIST=$(awk -F: '$3 >= 1000 && $1 != "nobody" {print $1}' /etc/passwd)if [ -z "$USER_LIST" ]; then
    echo "No deletable user accounts found on this Chromebook."
    echo "=========================================="
    exit 0fi

echo "Found the following accounts on this machine:"
echo ""
echo "$USER_LIST"
echo "------------------------------------------"
echo ""
while true; do
    read -p "Type the EXACT name of the account you want to delete: " TARGET_USER
    TARGET_USER=$(echo "$TARGET_USER" | tr -d ' ' | tr 'A-Z' 'a-z')
    if [ -z "$TARGET_USER" ]; then
        echo "The account name cannot be blank."
        echo ""
    elif ! id "$TARGET_USER" &>/dev/null; then
        echo "The account '$TARGET_USER' does not exist. Please check the list above."
        echo ""
    elif [ "$TARGET_USER" = "$SUDO_USER" ] || [ "$TARGET_USER" = "$(whoami)" ]; then
        echo "⚠️ WARNING: You cannot delete the account you are currently logged into!"
        echo ""
    else
        break
    fidone

echo ""
echo "------------------------------------------"
echo "          Storage Cleanup Option          "
echo "------------------------------------------"
echo "Should we delete their files too?"
echo ""
echo " [N] Keep Files (Safest)"
echo "     - Deletes the login account."
echo "     - KEEPS their documents and pictures inside /home/$TARGET_USER."
echo ""
echo " [Y] Erase Everything (Recommended for your 16GB Internal Storage)"
echo "     - Deletes the login account."
echo "     - PERMANENTLY erases all files, folders, and history for this user."
echo "     - Instantly frees up disk space."
echo "------------------------------------------"
while true; do
    read -p "Permanently erase their files and folders? (y/n): " CLEAN_FILES
    CLEAN_FILES=$(echo "$CLEAN_FILES" | tr 'A-Z' 'a-z')
    if [[ "$CLEAN_FILES" == "y" || "$CLEAN_FILES" == "yes" ]]; then
        DEL_FLAGS="-r -f"
        PURGE_DESC="Account and all files successfully wiped."
        break
    elif [[ "$CLEAN_FILES" == "n" || "$CLEAN_FILES" == "no" || -z "$CLEAN_FILES" ]]; then
        DEL_FLAGS="-f"
        PURGE_DESC="Account deleted. Their files were safely kept inside /home/$TARGET_USER."
        break
    else
        echo "Invalid choice. Please type 'y' for Yes or 'n' for No."
    fidone

echo ""
echo "Processing removal for '$TARGET_USER'..."
killall -u "$TARGET_USER" &>/dev/null
sleep 1

userdel $DEL_FLAGS "$TARGET_USER"if [ $? -eq 0 ]; then
    echo ""
    echo "=========================================="
    echo " SUCCESS! $PURGE_DESC"
    echo "=========================================="else
    echo "❌ Error: Could not finish removing the account."fi

## Ensure Ownership & Execution Access
Ensure all customized tools are marked with strict root tracking rights and are set to execute properly across any console (pp. 8, 10):

sudo chown root:root /usr/local/bin/*.sh
sudo chmod 755 /usr/local/bin/*.sh

------------------------------
## 7. Hardware & Performance Tweaks## Disabling a Malfunctioning or Damaged Touchscreen
If your touchscreen has phantom clicks, completely disable it via X11 structures and modprobe blacklists (p. 4).

   1. Create /etc/X11/xorg.conf.d/99-disable-touchscreen.conf (p. 4):
   
   Section "InputClass"
       Identifier "Disable Chromebook Minnie Touchscreen"
       MatchIsTouchscreen "on"
       Option "Ignore" "on"
   EndSection
   
   2. Create /etc/modprobe.d/blacklist.conf (p. 4):
   
   blacklist elants_i2c
   
   3. Rebuild your ARM system image parameters (p. 4):
   
   sudo mkinitcpio -P
   sudo systemctl reset-failed systemd-udevd
   sudo udevadm control --reload
   
   4. Create the udev rules asset at /etc/udev/rules.d/00-block-elan.rules (p. 4):
   
   SUBSYSTEM=="i2c", ENV{MODALIAS}=="i2c:elants*", OPTIONS="ignore_device"
   
   
## Real-Time Network Clock Configurations
Since Veyron hardware lacks a typical real-time clock (RTC) backup cell, map standard time servers across systemd daemons (p. 4). Adjust to your local timezone (p. 4):

sudo ln -sf /usr/share/zoneinfo/Asia/Manila /etc/localtime
sudo timedatectl set-ntp true
sudo systemctl enable --now systemd-timesyncd

## Direct Audio Power Adjustments
To prevent click/pop sounds caused by power savings on audio components, create /etc/modprobe.d/audio-power-fix.conf (p. 6):

options snd_hnd_intel power_save=0 power_save_controller=N
options snd_soc_core pm_disable=1

## Hardware Event Routing Rules
Bind your background tasks and rules explicitly down to systemd managers (pp. 8, 10).
## System Environment Integration (/etc/xdg/openbox/autostart)

@lxpanel --profile LXDE
@pcmanfm --desktop --profile LXDE
@xscreensaver -no-splash
@dunst
@/usr/local/bin/chromebook-audio-fix.sh

## Audio Hardware Triggers (/etc/udev/rules.d/99-chromebook-audio.rules)

SUBSYSTEM=="sound", ACTION=="add", RUN+="/usr/local/bin/chromebook-audio-init.sh"

## Intercepting System Power Triggers (/etc/systemd/logind.conf)
Modify logind parameters to allow Openbox to intercept the physical power switch (p. 10):

[Login]
HandlePowerKey=ignore

Apply the configuration instantly (p. 10):

sudo systemctl restart systemd-logind

## Native Keybind Interceptions (~/.config/openbox/lxde-rc.xml)
Inject these custom bindings inside your window settings configurations to map brightness, volume adjustments, and modern terminals like Sakura (pp. 3, 5):

<keyboard>
  <chainQuitKey>C-g</chainQuitKey>
  <keybind key="C-A-t">
    <action name="Execute">
      <command>sakura</command>
    </action>
  </keybind>
  <keybind key="F8">
    <action name="Execute">
      <command>/usr/local/bin/vol_change.sh toggle</command>
    </action>
  </keybind>
  <keybind key="F9">
    <action name="Execute">
      <command>/usr/local/bin/vol_change.sh 5%-</command>
    </action>
  </keybind>
  <keybind key="F10">
    <action name="Execute">
      <command>/usr/local/bin/vol_change.sh 5%+</command>
    </action>
  </keybind>
  <keybind key="XF86AudioLowerVolume">
    <action name="Execute">
      <command>/usr/local/bin/vol_change.sh 5%-</command>
    </action>
  </keybind>
  <keybind key="XF86AudioRaiseVolume">
    <action name="Execute">
      <command>/usr/local/bin/vol_change.sh 5%+</command>
    </action>
  </keybind>
  <keybind key="XF86PowerOff">
    <action name="Execute">
      <command>/usr/local/bin/power_dunst.sh</command>
    </action>
  </keybind>
</keyboard>

Reload your keybindings profile on the fly without restarting your session (pp. 4, 6):

openbox --reconfigure

## Manual Panel Weather Configuration Fallback
Since the legacy Yahoo Weather API within LXPanel is deprecated, you must manually inject hard-coded coordinates directly into your workspace configuration file (p. 10).
Edit ~/.config/lxpanel/LXDE/panels/panel and update the weather block using your local city details and WOEID (pp. 10-11):

Plugin {
    type=weather
    Config {
        alias="Floridablanca"
        city="Floridablanca"
        state="Pampanga"
        country="Philippines"
        woeid="1167440"
        units=c
        interval=15
        enabled=1
    }
}

------------------------------
## 8. Browser Performance Tweaks
To ensure modern sites run smoothly on older ARM processors, perform these manual configurations after installing Firefox (p. 11):

sudo pacman -Syu firefox


   1. Install uBlock Origin via Extension Channels: Open Firefox, navigate to the official Mozilla Add-ons registry, and add uBlock Origin (p. 11).
   2. Enable Strict Tracker Blocking: Open the uBlock Origin dashboard settings, navigate to Filter Lists, and ensure Suspicious domains and AdGuard URL Tracking Protection are active (p. 11). This prevents heavy tracking tracking scripts from placing overhead on your CPU (p. 11).
   3. Disable Firefox Telemetry: Navigate to about:preferences#privacy inside the address bar (p. 11). Locate Firefox Data Collection and Use and uncheck all analytic streams to save precious background execution cycles (p. 11).

------------------------------
## 9. Local Network Screen Streaming
You can stream your Chromebook's display across your network directly into standard observation toolsets like OBS Studio on a target machine (pp. 11-12).

   1. Install FFmpeg components and loopback audio parameters (p. 11):
   
   sudo pacman -S ffmpeg
   sudo modprobe snd-aloop
   
   2. Configure your internal mixer mapping file inside /etc/asound.conf (p. 11):
   
   pcm.!default {
       type plug
       slave.pcm "dmixer"
   }
   pcm.dmixer {
       type dmix
       ipc_key 1024
       ipc_key_add_uid false
       ipc_perm 0660
       slave {
           pcm "hw:0,0"
           rate 44100
           channels 2
       }
   }
   ctl.!default {
       type hw
       card 0
   }
   
   3. Deploy the network pipeline transmission script at /usr/local/bin/start_stream.sh (p. 12):
   
   #!/usr/bin/env bash
   RESOLUTION="1280x800"
   TARGET_IP="192.168.1.2"  # Change this to your target computer's IP address
   PORT="9999"
   
   arecord -f CD -D hw:1,0 2>/dev/null | ffmpeg -f x11grab -video_size "$RESOLUTION" -framerate 24 -i :0.0 -f s16le -ar 44100 -ac 2 -i pipe:0 -c:v libx264 -pix_fmt yuv420p -preset ultrafast -profile:v baseline -tune zerolatency -c:a mp2 -b:a 128k -f mpegts udp://$TARGET_IP:$PORT
   
   4. Execute sudo sh /usr/local/bin/start_stream.sh on the Chromebook (p. 12).
   5. Open OBS Studio on your receiving machine, add a new Media Source, uncheck Local File, and set the input parameter to udp://127.0.0.1:9999 (or udp://0.0.0.0:9999) using the mpegts input format (p. 12).

------------------------------
## 📝 To-Do / Future Roadmap

* Update the Openbox/rc.xml to include recent modifications for the function keys and a more flexible Backspace/Delete key (p. 16).
* Build a robust dynamic memory monitor script to prevent system lockups during RAM or CPU spikes (p. 16).
* Troubleshoot the persistent "DiskReading Light On" hardware issue that causes thermal buildup after a supposedly full shutdown (p. 16).
* Install xdotool, brightnessctl, and slock. Those are used in the new Openbox/rc.xml (p. 16).
* Enforce standard global environment configurations to explicitly treat nano or Dave'sTinyEditor as the primary text/file editor system-wide (p. 16).
* Archive all dependencies and Google Chromebook Restore Media for this device locally within this repository for long-term preservation (p. 16).
* Add more in-depth instructions that this AI-Generated summary based off of my actual Notes omitted (p. 16).

------------------------------

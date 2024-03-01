# A tutorial to make a clean and recent armbian-klipper system
This starts as a tutorial for everyone to repeat and follow - as simple as I can do it for you :)
Later I will add custom images to make it even faster and easier for all!

## Who and what?
This tutorial is written for folks who are „not completely happy“ with their QIDI printers and want to get full access to the printer and unleash its full potential.
If you copy or take parts of this tutorial, please mention me, the author of this :)
If you do have any questions, find errors or want to add something, please join the discord channel and ping me:
Discord channel: https://discord.gg/8AJE3EA4
Author: coco.33

If you like my work here, I would be happy if you consider a tip

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/B0B4V3TJ6)

### First off. Why?
QIDIs X-3 series printers are made out of hardware with very good potential, but they do have outdated software. The printers could work better/faster with higher usability for every user.
Here is a list of software that bottlenecks the full functionality of the printer by time of writing this:
klipper 0.10.0 instead 0.12.0
Armbian Buster instead Bookworm
Python 2.7 instead 3.12
And all the outdated software that comes with this (fluidd, moonraker, etc.)
The disk comes prefilled with a lot of unnecessary things. ~6-7GB are used of the 8GB EMMC disk.
This means that additional features or even some GCODE files will fill your disk space entirely.

### Disclaimer:

Before you start, please understand that you will wipe the entire disk and build your OS from scratch.
You will also update all sub-systems (print head „THR“ and the STM32F402 MCU on the mainboard) to the latest klipper.
So you will have an entirely free and custom 3D printer.
Please note, that this will all be done to your own risk and please do not contact the QIDI support if you have any issues. By making these changes, you will lose your warranty in this regard.
There is however a „recovery“-image, that will restore the system to a state that QIDI ships their printers. You then only have to „downgrade“ the THR toolhead board and the mainboard MCU and you are basically back to the start and you should have your warranty back :)


### Pro and Cons


| Pro:            | Con:              |
| --------------- | ----------------- |
| Latest operating system (armbian bookworm), firmware (klipper 0.12) and softwares.
Everything is a bit snappier
Only ~2.2GB of disk space used (instead of stock ~6.5GB)
Better webcam functionality (stock printer don’t even uses crowsnest)
Freedom to install whatever you want, eg: ShakeTune, TelegramBot, KlipperScreen etc. which can improve print speed/quality and usability.             | The stock display will be out of order until someone reengineers it.
(The display is basically a standalone device with a screen. It communicates with the maiboard via serial and is not a real (Klipper-)screen. This was done by a modified klipper from QIDI to allow such a combination. This solution works, but has many drawbacks).  |


## Lets get started

I will describe step by step what and how I did it. Please follow carefully if you are not familiar with these kinds of operations and ask us in the discord channel if you run into some trouble :)

### What you will need
* Physical access to the printer
* An EMMC reader
* A micro-SD card (up to 32GB in size)

### Installing Armbian and Klipper:

Thanks to redrathnure there is a great Armbian based OS that we use as a foundation. So download the latest Armbian from https://github.com/redrathnure/armbian-mkspi to get started.
I used „Armbian-unofficial_24.2.0-trunk_Mkspi_bookworm_edge_6.6.7.img.xz“
Turn your printer off, wait ~30s.
Open the back cover and remove the EMMC flash.
Use BalenaEtcher and flash the armbian image to your EMMC using the EMMC reader.
Insert the EMMC back to your printer.
Turn it on and connect the printer via ethernet cable to your router.
Connect to your printer via putty (or a similar program) using its IP address.
Log in as
user: root
password: 1234


You will be asked to create a new user.
Create
user: mks
password: makerbase
Now start a new session in putty and login as user „mks“.
Update everything with
```
sudo apt update
sudo apt upgrade
```

Install KIAUH
```
sudo apt-get update && sudo apt-get install git -y
cd ~ && git clone https://github.com/dw-0/kiauh.git
./kiauh/kiauh.sh
```

Via KIAUH install these software in this order:
```
Klipper
Moonraker
Mainsail (of Fluidd if you prefer this)
```

If you do have a webcam, also install Crowsnest.
When asked for the python version, make sure to select python 3.x.


When you access your printer via a web browser, you will see the web interface Mainsail (or Fluidd). It will show you an error like this:

<img src="/screenshots/01.png">
This is because the sub systems being not yet updated with the latest klipper firmware.


### Updating the „THR“ tool head

The tool head uses a derivate of a „MKS THR“ (https://github.com/makerbase-mks/MKS-THR36-THR42-UTC) and is powered by a RP2040 μC.
For flashing it, you need to turn it into „dfu mode“.
During this tutorial we will do this once. For the future this won’t be necessary.
The reason for this is, because we will flash a special bootloader named „katapult“ (formerly known as CanBoot) that is able to bring the μC in dfu mode without the need of physical access.


#### Installing katapult:

If not still open, login via putty as user „mks“ and do
```
git clone https://github.com/Arksine/katapult
cd ~/katapult
make menuconfig
```
First, change the μC Architecture to RP2040
<img src="/screenshots/02.png">
<img src="/screenshots/03.png">

Make sure to build a katapult deployment application with the right size:
Finish by hitting „q“ and save with „y“:
<img src="/screenshots/04.png">
Now build the firmware by first cleaning the workspace and then compile with all 4 cores:
```
make clean
make -j4
```
You should see something like this:
<img src="/screenshots/05.png">

You should be able to see the RP2040 if you type:
```
ls /dev/serial/by-id/*
```
<img src="/screenshots/06.png">

You can see there is klipper installed on the RP2040


Put the tool head board in dfu mode like this:

1. Remove the back cover of the tool head.
2. Press and hold the „boot“ button.
3. Press and release the „reset“ button.
4. Release the „boot“ button.

Now prepare everything for flashing the katapult bootloader by this:
```
sudo mount /dev/sda1 /mnt
systemctl daemon-reload
```
<img src="/screenshots/07.png">

You will be asked to authenticate by entering the password:
<img src="/screenshots/08.png">

Now flash katapult to the RP2040 and unmount it:
```
sudo cp out/katapult.uf2 /mnt
sudo umount /mnt
```
If you check your devices again, you will see, that the klipper firmware was replaced with katapult
```
ls /dev/serial/by-id
```
It’s now possible to flash klipper with no access to the printer.
You can attach the back cover of the print head back on.


Now let‘s flash latest klipper
```
cd ~/klipper
make menuconfig
```
 
Set everything like you did with katapult like this:
<img src="/screenshots/09.png">
Quit with „q“ and save with „y“:
<img src="/screenshots/10.png">

Now build the firmware by first cleaning the workspace and then compile with all 4 cores:
```
make clean
make -j4
```
Check again for your device and remember the device ID by

https://openqidi.com/link/16#bkmrk-ls-%2Fdev%2Fserial%2Fby-id-2
 ```
ls /dev/serial/by-id/*
```
<img src="/screenshots/11.png">

ID is in this case “/dev/serial/by-id/usb-katapult_rp2040_C5DA4D951E145858-if00”
Install python3-serial to be able to invoke the flash process:
```
sudo apt install python3-serial
```
Now flash the toolhead – make sure to use YOUR device ID:
```
python3 ~/katapult/scripts/flashtool.py -f ~/klipper/out/klipper.bin -d /dev/serial/by-id/usb-katapult_rp2040_C5DA4D951E145858-if00
```

You should see something like this:
<img src="/screenshots/12.png">


Done.The tool head is now on the latest klipper firmware :)


## Flashing klipper to the STM32F402 mainboard MCU

Similar to flashing the rp2040 you need to:
```
cd ~/klipper
make menuconfig
```
Set everything according to this screenshot:
<img src="/screenshots/13.png">
Quit with „q“ and save with „y“:
<img src="/screenshots/14.png">

Now build the firmware by first cleaning the workspace and then compile with all 4 cores:
```
make clean
make -j4
```
This will generate a **„klipper.bin“** file in the **/home/mks/klipper/out/** folder.
Use your favourite program to get this file onto your computer (I am using WinSCP).


Format a microSD card as FAT32 (not lager in size then 32GB – maybe larger sized SD-cards will work if you create a partition not larger then 32GB).
Copy the „klipper.bin“ file to this SD-card and rename it to **„X_4.bin“**.
Eject the microSD card.
Shut down your printer and wait at least 30sec.
Put the microSD card into the SD card slot of the printers mainboard.
Turn the printer on. The mainboard STM32F402 MCU will now be flashed (which takes about 10sec, but make sure to not turn it down before 1min… just in case).


If you now go to the machine settings of the web interface (mainsail/fluidd) you will see the current klipper version of the sub systems:
<img src="/screenshots/15.png">


Unleash the full potential of the printer

For best results, a CoreXY printer needs the right and very similar tension of the belts. Fortunately there is a tool called “Klippain Shake&Tune” made by Frix-x (see here https://github.com/Frix-x/klippain-shaketune).
This tool gives you insights and a nice graph to compare the belts etc.
To install it, simply do:
```
wget -O - https://raw.githubusercontent.com/Frix-x/klippain-shaketune/main/install.sh | bash
```
To activate it, include this to your printer.cfg:

[include K-ShakeTune/*.cfg]

You can use this module now and the probably most useful ones being the one for measuring/comparing the belt tension, that you use by writing this in the mainsail/fluid console:
```
BELTS_SHAPER_CALIBRATION
```
When you are happy with your belts and tensioned it to a state they look good, you may want to recalibrate the input shapers like:
```
AXES_SHAPER_CALIBRATION
```
There are tons of information, so please have a look at the great documentation and examples given by Frix-x on its github page.

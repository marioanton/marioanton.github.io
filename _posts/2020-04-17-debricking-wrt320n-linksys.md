---
layout: post
title: debricking router linksys wrt320n
image: /img/router.png
tags: [wrt320,bricked]
---

I was messing around with my home network these days of lockdown and decided to use OpenVPn integrated within DD-WRT that was flashed to my Linksys wrt320N ages ago. The openvpn it came with had to be  reasonable old so i decided to give it whirl and upgrade the firmware. 
Firmware for upgrades can be found in here [Router Database](https://dd-wrt.com/support/router-database/?model=WRT320N_v1.0) 

That is how i ended up bricking my router. I went for the image `DD-WRT Broadcom Generic - NEWD K2.6 - Mega ` instead of `dd-wrt.v24-40559_NEWD-2_K2.6_mini_wrt320n DD-WRT: Factory flash`

Instead of throwing the router to the bin i decided to spend some money and time and invenstigate how this was achievable.

First of all checked whether this router had capabilities for me to flash a new firmware. Yes, it is possible and apparently can be done via 2 ways.

> The router has a Broadcom chipset and comes with CFE [Common Firmware Environment](https://en.wikipedia.org/wiki/Common_Firmware_Environment). This enables you to do the first method

#### Method 30-30-30:
  - This method is going to trigger flashing of firmware that is living in the tftp/nvram if it was in there previously. It does trigger this in the background: ` flash -ctheader : flash1.trx`. This basically reads a bin file with the flash image. It is worth trying.
  - To run this, you need to press the WPS button continuosly at the time you are switching on your router. Unfortunately this did not work for me.

#### Manual method:

> We will need:
 - USB to SERIAL UART. [AMAZON](https://www.amazon.co.uk/gp/product/B072K3Z3TL/ref=ppx_yo_dt_b_asin_title_o00_s00?ie=UTF8&psc=1). This one works pretty well and comes with output voltage of 3.3 and 5.5. I know, i can do this with a raspberrypi as well.
 - Ethernet RJ45 cable, with usb convertor to ethernet for your laptop if you don't have interface for it.
 - Soldering Kit
 - TFTP client

```Firts things first```

Linksys wrt320n comes with two ways to reach serial comm. 
- Using the WAN port and home made interface which i ruled out. If you look closely in the WAN port of your router you will see the RX, TX, GND, and 3.3v pins.
- Using the serial interfaces avaialble in the router motherboard. This method is easier since soldering is pretty accurate.

Serial interface available in the router motherboard:

![serial](https://github.com/marioanton/marioanton.github.io/raw/master/img/pins.png "router serial comm pins interface")

You can see already the mapping of the pins for the serial interface and also can see where you can find the otehr pins in the WAN interface, where serial is possible to be done as well.

We DON'T need to connect 3.3v. Don't do it and you will not put your router in risk. We weill be using normal router power source.

> Don't have to say that you have to open the router. In the process of opening up your router you will have to remove 3 wires connected to the motherboard. They are antenna ones.

Once opened, solder this as the image states. The soldering wires for TX and RX should be linked to the opposite ones in the USB to serial adapter UART. So, for example TX should go with RX and RX with TX.

```Using iterm2 as console to comm with the serial port ```

I use a MACos Catalina and i did not want to install any extra software since i got already Iterm2.
Create a new profile and specify a command there for whenever the session with this profile is opened.

screen /dev/cu.usbserial-0001 115200

![iterm](https://github.com/marioanton/marioanton.github.io/raw/master/img/iterm.png "iterm config profile")
> You will have to replace the device accordingly with the name of your UART usb to serial HW. Can figure this out by running: 

ls /dev/ \|  grep cu.usbserial

```Connect ethernet cable ```

Apart from connecting the cable to the laptop and router you will have to accomodate an ip like 192.168.1.3 and mask /24 not to clash with router IP that will be in place once switched on (192.168.1.1).

This has to be done on your laptop, in the network ethernet interface tied to the cable plugged in.


```Debricking your router```

Once soldering is done, profile is configured and cables are connected, we can start proceding with the debricking of the router wrt320n.

First Stage. Connecting to the router.

 1. Switch everything off.
 2. Connect USB to SERIAL device to your laptop. This should be connected with the route motheraboard serial interface.
 3. Connect ethernet cable.
 4. Press CTRL-C and hold that on while you switch on your router.

You should be seeing something like:
![boot](https://github.com/marioanton/marioanton.github.io/raw/master/img/boot.png "boot")

Second Stage. Erasing NVRAM

1. Run command: erase nvram
2. Run command: reboot and hold CTRL-C while rebooting.

Third Stage. Flashing.

1. Open a console and get located in the folder where the binary lives
1. run this command: tftp 192.168.1.1
2. run this command within tftp console: binary
3. run this command within serial console:  flash -ctheader : flash1.trx
3. run this command within tftp console:  put image.bin image.bin
4. run this command within serial console: reboot 

You should be seeing some like this:

![tftp](https://github.com/marioanton/marioanton.github.io/raw/master/img/tftp.png "tftp")

There are some errrors in this screenshot because:
- Image was incorrect one (CODE pattern Invalid error)
- Ethernet cable not connected / Router not switched on but in the end you can see image being uploaded correctly.

On this way, you have achieved to get the router back to life with a DD-WRT original image for this router and with all functionality. 
Switch it off and on to verify everything works as expected and restore your config and reset your password again :)

Hope you enjoyed it and got your wrt320n back to life. :)
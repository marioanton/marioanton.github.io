---
layout: post
title: debricking router linksys wrt320n
image: /img/redis-sentinel.png
tags: [wrt320,bricked]
---

I was messing around with my home network these days of lockdown and decided to use OpenVPn integrated within DD-WRT that was flashed to my Linksys wrt320N ages ago. The openvpn it came with had to be  reasonable old so i decided to give it whirl and upgrade the firmware. 
Firmware for upgrades can be found in here [Router Database](https://dd-wrt.com/support/router-database/?model=WRT320N_v1.0) 

That is how i ended up bricking my router. I went for the image `DD-WRT Broadcom Generic - NEWD K2.6 - Mega ` instead of `DD-WRT Broadcom Generic - NEWD K2.6 - Mini`

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

![](https://github.com/marioanton/marioanton.github.io/tree/master/img/pins.png "router serial comm pins interface")

You can see already the mapping of the pins for the serial interface and also can see where you can find the otehr pins in the WAN interface, where serial is possible to be done as well.

We DON'T need to connect 3.3v. Don't do it and you will not put your router in risk. We weill be using normal router power source.

> Don't have to say that you have to open the router. In the process of opening up your router you will have to remove 3 wires connected to the motherboard. They are antenna ones.

Once opened, solder this as the image states. The soldering wires for TX and RX should be linked to the opposite ones in the USB to serial adapter UART. So, for example TX should go with RX and RX with TX.

```Using iterm2 as console to comm with the serial port ```

I use a MACos Catalina and i did not want to install any extra software since i got already Iterm2.
Create a new profile and specify a command there for whenever the session with this profile is opened.

screen /dev/cu.usbserial-0001 115200

![](https://github.com/marioanton/marioanton.github.io/tree/master/img/iterm.png "iterm config profile")
> You will have to replace the device accordingly with the name of your UART usb to serial HW. Can figure this out by running: ls /dev/ | grep cu.usbserial

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
![](https://github.com/marioanton/marioanton.github.io/tree/master/img/boot.png "boot")

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

![](https://github.com/marioanton/marioanton.github.io/tree/master/img/tftp.png "tftp")

There are some errrors in this screenshot because:
- Image was incorrect one (CODE pattern Invalid error)
- Ethernet cable not connected / Router not switched on but in the end you can see image being uploaded correctly.








#### What is redis sentinel NOT used for

- Sentinel doesn't manage connections initiated by clients.  

#### What is redis sentinel used for

- Monitoring
- Alerting
- Redis Discovery
- Automate failover

**Monitoring and alerting** are basics, these allow us to monitor and using the sentinel api, to grab information regarding the status of redis instances.
**Redis Discovery or also called Service Discovery** let us use Redis Sentinel to discover config like what instance is master/slave and drive requests through them. This can be used by redis client with sentinel support for managing a HA scenario from app prespective.
**Automatic Failover** will do what the word says, failover a master to promote a slave to become master.

#### Achieving HA Redis

Sentinel will cope with it by tying itself against the redis instances to be monitored. Sentinel wil be managing slave / master instances and will act upon then health on them. There are several paramenters that can be configured to control how and when instances are failed over.

Sentinel instances monitor by themselves a set of redis instaces. The data that is retrieved by doing that monitoring is following:

- Which redis instance is master/slave
- Which sentinel server has got attached to.
- Redis instance status, like down, up etc.

All  Sentinel instances toghether (3 of them is ideal) must  achieve a quorum to take decisions, so with the data previously gathered, they communicate each other (sentinels) and vote for a decision and given that the number of sentinels  should be odd, there  should be no problem unless complex network partitions are taking place.

There is no config in place to let the sentinel instances to comm each other but that data is gathered from redis.

#### Achieving HA Redis from app prespective

In order to achieve HA is not enough with setting up Sentinel along with redis to manage the cluster properly. We gonna need to think of best way to drive requests to the right instance, for example, slaves for reading and master for writes. (Bear in mind syncronization among redis instances is async so read could be returning inconsistent/no up to date information)

| By mean of                     | Refer to |
| :---                           |   ----:  |
| Using Redis Client             | [Redis Clients](https://redis.io/clients)|
| Using alternative Ha methods   | [HaProxy](http://www.haproxy.org/)|


One example of **redis client** would be StackExchange.Redis. This will allow us have a highly available and meeting all required perfomance needs. Using multiplexer funcionalities allows us to pass a set of servers from which a master will be populated. This is not using redis Sentinel itself.

There are other clients that support usage of **Sentinel** to identify the master where connection should be made against. One example i could be thinking of is to use consul service discovery to hit sentinel and return the master instance where consul ready applications could be querying to.

Another way to set up a HA redis infra without using Sentinel itself would be using **Haproxy** where the there would be some rules for the haproxy backends  that would execute prechecks to deliver the  connectiong to the right server.

<pre>
  option tcp-check
  tcp-check connect
  tcp-check send AUTH\ password\r\n
  tcp-check expect string +OK
  tcp-check send PING\r\n
  tcp-check expect string +PONG
  tcp-check send info\ replication\r\n
  tcp-check expect string role:master
  tcp-check send QUIT\r\n
  tcp-check expect string +OK
  server host1 1.1.1.1:6379 check
  server host2 1.1.2.1:6379 check
  server host3 1.1.3.1:6379 check
</pre>

#### Sentinel and Redis facts

- **min-slaves-to-write** can be used to avoid critical situations when network paritions take place.  Clients could be writting to old masters, this would prevent it.
- Sentinel, Docker, or other forms of **Network Address Translation** or Port Mapping should be mixed with care
- In Sentinel, only masters are specified to be monitored, each one with a different name. Slaves will be self-discovered
- Sentinel **will not forget sentinel servers** in case they are decommissioned. They need to be forgotten by trigerring RESET command. Not resetting the snetinel config is useful, because Sentinels should be able to correctly reconfigure a returning slave after a network partition or a failure event. If not that should be done to have the right state of sentinel servers
- **SDOWN** sentinel status is Subjective Down and it is tied to a sentinel instance. On the contrary **ODOWN** means Objective Down which is status achieved by quorum of all sentinel instances.
- **Sentinel configuration is persisted to disk** so restarting the sentinel instance is safe to do so when restarting will pick up the most recent config in the files.

#### Sentinel and Redis useful commands

``` bash
redis-cli -p 26379 SENTINEL failover $redisInstance
```

>This will promote a slave of the current master as master and master will become slave of this one. This is achieved by internall calling redis  instance and issuing **slaveof** and **slaveof no one** commands

```bash
redis-cli -p 26379 SENTINEL reset $redisInstance/*
```

>This will reset redis sentinel known config and will be populated again within less than 10 seconds. Safe to do and required when old sentinel instances been decommisioned

```bash
redis-cli -p 26379  SENTINEL masters
```

> Show a list of monitored masters and their state

```bash
redis-cli -p 26379  slaveof $redisIsntace
```

> make a redis instance to be slave of given redis instance

```bash
redis-cli -p 6379  slaveof no one
```

> make a redis instance to be master

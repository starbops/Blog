================================================
 Make Your Raspberry Pi a Wireless Access Point
================================================

:date: 2015-09-14 16:08
:modified: 2015-09-21 14:47
:tags: raspberrypi, nat, iptables, hostapd, dnsmasq
:category: memo
:slug: make-your-raspberry-pi-a-wireless-access-point
:authors: Zespre Schmidt
:about_author: Bug generator
:email: starbops@gmail.com
:summary: I moved into dorm this semester again. There is only one Ethernet
          hole for me. But I have plenty devices that need to access the
          Internet! So I combine Raspberry Pi and a Wi-Fi USB adapter to build
          a Wireless access point.

Prerequisites
=============

This tutorial is based on two hardware which are listed below:

- Raspberry Pi model B
- Edimax EW-7811Un Wi-Fi nano USB adapter

First you need to make sure the Raspberry Pi is running Raspbian. If not,
download the zip file from the official website and unzip it:

.. code-block:: bash

    $ unzip 2015-05-05-raspbian-wheezy.zip

Insert the SD card which will be inserted into the Raspberry Pi into your
computer. Here, I use Macbook Air as example. Locate the SD card:

.. code-block:: bash

    $ diskutil list

In my case, ``/dev/disk2`` is the SD card. Then try to unmount the SD card
(whole disk, not specific partition) and write the image which we just
downloaded and unzipped into it:

.. code-block:: bash

    $ diskutil unmountDisk /dev/disk2
    # dd if=2015-05-05-raspbian-wheezy.img of=/dev/rdisk2 bs=1m

Now the SD card is ready. Insert the card into the Raspberry Pi. Make sure the
Ethernet cable and Wi-Fi USB adapter is connected, too. It's time to turn on
the power of the Pi!

Configuring Network
===================

There are two network interfaces need to be configured:

- eth0
- wlan0

The wired one, i.e. eth0, is already configured, so we only need to setup
wlan0. This interface is acted as a default gateway of our wireless devices
that will connect with later. The configuration file is
``/etc/network/interfaces``:

.. code-block:: text

    iface wlan0 inet static
    address 10.0.0.1
    network 10.0.0.0
    netmask 255.255.255.0
    broadcast 10.0.0.255

To prevent the network interface settings being overwritten by DHCP server, we
need to wake up the interface manually by inserting one line in
``/etc/rc.local``:

.. code-block:: bash

    ...
    ifup wlan0
    ...
    exit 0

Also, the Raspberry Pi is a router in our scenario. It must have the ability to
forward packets, i.e. receive the packets which are not destined to itself and
lookup the routing table then forward them. Add this line in
``/etc/sysctl.conf`` to enable this when every time the Pi boots up:

.. code-block:: text

    net.ipv4.ip_forward=1

Now we make the Pi receive the packets, the next important thing is NAT. The
NAT function is done by iptables:

.. code-block:: bash

    # iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    # iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
    # iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT

To enable NAT function when the Pi boots up every time, the rules of iptables
need to be dumped to an file for later use:

.. code-block:: bash

    # iptables-save > /etc/iptables.rules

Let iptables read the rules from the file we created above before the network
interface starts up. Write a simple shell scirpt in
``/etc/network/if-pre-up.d/iptables``:

.. code-block:: bash

    #!/bin/sh
    /sbin/iptables-restore < /etc/iptables.rules && exit 0

And finally set the execution bit up for the script:

.. code-block:: bash

    # chmod a+x /etc/network/if-pre-up.d/iptables

HostAPD
=======

HostAPD is a user space daemon for access point and authentication servers.
Install it from `apt-get`:

.. code-block:: bash

    # apt-get install hostapd

But the apt hosted HostAPD is a little buggy. It is not compatible with the
RTL8188CUS chipset. Luckily, thanks to the Edimax team, they provide a patched
version of it on the Internet. Download the patched HostAPD here_.

.. code-block:: bash

    $ unzip 0001-RTL8188C_8192C_USB_linux_v4.0.2_9000.20130911.zip
    $ cd RTL8188C_8192C_USB_linux_v4.0.2_9000.20130911/wpa_supplicant_hostapd/
    $ tar -zxvf wpa_supplicant_hostapd-0.8_rtw_r7475.20130812.tar.gz
    $ cd wpa_supplicant_hostapd-0.8_rtw_r7475.20130812/hostapd/
    $ make
    # make install

Replace the ``hostapd`` binary with the newly compiled one:

.. code-block:: bash

    # mv /usr/sbin/hostapd /usr/sbin/hostapd.bak
    # cp /usr/local/bin/hostapd /usr/sbin/hostapd.edimax
    # ln -s /usr/sbin/hostapd.edimax /usr/sbin/hostapd

Configure the HostAPD by editing ``/etc/hostapd/hostapd.conf``. Remember to
replace the "ssid" and "wpa_passphrase" for your own needs:

.. code-block:: text

    interface=wlan0
    driver=rtl871xdrv
    ssid=my_wife
    hw_mode=g
    channel=6
    macaddr_acl=0
    auth_algs=1
    ignore_broadcast_ssid=0
    wpa=2
    wpa_passphrase=00000000000000
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=TKIP
    rsn_pairwise=CCMP

To make the HostAPD start at system boot, edit ``/etc/default/hostapd``:

.. code-block:: text

    DAEMON_CONF="/etc/hostapd/hostapd.conf"

Dnsmasq
=======

Last part of the tutorial. The wireless devices will need IP addresses to be
able to access the network. Then DHCP server comes!

.. code-block:: bash

    # apt-get install dnsmasq

The DHCP server will start up when it is installed. You need to stop it to edit
the configuration file:

.. code-block:: bash

    # service dnsmasq stop
    # mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
    # touch /etc/dnsmasq.conf

Basic configuration:

.. code-block:: text

    interface=wlan0
    expand-hosts
    domain=local
    dhcp-range=10.0.0.10,10.0.0.20,24h
    dhcp-option=3,10.0.0.1

Annnnnnnnnnnnnnnnd we are all set! Try to start the DHCP server or reboot the
Raspberry Pi.

References
==========

- `Turn Your Raspberry Pi Into a WiFi Hotspot with Edimax Nano USB EW-7811Un
  (RTL8188CUS chipset)`__
- `Using you Raspberry Pi as a Wireless Router and Web Server`__
- `RPI-Wireless-Hotspot`__
- `Setting Up WiFi Access Point with Edimax EW-7811UN on Raspberry Pi`__

.. __: http://www.daveconroy.com/turn-your-raspberry-pi-into-a-wifi-hotspot-with-edimax-nano-usb-ew-7811un-rtl8188cus-chipset/
.. __: http://www.daveconroy.com/using-your-raspberry-pi-as-a-wireless-router-and-web-server/
.. __: http://elinux.org/RPI-Wireless-Hotspot
.. __: https://ariandy1.wordpress.com/2013/04/07/setting-up-wifi-access-point-with-edimax-ew-7811un-on-raspberry-pi/
.. _here: http://www.realtek.com.tw/downloads/downloadsView.aspx?Langid=1&PNid=21&PFid=48&Level=5&Conn=4&DownTypeID=3&GetDown=false


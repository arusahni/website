.. link: 
.. tags: gadgets, chromecast
.. date: 2013-08-25 19:45:00
.. slug: chromecast-and-ubuntu
.. title: Capabale Chromecast Configuration (or, how I got Chromecast to work with Ubuntu)
.. description: In which I get my Chromecast to play nicely with my software firewall.
.. comments: true

So, I now own a Chromecast. Having acquired it while the Netflix promotion was in effect, I only ended up spending $11 on it.  Not bad for a YouTube machine (the only reason I bought it since Roku still doesn't have YouTube support).  I've been pretty happy with it thusfar as it turned out to be a great party tool, and the HDMI CEC support makes it really easy to throw YouTube videos on in the background, even if the TV is off.

However, one feature I was having trouble getting to work under Linux was tab casting.  Using the `Chromecast Chrome extension <https://chrome.google.com/webstore/detail/google-cast/boadgeojelhgndaghljhdicfkmllpafd>`_, one should be able to transmit the contents of a tab to a Chromecast on the local network. For some, reason, though, I was unable to do so.  After some poking around, I determined that my firewall (ufw, the greatest misnomer of them all) was causing this.  Since it was using a randomly assigned local port to establish the connection, simply opening a port wasn't the answer.  Not to be deterred, I set about conquering this problem. Here's what I did.

#. **Find the Chromecast's MAC address**. In my case this was a two-step operation... \

   #. Launch the Chromecast app to determine the device's IP address. \
   
      .. figure:: http://i.imgur.com/z4nhh4T.png?2
         :alt: A screenshot of the IP address of a Chromecast desplayed within the Chromecast app.
         :target: http://imgur.com/z4nhh4T
   
         The IP address is circled in red.

   #. Ping the device and then check the address resolution stats.\

      .. code:: bash
  
       [aru@Ananke:~]$ ping -c 1 192.168.1.42
       PING 192.168.1.42 (192.168.1.42) 56(84) bytes of data.
       64 bytes from 192.168.1.42: icmp_req=1 ttl=64 time=1.66 ms
       
       --- 192.168.1.42 ping statistics ---
       1 packets transmitted, 1 received, 0% packet loss, time 0ms
       rtt min/avg/max/mdev = 1.669/1.669/1.669/0.000 ms
   
       [aru@Ananke:~]$ arp -a 192.168.1.42
       ? (192.168.1.42) at d0:e7:82:7c:15:76 [ether] on wlan0

#. **Assign a static IP to the device**. Configure your router to assign a static IP to the MAC address you found in the previous step. If you don't know how to do this, `Port-Forward.com has some decent tutorials <http://portforward.com/english/routers/port_forwarding/>`_.  You may have to restart your router after setting up this rule.

#. **Determine your local port range**.  In my case, since I'm using IPv4, I did the following: \
   
   .. code:: bash
   
    [aru@Ananke:~]$ cat /proc/sys/net/ipv4/ip_local_port_range
    32768   61000
    
   The two numbers are the lower and upper bounds of the local ports available for casting.
   
#. **Whitelist your Chromecast**. Using the now-static IP address and your local port ranges, tell ufw to step off!\

   .. code:: bash
   
    [aru@Ananke:~]$ sudo ufw allow proto udp from 192.168.1.42 to any port 32768:61000
    Rule added
   
Once you've done this, open up Chrome, click on the Chromecast button, and start tabcasting!

.. figure:: http://i.imgur.com/clvTWQB.png
   :alt: Chromecast extension screenshot.
   :target: http://imgur.com/clvTWQB
   
   As you can see, I don't use Chrome for very much.

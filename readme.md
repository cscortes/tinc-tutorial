
# Setting up Tinc VPN on Linode



## What is Tinc?

According to Tinc's own documentation:

>tinc is a Virtual Private Network (VPN) daemon that uses tunnelling and encryption to create a secure private network between hosts on the Internet. tinc is Free Software and licensed under the GNU General Public License version 2 or later. Because the VPN appears to the IP level network code as a normal network device, there is no need to adapt any existing software. This allows VPN sites to share information with each other over the Internet without exposing any information to others. 

Of course, to install the latest software version of tinc, one can pull the source code from their [download page](https://www.tinc-vpn.org/download/). An easier way may be provided by your operating system.  For this example, we will use Fedora 30 and use the built-in installer to grab the necessary tinc binaries.


Since system administration will be part of this installation, referencing 
guides on [Introduction to Linux Concepts](https://www.linode.com/docs/tools-reference/introduction-to-linux-concepts) and
[Linux System Administration Basics](https://www.linode.com/docs/tools-reference/linux-system-administration-basics) might come in handy.

## Before You Begin

1.  Familiarize yourself with our [Getting Started](/docs/getting-started) guide, and complete the steps for setting up your Linode's hostname and timezone.  Our Linode system will have a hostname of **oxygen**.

2.  This guide will use `sudo` wherever possible. Complete the sections of the Guide's [Securing Your Server](/docs/security/securing-your-server) section to create a standard user account, harden SSH access, and remove unnecessary network services. Do **not** follow the Configure a Firewall section yet--this guide includes firewall rules specifically for a tinc server.

3.  Obtain a client system (laptop or computer) with fedora 30 workstation OS installed.  Our client system will be called **carbon**.

> The real world IP addresses we will use are completely made up.  They are provided to give you a reference for the configruation files.  Do not use them for your own server or client. 
 
Please replace 11.22.33.10/8 with the real world ip address of your Linode server.  Also, replace 11.22.33.12/8 with the real world ip address of your wireless router.  

> The assumption for your real world IP address for your wireless router is that you have a static IP address.  Otherwise, you need to find out what has been assigned to your client machine by a dhcp server everytime and reconfigure your tinc host file on your Linode server.


## The Target Scenario


Our aim is to setup tinc on our linode server in the cloud and then make at least one vpn connection using tinc on a fedora 30 client. To add a little bit of complexity, our client using tinc is behind a wireless router.  The wireless router should be able to obtain a real world ip address from the Internet and our client is assigned a NAT address from our wireless router.  This configuration would approximate a real world scenario.

![scenario network layout](tinclayout.png)


## Configuration


### Install Package Dependencies

1.  Log into your server machine.  Then make sure your fedora 26 OS is up to date:

        sudo dnf update

        sudo dnf upgrade

2.  Install tinc, only 1 package is needed for a server or a client:

        sudo dnf install tinc


###  Configuration files for Server

1.  We are going to create the directory, configuring all the way down to a netname directory.  Normally, you would pick a netname, and create a directory under your tinc folder with that same name.  We will do this all in 1 step.  Assuming that our netname is "myvpn", we will create all necessary folders with the following command:

        sudo mkdir -p /etc/tinc/myvpn

2.  There are 2 files that control your vpn interfaces.  For a basic setup, we can use the same instructions from the sample tar file in the tinc share directory.  These interfaces are so easy. I will show you how to create them with the cat command:

        sudo cat > /etc/tinc/myvpn/tinc-down 
        #!/bin/sh
        ifconfig $INTERFACE down
        CNTL-Z

        sudo cat > /etc/tinc/myvpn/tinc-up
        #!/bin/sh 
        ifconfig $INTERFACE 192.168.11.1 netmask 255.255.255.0
        CNTL-Z

3. After you have created these files, make sure the permissions are correctly set for the execution of these 2 files:

        sudo chmod 755 /etc/tinc/myvpn/tinc-*

4. For the server, we will only have 2 entries in this file.  One will be the hostname of your server machine, and the other will be the device name:

        sudo cat > /etc/tinc/myvpn/tinc.conf
        Name = oxygen
        Device = /dev/net/tun
        CNTL-Z

5. Set the hostname of your server.  This is normally not necessary as Linode provides you with a name.  Although, that name is not as memorable as one you give it yourself.

        sudo hostnamectl set-hostname oxygen
        sudo cat >> /etc/hosts
        11.22.33.10     oxygen
        CNTL-Z

6.  To find out your IP address, just type `ifconfig` and on the eth0 line, you will see inet followed by your real world IP address.  Something like this:

eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500  
        inet **11.22.33.10**  netmask 255.255.255.0  broadcast 11.22.33.255  
        ...

7.  Create a node name file for your server.  There are 3 variable that we will want to set, one is not really needed since it will default to port 655 if you don't specify it.  We will put it in anyway, since we can forget 3 months from now.  The other 2 are the real world ip address and the VPN ip address of your server. The Subnet tag will set your server's VPN ip address and that is the one you need to use for encrypted communication to your server.  

        sudo cat > /etc/tinc/myvpn/hosts/oxygen
        Address = 11.22.33.10
        Subnet = 192.168.11.1/32
        Port = 655
        CNTL-Z

>The subnet ip should match the one you set in your tinc-up file.  If you change one, you need to change the other.

1.  Create public and private keys for your server.  Tinc makes this easy with the following command:

        sudo tincd -n myvpn -K

It will add your public key to your oxygen host file, which we will need to transfer at a later time to the client machine.  

9. Add firewall rules to let the client connect to this server. 

		sudo firewall-cmd --add-service=tinc
		sudo firewall-cmd --permanent --add-service=tinc


### Configuration Files for Client


1.  Log in to your client machine.  We are going to create the directory all the way down to a netname directory.  I recommend using the same netname as the one you used for the server.  Assuming that we are using the same netname "myvpn", we create all necessary folders with the following command:

        sudo mkdir -p /etc/tinc/myvpn

2.  There are 2 files that control your vpn interface for your client machine (note that the VPN IP address is different from the one on the VPN IP address for the server):

        sudo cat > /etc/tinc/myvpn/tinc-down 
        #!/bin/sh
        ifconfig $INTERFACE down
        CNTL-Z

        sudo cat > /etc/tinc/myvpn/tinc-up
        #!/bin/sh 
        ifconfig $INTERFACE 192.168.11.2 netmask 255.255.255.0
        CNTL-Z

3. After you have created these files, make sure the permissions are correctly set for execution of these 2 files:

        sudo chmod 755 /etc/tinc/myvpn/tinc-*

4. For the client, we will have 3 entries in the tinc.conf file. The name of your client machine, where to connect to (your server machine) and the device name:

        sudo cat > /etc/tinc/myvpn/tinc.conf
        Name = carbon
        ConnectTo = oxygen
        Device = /dev/net/tun
        CNTL-Z

5. Set the hostname of your client, but add the real world IP address of the server to the hosts file.

        sudo hostnamectl set-hostname carbon
        sudo cat >> /etc/hosts
        11.22.33.10     oxygen
        CNTL-Z

6. To find your real world IP address on your client machine, you can bring up a browser and hit a website like [whatismyipaddress.com](http://whatismyipaddress.com).  Just to follow my example, let us say that your real world IP address is 11.22.33.12.


7.  Create a node name file for your client.  There are 3 variable that we will want to set, one is not really needed since it will default to port 655 if you don't specify it.  We will put it in anyway, since we can forget 3 months from now.  The other 2 are the real world ip address and the VPN ip address of your client machine. The Subnet tag will set your client's VPN ip address and that is the one you need to use for encrypted communication to your server.  

        sudo cat > /etc/tinc/myvpn/hosts/carbon
        Address = 11.22.33.12
        Subnet = 192.168.11.2/32
        Port = 655
        CNTL-Z

> The subnet ip should match the one you set in 
your tinc-up file.  If you change one, you need to change the other.

1.  Create public and private keys for your client machine.  Tinc makes this easy with the following command:

        sudo tincd -n myvpn -K

    It will add your public key to your carbon host file, which we will need to transfer at a later time to the server machine.


2. Add firewall rules to let the server connect to this client. 

		sudo firewall-cmd --add-service=tinc
		sudo firewall-cmd --permanent --add-service=tinc
    

## Exchange Public Keys Via Host Files

1.  Log into your client machine.  The general idea is that each machine has a correctly configured host file with a public key.  We need a copy one for the server of the client's host file and vis-versa so that each system recognizes the other.  


    This can be accomplished by using scp to copy the server's host file down to the client.

        sudo scp oxygen:/etc/tinc/myvpn/hosts/oxygen /etc/tinc/myvpn/hosts

    And the client's up to the server.

        sudo scp /etc/tinc/myvpn/hosts/carbon  oxygen:/etc/tinc/myvpn/hosts/

    
    ## The Wireless Router Configuration

    The wireless router configuration was necessary because this would simulate an actual setup you might have at home.  Otherwise, you would already be done.  Assuming you have a router with _port forward_ feature, which most do.  
    
    In fact, when I tested this configuration, I did it using my android phone's hotspot and an app called "Fwd: the port forwarding app".  
    
    What you want to have in mind is that the wireless router sits between your server and client machines.  The real world IP of your client is really assigned to the wireless router.  All the wireless router has to do is forward any connections to a port from it to the client.  
    
    In my particular case, I couldn't forward a port between 1-1024 on my phone (because those ports are restricted), so I just picked one outside of the restricted range like 40655 and forwarded traffic to my client's port 655.  I did have to make some adjustments to my conf files.

1.  Log in to the server and make the following change to the client host file, in our case it is called carbon.  Port = 40655. 

2.  Add to the wireless router a new forwarding rule.  Make sure you make one for both udp and tcp traffic to forward all traffic from port 40655 to your client machines NAT IP address.  In our layout that would be 192.168.1.5.  This is how I did it on the app on my phone. Remember, my phone was for my personal testing, not a permanent solution.  Please use your wireless router and configure it accordingly.

![The fwd: app tinc configuration](fwdconfig.png)

##  Testing Your Setup

1.  Log in to your server, and manually run tinc like so:

        tincd -n myvpn -d5 -D

2. Log in to your client machine, and manually run tinc with the same command:

        tincd -n myvpn -d5 -D

This time though, you should see a bunch of data flying up on your terminal screen.  One should be a line that starts with "Sending METAKEY to oxygen ..." and another "Got METAKEY from oxygen ...".  Basically, trying to do a handshake and make sure that they other machine is who it says it is.  

3. Finally, ping the VPN IP address of the other machine, if you are logged into the client, try:

        ping 192.168.11.1 

If you are on the server machine, try:

        ping 192.168.11.2

If you get ping results, you did it!

## Troubleshooting

This is a complex setup.  You are doing something that should be done with a little patience since many things could go wrong.  Remember only your VPN ips are encypted.

1. **Double check your IPs.**  I know there are a lot of them, but make sure you understand the different between real world IPs, NAT IPs and VPN IPs.  Study the network layout and redo it for your own layout.  This might help you figure out which one is faulty.

2. **ID Mismatch.**  Watch out for this one.  This is where I forgot to run the public/private generation on both the client and server machine. OR I ran it twice, but forgot to update the version on the other machine.  Also, when you add another client, only update the public/private keys for that client.  You don't need to regenerate the public/private keys for your server or first client.

3.  **If in doubt, try it without a wireless router.** Do a simple setup without the port forwarding.  Your machines should be setup next to each other with only a switch to traffic your network data.  This is the easiest setup, and will help you understand things before you add a step of complexity which is the wireless configuration.

4. **Consult tinc documentation.**  It has several more troubleshooting steps you could try.


## Permanently Adding a Tinc Service


1. Once you are satified with your tinc server machine.  You can make it a permanent service with the following commands:

        sudo systemctl enable tinc@myvpn

2. On linode though, there is one change you have to make, move the service file from the tinc.service.wants directory to the multi-user.target.wants directory.

        sudo mv /etc/systemd/system/tinc.service.wants/tinc@myvpn.service /etc/systemd/system/multi-user.target.wants

3.  Reboot your server and your new service should be up and running!








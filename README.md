# stop t-mobile throttling!

Here's how I used a raspberry pi to overcome T-mobile throttling. This will enable you to use your home router with many devices connected behind it to access the internet through your T-mobile sim card, without experiencing throttling.

## How t-mobile throttles currently
TMobile uses the time-to-live value of packets to determine if they have been routed through a phone or originate from the phone itself. To circumvent this, you want your tethered traffic to have the same TTL as phone traffic. The idea is to tether a device capable of overwriting TTL and set it to +1 over what you expect the phone's TTL to be, so that when it is routed by the phone and the TTL is decremented by 1 it is then the expected value.

Most phones have a TTL of 64. This means we need our tethered device's TTL to be 65, so that when it is decremented by passing through the phone it has the identical value of 64 and cannot be differentiated.

## Raspberry pi to the rescue
We'll be configuring the pi to act as a wifi to ethernet bridge and then using IP Tables to overwrite the TTL on each packet. 

This tutorial assumes you've installed jesse and you're logged into the console (e.g. not ssh'd)

    sudo apt-get update && sudo apt-get upgrade -y && sudo apt-get install rpi-update dnsmasq iptables-persistent -y && sudo rpi-update

# connect pi wifi to your t-mobile hotspot

I suppose you could just turn on the wifi hotspot on your android phone.  I found it more reliable to pick up a hotspot device. I'm using an unlocked version of the AT&T Unite Explore I got on amazon for $150 (make sure it's unlocked as in "not locked to a carrier." -- Don't ask me how I know.. thank you amazon return policy).  

    sudo nano /etc/wpa_supplicant/wpa_supplicant.conf

Add your network information like this: 

    network={ ssid="mynetwork" psk="secret" key_mgmt=WPA-PSK }

Give the ethernet connection a static ip:

    sudo nano /etc/network/interfaces

remove or comment out the following line:

    #iface eth0 inet manual

and add the following:

    allow-hotplug
    eth0 iface eth0
    inet static address 172.24.1.1
    netmask 255.255.255.0
    network 172.24.1.0
    broadcast 172.24.1.255




## Setting up dnsmasq

dnsmaq will bridge our wifi and ethernet connections.  
let's backup what's there:

    sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
    sudo nano /etc/dnsmasq.conf
Now add the following:

    interface=eth0 # Use interface eth0 
    listen-address=172.24.1.1 # Explicitly specify the address to listen on 
    bind-interfaces # Bind to the interface to make sure we aren't sending things elsewhere 
    server=8.8.8.8 # Forward DNS requests to Google DNS 
    domain-needed # Don't forward short names 
    bogus-priv # Never forward addresses in the non-routed address spaces. 
    dhcp-range=172.24.1.50,172.24.1.150,12h # Assign IP addresses between 172.24.1.50 and 172.24.1.150 with a 12 hour lease time



## enable IPv4 forwarding

    sudo nano /etc/sysctl.conf

Uncomment the following line:

    net.ipv4.ip_forward=1

## IP Tables
(This is where the magic happens)

    sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
    sudo iptables -t mangle -A POSTROUTING -j TTL --ttl-set 65

We need these to persist after reboot so let's use the package we installed earlier to save them:

    sudo iptables-persistent save

Go ahead and reboot.  

    sudo reboot
if all worked well you can now plug an ethernet cable between your raspberry pi and the WAN port on your home router. All you devices should be using your T-mobile connection without being throttled.

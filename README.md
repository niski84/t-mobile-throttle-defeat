# stop t-mobile throttling!

Here's how to use a Raspberry Pi to defeat T-Mobile hotspot throttling. This lets you use your home router — with many devices behind it — over a T-Mobile SIM without being throttled.

> **Note:** This may violate T-Mobile's Terms of Service. This guide is for educational purposes only.

---

## How T-Mobile throttles

T-Mobile inspects the **TTL (Time-To-Live)** field of IP packets to detect tethered traffic. Traffic originating from the phone itself has a TTL of 64. Tethered/routed traffic passes through the phone, which decrements the TTL by 1 — making it 63 — and T-Mobile flags anything below 64 as hotspot traffic subject to throttling.

**The fix:** Force all outgoing packets to TTL 65. When they pass through the phone (TTL −1), they exit at 64 — indistinguishable from native phone traffic.

---

## Hardware options

The Pi sits between your modem/hotspot and your home router. Two common setups:

### Option A — WiFi hotspot or phone tethering
```
[T-Mobile hotspot / phone]
          |  WiFi (wlan0)
    [Raspberry Pi]
          |  Ethernet (eth0)
   [Home router WAN]
```

### Option B — USB or Ethernet LTE modem (more reliable for permanent installs)
```
[LTE modem e.g. Netgear LB1121]
          |  Ethernet (eth0)
    [Raspberry Pi]
          |  USB-to-Ethernet (eth1)
   [Home router WAN]
```

---

## Step 0 — Detect your interfaces and subnet

Before configuring anything, run these commands on the Pi to identify your interface names and modem subnet:

```bash
# List all interfaces and their current addresses
ip -br addr show

# See which interface has a default route (this is your WAN/modem side)
ip route show

# If using a USB modem or USB-to-Ethernet adapter, identify it
lsusb
dmesg | grep -E "eth|usb|wlan" | tail -20
```

Use this output to fill in the variables in `setup.sh`.

**Common interface names:**

| Interface | Typical use |
|---|---|
| `wlan0` | Built-in WiFi (connecting to a WiFi hotspot) |
| `eth0` | Built-in Ethernet |
| `eth1` | USB-to-Ethernet adapter |
| `usb0` | USB tethering directly from a phone |
| `enxXXXXXXXXXXXX` | Some USB Ethernet adapters use MAC-based names |

---

## Step 1 — Configure and run `setup.sh`

Edit the variables at the top of `setup.sh` for your hardware:

```bash
nano setup.sh
```

| Variable | Description | Example |
|---|---|---|
| `WAN_IFACE` | Interface facing the modem/hotspot | `wlan0`, `eth0` |
| `LAN_IFACE` | Interface facing your home router | `eth0`, `eth1` |
| `LAN_IP` | Static IP for the Pi on the LAN side | `192.168.6.1` |
| `LAN_SUBNET` | Subnet for DHCP | `192.168.6.0` |
| `LAN_NETMASK` | Netmask | `255.255.255.0` |
| `DHCP_START` | Start of DHCP range | `192.168.6.50` |
| `DHCP_END` | End of DHCP range | `192.168.6.150` |
| `DNS_SERVER` | Upstream DNS | `1.1.1.1` or `8.8.8.8` |
| `TTL_VALUE` | TTL to force on outgoing packets | `65` (don't change) |

Then run it:

```bash
chmod +x setup.sh
sudo ./setup.sh
```

---

## Step 2 — WiFi hotspot configuration (Option A only)

If `WAN_IFACE` is `wlan0`, connect it to your hotspot:

```bash
sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
```

Add:

```
network={
    ssid="YourHotspotName"
    psk="YourPassword"
    key_mgmt=WPA-PSK
}
```

Reboot, then plug an Ethernet cable from the Pi into your home router's WAN port. All devices on the router should now use the T-Mobile connection without throttling.

---

## Manual setup (legacy reference — Debian Jessie)

> The steps below are the original manual instructions for **Debian Jessie**. For modern Raspberry Pi OS (Bookworm/Bullseye), use `setup.sh` instead. Kept here for historical reference.

### Install packages

```bash
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install dnsmasq iptables-persistent -y
```

### Static IP (`/etc/network/interfaces`)

```bash
sudo nano /etc/network/interfaces
```

Comment out:
```
#iface eth0 inet manual
```

Add:
```
allow-hotplug eth0
iface eth0 inet static
  address 172.24.1.1
  netmask 255.255.255.0
  network 172.24.1.0
  broadcast 172.24.1.255
```

### dnsmasq

```bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
sudo nano /etc/dnsmasq.conf
```

```
interface=eth0
listen-address=172.24.1.1
bind-interfaces
server=8.8.8.8
domain-needed
bogus-priv
dhcp-range=172.24.1.50,172.24.1.150,12h
```

### Enable IPv4 forwarding

```bash
sudo nano /etc/sysctl.conf
# Uncomment: net.ipv4.ip_forward=1
```

### iptables

```bash
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -t mangle -A POSTROUTING -j TTL --ttl-set 65

# Persist rules (Jessie used iptables-persistent; modern OS uses netfilter-persistent)
sudo iptables-persistent save
```

---

## Troubleshooting

**No internet after reboot:**
```bash
sudo systemctl status dnsmasq
sudo iptables -t nat -L -v
sudo iptables -t mangle -L -v
```

**Check TTL is actually being set:**
```bash
sudo tcpdump -i <WAN_IFACE> -n 'icmp' -v | grep ttl
```

**iptables rules not persisting:**
```bash
sudo systemctl enable netfilter-persistent
sudo netfilter-persistent save
```

**Find modem gateway IP:**
```bash
ip route show | grep default
```

+++
title = "AP+STA mode on Raspberry Pi 5"
date = 2025-09-06

[taxonomies]
tags = ["raspberry pi"]
+++

If you've built any headless Raspberry Pi projects, you probably know that one of your main concerns is how to communicate with them remotely. This is especially true for robots, where you may not be around any known networks, [or maybe even any at all](https://www.youtube.com/shorts/vv3AigEM6fE).

A fairly common solution I've noticed is to build a [so-called "travel router"](https://www.amazon.com/GL-iNet-GL-AR300M16-Ext-Pre-Installed-Performance-Programmable/dp/B07794JRC5) into the robot, and this is the approach I took with my outdoor robot _Kybernetes 2_. The robot itself serves as a wifi extender - the internal Raspberry Pi acts as a router between the onboard network and a set of pre-defined wifi networks it's aware of, such as my home wifi and my phone's hotspot. It doesn't _have_ to be connected to an upstream network, but if it is, devices that are connected to the robot are able to reach the internet. However, the downside is that I now have another device consuming power and I had to squirrel it away somewhere in my robot.

## A Little-Known Feature

If you've spent any time with the ESP32, you may have bumped into a mode called "AP+STA" or "Access Point plus Station". Conceptually, this means a device _both_ connects to a wifi network as a client (station) _and_ serves a network for other devices to connect to (access point), but when talking about "AP+STA", this typically refers to being able to do this with a single radio. It's commonly used to host configuration portals on IoT devices so that a user can configure the upstream network connection without needing to physically interact with the device.

As it turns out, the wifi chipset in the Raspberry Pi 4 and 5 (the CYW43455) is capable of operating in "AP+STA" mode with some caveats - the caveat being that it can't operate in both the 2.4 GHz and 5 GHz bands simultaenously. (e.g. if the upstream network is 5 GHz, the downstream network must also be 5 GHz)

The 5 GHz bandwidth on the Pi 5 is actually quite remarkable, I was able to achieve nearly 300 megabit between my laptop and the Pi when in the same room.

## The Setup

So let's actually set this up on a Raspberry Pi 5.

The following instructions have been tested with a clean install of Ubuntu Server 24.04, Ubuntu Desktop 24.04, and Raspberry Pi OS Bookworm. It'll create a downstream wifi network that's bridged via NAT to an upstream wifi network. The onboard ethernet port will also be placed on the same virtual bridge as the downstream network to service any internal ethernet devices on the robot. (Teensy 4.1, etc.)

Please note that after the creation of the downstream AP there are two separate guides - one for Ubuntu **Server** 24.04, as it uses Canonical's `netplan` on top of `systemd-network` to configure networking, and a second for Ubuntu **Desktop** 24.04 and Raspberry Pi OS, which both use `NetworkManager`.

## Install Dependencies

```bash
sudo apt install dnsmasq hostapd iw
```

## Configure Downstream WiFi Access Point

The first thing we're going to do is create a systemd service to automatically create the access point virtual device on the internal wifi radio when the system boots. Create a file named `/etc/systemd/system/wifi-ap-interface@.service` with the following contents:

### A systemd service to create the AP

```ini
# /etc/systemd/system/wifi-ap-interface@.service
[Unit]
Description=Create WiFi AP virtual interface %i
After=network.target
Before=hostapd.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/iw dev wlan0 interface add %i type __ap
ExecStop=/usr/sbin/iw dev %i del
StandardOutput=journal

[Install]
WantedBy=multi-user.target
```

Now enable the service to create the virtual AP called `ap0` when the system boots.

```bash
systemctl enable wifi-ap-interface@ap0.service
```

### Configure access point service

Use the `wpa_passphrase` command to generate the pre-shared key (PSK) for the network we're creating. Replace `<enter your ssid>` with the network name you want the Raspberry Pi to host.

```bash
# Get the PSK based on your SSID and the password you enter
wpa_passphrase "<enter your ssid>"
```

Create a file called `/etc/hostapd/hostapd.conf` with the following contents, replacing `<enter your ssid>` with the same SSID you used with `wpa_passphrase`.

```ini
# /etc/hostapd/hostapd.conf
interface=ap0
bridge=br0
driver=nl80211
ssid=<enter your ssid>
hw_mode=a
channel=36
country_code=US

# Enable 802.11n with basic capabilities
ieee80211n=1
ht_capab=[HT40+][SHORT-GI-20][SHORT-GI-40]

# Enable 802.11ac with basic capabilities
ieee80211ac=1
vht_capab=[SHORT-GI-80]
vht_oper_chwidth=1
vht_oper_centr_freq_seg0_idx=42

# Modern WPA settings
wpa=2
wpa_psk=<enter your psk>
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
```

Now enable the service to start the downstream wifi access point when the system boots.

```bash
systemctl enable hostapd.service
```

## Ubuntu Server 24.04

### Configure upstream WiFi networks via `netplan`

Use `wpa_passphrase` to determine the pre-shared keys for any upstream networks you want the Raspberry Pi to be able to connect to.

```bash
wpa_passphrase "SSID 1"
wpa_passphrase "SSID 2"
```

Edit `/etc/netplan/50-cloud-init.yaml` to contain the upstream networks you want to be able to connect to. As far as I'm aware, the connection priority is the order that the networks are defined.

```yaml
# /etc/netplan/50-cloud-init.yaml
network:
  version: 2
  wifis:
    wlan0:
      optional: true
      dhcp4: true
      access-points:
        "SSID 1":
          auth:
            key-management: "psk"
            password: "enter SSID 1's PSK"
        "SSID 2":
          auth:
            key-management: "psk"
            password: "enter SSID 2's PSK"
```

Apply the changes we made to the `netplan` configuration.

```bash
sudo netplan apply
```

### Create a network bridge for downstream network

Create a file called `/etc/systemd/network/20-br0.netdev` with the following contents to create a network bridge called `br0`.

```ini
# /etc/systemd/network/20-br0.netdev
[NetDev]
Name=br0
Kind=bridge
```

Create a file called `/etc/systemd/network/20-br0.network` with the following contents to set up NAT from `br0` to any upstream network interfaces.

```ini
# /etc/systemd/network/20-br0.network
[Match]
Name=br0

[Network]
Address=192.168.4.1/24
IPForward=yes
IPMasquerade=ipv4
```

Create a file called `/etc/systemd/network/25-eth0.network` with the following contents to add `eth0` to the `br0` bridge.

```ini
# /etc/systemd/network/25-eth0.network
[Match]
Name=eth0

[Network]
Bridge=br0
```

### Set up DHCP and DNS for downstream network

While `systemd-networkd` has support for enabling a DHCP server for a downstream interface, it doesn't provide DNS server support. We don't want our clients to have to rely on upstream DNS, so we'll use `dnsmasq` to provide both DHCP and DNS.

```ini
# /etc/dnsmasq.d/br0.conf
interface=br0
bind-dynamic
dhcp-range=192.168.4.100,192.168.4.199,255.255.255.0,24h
port=53
resolv-file=/etc/resolv.conf
cache-size=300
```

Enable `dnsmasq` to run when the system boots.

```bash
systemctl enable dnsmasq.service
```

## Ubuntu Desktop 24.04, Raspberry Pi OS Bookworm

### Configure upstream WiFi networks

{{ img(path="@/blog/assets/rpi-hotspot-nmtui.png", class="bordered", alt="Screenshot", caption="Configure your networks as you would on any other Ubuntu Desktop system") }}

### Ignore `ap0`

`NetworkManager` aggressively tries to take over configuration for every network device it finds in the system. Create a file named `/etc/NetworkManager/conf.d/99-unmanaged-interfaces.conf` with the following contents to exclude the `ap0` device from `NetworkManager`, as `hostapd` is managing this device for us.

```ini
# /etc/NetworkManager/conf.d/99-unmanaged-interfaces.conf
[device-ap0-unmanaged]
match-device=interface-name:ap0
managed=0
```

### Create a network bridge for downstream network via `nmcli`

```bash
# Add bridge device
nmcli connection add type bridge con-name bridge0 ifname br0

# Add wired ethernet to the bridge
nmcli connection add type ethernet slave-type bridge con-name bridge0-wired ifname eth0 master br0
nmcli connection up bridge0-wired

# Share internet connection to bridge
nmcli connection add con-name bridge0-share type ethernet ifname br0 ipv4.method shared ipv6.method ignore
nmcli connection modify bridge0-share connection.mdns yes
nmcli connection up bridge0-share
```

## Finale

At this point you can reboot your system and the wifi network you configured should come up automatically on next boot!

# Comments

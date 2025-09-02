+++
title = "AP+STA mode on Raspberry Pi 5"
[taxonomies]
tags = ["raspberry pi"]
+++

## Todo - Some Narative

## Install Dependencies

```bash
# sudo apt install dnsmasq hostapd iw
```

## Configure Upstream WiFi Networks

### Option 1 - Netplan (Ubuntu Server 24.04)

```yaml
# /etc/netplan/50-cloud-init.yaml
network:
  version: 2
  wifis:
    wlan0:
      optional: true
      dhcp4: true
      access-points:
        "<SSID 1>":
          auth:
            key-management: "psk"
            password: "enter SSID 1's PSK"
        "<SSID 2>":
          auth:
            key-management: "psk"
            password: "enter SSID 2's PSK"
```

```bash
# sudo netplan apply
```

### Option 2 - NetworkManager (Ubuntu Desktop 24.04)

{{ img(path="@/blog/assets/rpi-hotspot-nmtui.png", class="bordered", alt="Screenshot", caption="Configure your networks as you would on any other Ubuntu Desktop system") }}

## Configure Downstream WiFi Access Point

### Set up an access point virtual interface on the Pi's onboard WiFi radio

```ini
# /etc/systemd/system/wifi-ap-interface@.service
[Unit]
Description=Create WiFi AP virtual interface %i
After=network.target
Before=hostapd.service dnsmasq.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/iw dev wlan0 interface add %i type __ap
ExecStop=/usr/sbin/iw dev %i del
StandardOutput=journal

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/hostapd/hostapd.conf
interface=ap0
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

```bash
# sudo systemctl enable wifi-ap-interface@ap0.service
# sudo systemctl enable hostapd.service
```

### Set up a network bridge device to host the downstream DHCP and DNS server

```ini
# /etc/systemd/network/20-br0.netdev
[NetDev]
Name=br0
Kind=bridge
```

```ini
# /etc/systemd/network/20-br0.network
[Match]
Name=br0

[Network]
Address=192.168.4.1/24
IPForward=yes
IPMasquerade=ipv4
```

While systemd-networkd has support for enabling a DHCP server on an upstream interface, it doesn't provide DNS server support. So we'll use dnsmasq to provide both DHCP and DNS for the downstream network.

```ini
# /etc/dnsmasq.d/br0.conf
interface=br0
bind-dynamic
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
port=53
resolv-file=/etc/resolv.conf
cache-size=300
```

```bash
# sudo systemctl enable dnsmasq.service
```

### Add the access point and ethernet port to the network bridge

```ini
# /etc/systemd/network/25-ap0.network
[Match]
Name=ap0

[Network]
Bridge=br0
```

```ini
# /etc/systemd/network/26-eth0.network
[Match]
Name=eth0

[Network]
Bridge=br0
```

### Final Tasks for ONLY Ubuntu Desktop 24.04

We need to add a rule for systemd-networkd to ignore wlan0, as we intend to manage it with NetworkManager

```ini
# /etc/systemd/network/10-wlan0.network
[Match]
Name=wlan0

[Link]
Unmanaged=yes
```

And we need to add rules for NetworkManager to ignore br0, eth0, and ap0, as they'll be managed by systemd-networkd

```ini
# /etc/NetworkManager/conf.d/99-unmanaged-interfaces.conf
[device-br0-unmanaged]
match-device=interface-name:br0
managed=0

[device-ap0-unmanaged]
match-device=interface-name:ap0
managed=0

[device-eth0-unmanaged]
match-device=interface-name:eth0
managed=0
```

```bash
# # On Ubuntu Desktop, systemd-networkd is disabled by default
# sudo systemctl enable systemd-networkd
```
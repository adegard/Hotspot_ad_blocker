
# Autonomous DNS-Filtering Wi-Fi Hotspot (Ad Blocker)

This guide explains how to turn your ARM-based Ubuntu server into a **self-contained Wi-Fi hotspot** with **DNS filtering** to block ads, trackers, and unwanted services—without needing Pi-hole or router reconfiguration.

---

## 🧰 Requirements

- ARM-based Ubuntu server (e.g., Raspberry Pi, Orange Pi)
- Ethernet connection to your main router
- Wi-Fi adapter (USB or onboard)
- Ubuntu 20.04+ recommended

---

## 📦 Packages Used

- `hostapd` — creates the Wi-Fi hotspot
- `dnsmasq` — handles DHCP and DNS
- `iptables` — enables NAT and traffic routing
- `curl` — downloads DNS blocklists

---

## 🔐 Features

- Secure WPA2 guest Wi-Fi network
- DNS filtering using public blocklists
- No need to modify router settings
- Auto-refresh blocklist daily via cron

---

## 🚀 Setup Instructions

### 1. Install Required Packages

```bash
sudo apt update
sudo apt install hostapd dnsmasq iptables curl
sudo apt install ifupdown


```

Enable services:

```bash
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
```

---

### 2. Configure Static IP for Wi-Fi Interface

Step 3: Configure /etc/network/interfaces
Edit the file:

```bash
sudo nano /etc/network/interfaces
Add this configuration:

ini
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet dhcp

auto wlan0
iface wlan0 inet static
  address 192.168.50.1
  netmask 255.255.255.0
```

🔧 Step 4: Restart Networking
```bash
sudo ifdown wlan0 && sudo ifup wlan0
```
---

### 3. Configure `hostapd` (Wi-Fi Access Point)

Create config file:

```bash
sudo nano /etc/hostapd/hostapd.conf
```

```ini
interface=wlan0
driver=nl80211
ssid=GuestWiFi
hw_mode=g
channel=6
auth_algs=1
wmm_enabled=0
wpa=2
wpa_passphrase=YourSecurePassword123
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
```

Point to config:

```bash
sudo nano /etc/default/hostapd
```

Add:

```bash
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

---

### 4. Configure `dnsmasq` (DHCP + DNS)

Backup original config:

```bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
```

Create new config:

```bash
sudo nano /etc/dnsmasq.conf
```

```ini
interface=wlan0
dhcp-range=192.168.50.10,192.168.50.100,12h
dhcp-option=6,192.168.50.1
addn-hosts=/etc/dns-blocklist.txt
```

Download blocklist:

```bash
curl -s https://raw.githubusercontent.com/notracking/hosts-blocklists/master/dnscrypt-proxy/dns-blocklist.txt -o /etc/dns-blocklist.txt
```
Find other DNS list at [https://github.com/hagezi/dns-blocklists](https://github.com/hagezi/dns-blocklists)
---

### 5. Enable IP Forwarding

Edit sysctl:

```bash
sudo nano /etc/sysctl.conf
```

Uncomment or add:

```bash
net.ipv4.ip_forward=1
```

Apply:

```bash
sudo sysctl -p
```

---

### 6. Set Up NAT with `iptables`

```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo apt install iptables-persistent
```

---

### 7. Start Services

```bash
sudo systemctl restart hostapd
sudo systemctl restart dnsmasq
```

---

## 🔁 Auto-Update DNS Blocklist

Create update script:

```bash
sudo nano /usr/local/bin/update-dns-blocklist.sh
```

```bash
#!/bin/bash
curl -s https://raw.githubusercontent.com/notracking/hosts-blocklists/master/dnscrypt-proxy/dns-blocklist.txt -o /etc/dns-blocklist.txt
systemctl restart dnsmasq
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/update-dns-blocklist.sh
```

Add to crontab:

```bash
sudo crontab -e
```

Add:

```cron
0 3 * * * /usr/local/bin/update-dns-blocklist.sh
```

---

## 🧪 Testing

- Connect a device to `GuestWiFi`
- Check IP range: `192.168.50.x`
- Try visiting ad-heavy sites—ads should be blocked
- Use `dig` or `nslookup` to verify DNS resolution

---



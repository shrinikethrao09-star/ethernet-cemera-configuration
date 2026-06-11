# Ethernet Internet Sharing (NAT Gateway)

This guide configures a Linux system to share Internet access from `wwan0`, `usb0`, or `wlan0` to devices connected on `eth0`.

## 1. Configure eth0

Check available connections:

```bash
nmcli connection show
```

Set a static IP on eth0:

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.addresses 192.168.0.1/24
sudo nmcli connection modify "Wired connection 1" ipv4.method manual
sudo nmcli connection up "Wired connection 1"
```

Verify:

```bash
ip addr show eth0
```

---

## 2. Enable IP Forwarding

Edit sysctl configuration:

```bash
sudo nano /etc/sysctl.conf
```

Add or uncomment:

```text
net.ipv4.ip_forward=1
```

Apply changes:

```bash
sudo sysctl -p
```

Verify:

```bash
cat /proc/sys/net/ipv4/ip_forward
```

Expected output:

```text
1
```

---

## 3. Install iptables-persistent

```bash
sudo apt update
sudo apt install iptables-persistent
```

Verify:

```bash
dpkg -l | grep iptables-persistent
```

---

## 4. Remove Existing Rules (Optional)

```bash
sudo iptables -t nat -F
sudo iptables -F FORWARD
```

Verify:

```bash
sudo iptables -t nat -L -n -v
sudo iptables -L FORWARD -n -v
```

---

## 5. Configure NAT

### Share Internet from WWAN

```bash
sudo iptables -t nat -A POSTROUTING -o wwan0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wwan0 -j ACCEPT
sudo iptables -A FORWARD -i wwan0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

### Share Internet from USB Network

```bash
sudo iptables -t nat -A POSTROUTING -o usb0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o usb0 -j ACCEPT
sudo iptables -A FORWARD -i usb0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

### Share Internet from Wi-Fi

```bash
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wlan0 -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

---

## 6. Save Rules

```bash
sudo netfilter-persistent save
```

Reload saved rules:

```bash
sudo netfilter-persistent reload
```

---

## 7. Verify

Check NAT table:

```bash
sudo iptables -t nat -L -n -v
```

Check forwarding rules:

```bash
sudo iptables -L FORWARD -n -v
```

Check IP forwarding:

```bash
cat /proc/sys/net/ipv4/ip_forward
```

---

## Network Diagram

```text
          Internet
              |
    +---------+---------+
    |                   |
wwan0 / usb0 / wlan0
              |
      Linux Gateway
      eth0 192.168.0.1
              |
       Ethernet Clients
```

## Notes

* `eth0` is the LAN interface.
* `wwan0`, `usb0`, or `wlan0` is the Internet-facing interface.
* Enable only one NAT configuration for the active WAN interface.
* Save rules using `netfilter-persistent save` so they survive reboot.

  ## 🌐 Restore DHCP Mode (Default Network)

If you want to return `eth0` to normal DHCP mode:

```bash
sudo nmcli connection modify "Wired connection 1" ipv4.method auto
sudo nmcli connection modify "Wired connection 1" ipv4.addresses ""
sudo nmcli connection up "Wired connection 1"
```

Verify:

```bash
ip addr show eth0
nmcli connection show "Wired connection 1"
```

**Restores:**

* DHCP IP assignment
* Automatic gateway configuration
* Automatic DNS configuration

---

## 🧹 Disable NAT and IP Forwarding (Optional)

```bash
sudo iptables -t nat -F
sudo iptables -F FORWARD
sudo sysctl -w net.ipv4.ip_forward=0
sudo netfilter-persistent save
```

**Disables:**

* NAT (MASQUERADE)
* Packet forwarding
* Internet sharing

```
```


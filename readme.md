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

# 💾 Auto Mount USB Pen Drive on RK3568

This guide automatically mounts a USB pen drive to `/mnt/usb` whenever it is plugged into the RK3568 device.


Verify:

```bash
ls -ld /mnt/usb
```

---

## ⚙️ Step 2: Create systemd Service

Create a systemd template service:

```bash
sudo nano /etc/systemd/system/usb-mount@.service
```

Paste:

```ini
[Unit]
Description=Auto mount pendrive %I

[Service]
Type=oneshot
ExecStart=/bin/mount /dev/%I /mnt/usb
```

Save and exit.

---

## 🔄 Step 3: Reload systemd

```bash
sudo systemctl daemon-reload
```

Verify:

```bash
systemctl status usb-mount@.service
```

---

## 🔌 Step 4: Create udev Rule

Create a udev rule to detect USB partitions:

```bash
sudo nano /etc/udev/rules.d/99-usb-mount.rules
```

Paste:

```udev
ACTION=="add", KERNEL=="sd[a-z][0-9]", TAG+="systemd", ENV{SYSTEMD_WANTS}="usb-mount@%k.service"
```

Save and exit.

---

## 🔄 Step 5: Reload udev Rules

```bash
sudo udevadm control --reload
sudo udevadm trigger
```

Verify:

```bash
sudo udevadm test /sys/block/sda/sda1
```

---

## 🚀 Step 6: Test

Insert a USB pen drive.

Check if it is mounted:

```bash
mount | grep /mnt/usb
```

Or:

```bash
df -h
```

Expected output:

```text
/dev/sda1 on /mnt/usb
```

---

## 📂 Access Files

```bash
cd /mnt/usb
ls -la
```

---

## 🛠️ Troubleshooting

Check detected USB devices:

```bash
lsblk
```

Check systemd logs:

```bash
journalctl -u usb-mount@sda1.service
```

Check udev events:

```bash
udevadm monitor
```

---

## 📌 Notes

* Supports USB partitions such as `sda1`, `sdb1`, `sdc1`, etc.
* Mount point is `/mnt/usb`.
* Requires a supported filesystem (FAT32, exFAT, NTFS, ext4).
* If another USB drive is inserted while one is already mounted, the mount command may fail because all devices use the same mount point.

  # RK3568 VNC + SSH Proxy (Clean Setup)

## 1. Connect to RK3568 (from ThinkPad)

```bash
ssh -o 'ProxyCommand socat - PROXY:138.201.17.179:%h:%p,proxyport=5002' \
sh@KA01MS8977.example.com
```

---

## 2. Install VNC Viewer on ThinkPad

### Ubuntu/Debian

Download:

https://www.realvnc.com/en/connect/download/viewer/

Or install via terminal:

```bash
wget https://downloads.realvnc.com/download/file/viewer.files/VNC-Viewer-Latest-Linux-x64.deb
sudo apt install ./VNC-Viewer-Latest-Linux-x64.deb
```

Launch:

```bash
vncviewer
```

---

## 3. Start VNC Server on RK3568

### Create password (first time only)

```bash
x11vnc -storepasswd
```

Password file:

```text
/home/sh/.vnc/passwd
```

### Start x11vnc

```bash
sudo x11vnc \
-display :0 \
-auth /var/run/lightdm/root/:0 \
-rfbauth /home/sh/.vnc/passwd \
-forever \
-shared
```

Expected output:

```text
PORT=5901
The VNC desktop is: :1
```

Leave this terminal running.

---

## 4. Create SSH Tunnel (NEW terminal on ThinkPad)

```bash
ssh -N \
-L 5901:127.0.0.1:5901 \
-o 'ProxyCommand socat - PROXY:138.201.17.179:%h:%p,proxyport=5002' \
sh@KA01MS8977.example.com
```

Keep this terminal open.

---

## 5. Open VNC Viewer

Launch:

```bash
vncviewer
```

Connect to:

```text
localhost:5901
```

Then:

1. Click Connect
2. Enter VNC password
3. Accept warning if shown
4. RK3568 desktop should appear

---

## 6. Stop Everything

### Stop VNC Server

On RK3568 terminal:

```text
Ctrl + C
```

### Stop SSH Tunnel

On ThinkPad tunnel terminal:

```text
Ctrl + C
```

### Verify VNC stopped

```bash
ss -tlnp | grep 5901
```

No output means VNC server is stopped.




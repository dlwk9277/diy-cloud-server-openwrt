# OpenWrt WireGuard VPN + USB File Sharing Setup Guide

Complete guide to set up WireGuard VPN and Samba file sharing on OpenWrt 23.05.x with USB storage support.

## 📋 Table of Contents
- [Prerequisites](#prerequisites)
- [Part 1: Initial Router Configuration](#part-1-initial-router-configuration)
- [Part 2: USB Storage Setup](#part-2-usb-storage-setup)
- [Part 3: WireGuard VPN Setup](#part-3-wireguard-vpn-setup)
- [Part 4: Samba File Sharing](#part-4-samba-file-sharing)
- [Part 5: Port Forwarding (Optional)](#part-5-port-forwarding-optional)
- [Testing & Troubleshooting](#testing--troubleshooting)

---

## Prerequisites

- OpenWrt router running **23.05.x firmware** with USB port
- USB drive (formatted as FAT32, exFAT, or ext4)
- Router connected as **access point** to main router (LAN-to-LAN)
- SSH access to OpenWrt router
- Basic command line knowledge

**Compatible with:** Any OpenWrt-supported router regardless of hardware brand (TP-Link, Netgear, Linksys, etc.)

---

## Part 1: Initial Router Configuration

### Step 1: Connect to OpenWrt Router

```bash
ssh root@192.168.1.1
```
*Replace `192.168.1.1` with your OpenWrt router's IP*

### Step 2: Set Root Password (IMPORTANT!)

```bash
passwd
```
Enter a strong password when prompted.

### Step 3: Configure as Access Point

**Replace `X.X.X.X` with your main router's IP address** (e.g., `192.168.50.1`)

```bash
# Disable WAN interface
uci set network.wan.proto='none'
uci set network.wan6.proto='none'

# Set main router as gateway and DNS
uci set network.lan.gateway='X.X.X.X'
uci set network.lan.dns='X.X.X.X'

# Disable DHCP server (main router handles this)
uci set dhcp.lan.ignore='1'

# Change LAN IP to avoid conflict
# Replace Y.Y.Y.Y with an available IP on your network (e.g., 192.168.50.2)
uci set network.lan.ipaddr='Y.Y.Y.Y'

# Commit changes
uci commit
/etc/init.d/network restart
/etc/init.d/dnsmasq restart
```

**Your SSH will disconnect.** Wait 30 seconds, then reconnect using the new IP:

```bash
ssh root@Y.Y.Y.Y
```

### Step 4: Fix SSL Certificate Issues

```bash
echo "option check_signature 0" >> /etc/opkg.conf
```

### Step 5: Update Package Lists

```bash
opkg update
```

---

## Part 2: USB Storage Setup

### Step 1: Install USB Support Packages

```bash
opkg install kmod-usb-storage block-mount kmod-fs-ext4 kmod-fs-vfat kmod-fs-exfat usbutils
```

### Step 2: Verify USB Drive Detection

Plug in your USB drive, then check:

```bash
lsusb
ls -l /dev/sd*
```

You should see `/dev/sda1` (or similar).

### Step 3: Mount USB Drive

```bash
# Create mount point
mkdir -p /mnt/usb

# Mount the drive
mount /dev/sda1 /mnt/usb

# Verify
df -h /mnt/usb
ls -la /mnt/usb
```

### Step 4: Auto-Mount on Boot

```bash
block detect > /etc/config/fstab
uci set fstab.@mount[0].enabled='1'
uci set fstab.@mount[0].target='/mnt/usb'
uci commit fstab
/etc/init.d/fstab enable
```

---

## Part 3: WireGuard VPN Setup

### Step 1: Install WireGuard

```bash
opkg install wireguard-tools luci-app-wireguard qrencode
```

### Step 2: Generate Server Keys

```bash
mkdir -p /etc/wireguard

wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key

echo "Server Private Key:"
cat /etc/wireguard/server_private.key
echo ""
echo "Server Public Key:"
cat /etc/wireguard/server_public.key
```

**Save these keys somewhere safe!**

### Step 3: Configure WireGuard Interface

**Replace `SERVER_PRIVATE_KEY` with your actual server private key from Step 2:**

```bash
uci set network.wg0=interface
uci set network.wg0.proto='wireguard'
uci set network.wg0.private_key='SERVER_PRIVATE_KEY'
uci set network.wg0.listen_port='51820'
uci add_list network.wg0.addresses='10.0.0.1/24'
uci commit network
```

### Step 4: Configure Firewall

```bash
# Create WireGuard zone
uci add firewall zone
uci set firewall.@zone[-1].name='wg'
uci set firewall.@zone[-1].input='ACCEPT'
uci set firewall.@zone[-1].output='ACCEPT'
uci set firewall.@zone[-1].forward='ACCEPT'
uci set firewall.@zone[-1].masq='1'
uci add_list firewall.@zone[-1].network='wg0'

# Allow WireGuard to access LAN
uci add firewall forwarding
uci set firewall.@forwarding[-1].src='wg'
uci set firewall.@forwarding[-1].dest='lan'

# Allow WireGuard port through firewall
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-WireGuard'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].dest_port='51820'
uci set firewall.@rule[-1].proto='udp'
uci set firewall.@rule[-1].target='ACCEPT'

uci commit firewall
/etc/init.d/network restart
/etc/init.d/firewall restart
```

**Your SSH will disconnect.** Reconnect after 30 seconds.

### Step 5: Verify WireGuard Interface

```bash
ip addr show wg0
```

You should see `10.0.0.1/24`.

### Step 6: Add WireGuard Client

#### Generate Client Keys

```bash
wg genkey | tee /etc/wireguard/client1_private.key | wg pubkey > /etc/wireguard/client1_public.key

echo "Client Private Key:"
cat /etc/wireguard/client1_private.key
echo ""
echo "Client Public Key:"
cat /etc/wireguard/client1_public.key
```

#### Add Client to Server

**Replace `CLIENT_PUBLIC_KEY` with your actual client public key:**

```bash
uci add network wireguard_wg0
uci set network.@wireguard_wg0[-1].public_key='CLIENT_PUBLIC_KEY'
uci set network.@wireguard_wg0[-1].description='Client1'
uci add_list network.@wireguard_wg0[-1].allowed_ips='10.0.0.2/32'
uci commit network
ifup wg0
```

#### Get Your Public IP

```bash
wget -O - http://ipinfo.io/ip 2>/dev/null
echo ""
```

#### Create Client Configuration

**Replace the following in the commands below:**
- `CLIENT_PRIVATE_KEY` = Your client private key
- `YOUR_MAIN_ROUTER_IP` = Your main router's IP (e.g., 192.168.50.1)
- `SERVER_PUBLIC_KEY` = Your server public key from Step 2
- `YOUR_PUBLIC_IP` = Your public IP from the command above

```bash
cat > /tmp/client1.conf << EOF
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.0.0.2/24
DNS = YOUR_MAIN_ROUTER_IP

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = YOUR_PUBLIC_IP:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF

cat /tmp/client1.conf
```

#### Generate QR Code (for mobile devices)

```bash
qrencode -t ansiutf8 < /tmp/client1.conf
```

**Scan this QR code with the WireGuard app on your phone!**

Or save the config file and import it into WireGuard desktop client.

---

## Part 4: Samba File Sharing

### Step 1: Install Samba (if not already installed)

```bash
opkg install samba4-server samba4-utils luci-app-samba4
```

If installation fails, you can configure manually (see below).

### Step 2: Create Shared Directories

```bash
# Create directories on USB drive
mkdir -p /mnt/usb/shared
mkdir -p /mnt/usb/uploads

# Set permissions
chmod 755 /mnt/usb/shared
chmod 775 /mnt/usb/uploads
```

### Step 3: Create Samba Users

#### Option A: View-Only Access (No Password Required)

For read-only access without authentication, skip user creation and configure Samba for guest access (see Step 4A).

#### Option B: Upload Access (Password Required)

```bash
# Create a system user for upload access
# Replace 'uploaduser' with your preferred username
opkg install shadow-useradd

useradd -M -s /bin/false uploaduser

# Set Samba password
# Replace 'uploaduser' with your username
smbpasswd -a uploaduser
```

Enter a password when prompted.

### Step 4: Configure Samba

Edit the Samba configuration:

```bash
vi /etc/samba/smb.conf.template
```

**Or create it from scratch:**

```bash
cat > /etc/samba/smb.conf.template << 'EOF'
[global]
    netbios name = OpenWrt
    workgroup = WORKGROUP
    server string = OpenWrt Samba Server
    security = user
    guest account = nobody
    map to guest = Bad User
    
[Shared]
    path = /mnt/usb/shared
    browseable = yes
    read only = yes
    guest ok = yes
    
[Uploads]
    path = /mnt/usb/uploads
    browseable = yes
    read only = no
    guest ok = no
    valid users = uploaduser
    create mask = 0664
    directory mask = 0775
EOF
```

**Configuration Explanation:**
- **[Shared]**: Read-only share, accessible without password (`guest ok = yes`)
- **[Uploads]**: Read/write share, requires password (`valid users = uploaduser`)

**If you only want read-only access and don't need uploads, remove the `[Uploads]` section.**

### Step 5: Start Samba Service

```bash
/etc/init.d/samba4 enable
/etc/init.d/samba4 restart
```

### Step 6: Configure Firewall for Samba

```bash
# Allow Samba from WireGuard clients
uci add firewall rule
uci set firewall.@rule[-1].name='Allow-Samba-from-WG'
uci set firewall.@rule[-1].src='wg'
uci set firewall.@rule[-1].dest_port='445'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].target='ACCEPT'

uci add firewall rule
uci set firewall.@rule[-1].name='Allow-Samba-NetBIOS-from-WG'
uci set firewall.@rule[-1].src='wg'
uci set firewall.@rule[-1].dest_port='137-139'
uci set firewall.@rule[-1].proto='tcp'
uci set firewall.@rule[-1].target='ACCEPT'

uci commit firewall
/etc/init.d/firewall restart
```

---

## Part 5: Port Forwarding (Optional)

**Only needed if you want to connect to WireGuard from outside your home network** (e.g., from mobile data, coffee shop WiFi).

### On Your Main Router

1. Login to your main router (e.g., `192.168.50.1`)
2. Navigate to **WAN → Virtual Server / Port Forwarding**
3. Add a new rule:
   - **Service Name:** WireGuard
   - **Protocol:** UDP
   - **External Port:** 51820
   - **Internal IP:** `Y.Y.Y.Y` (your OpenWrt router's IP)
   - **Internal Port:** 51820
   - **Source IP:** Leave blank
4. **Save and Apply**

**Without port forwarding:** WireGuard only works when you're connected to your home WiFi.

**With port forwarding:** WireGuard works from anywhere with internet.

---

## Testing & Troubleshooting

### Test WireGuard Connection

1. Install WireGuard app on your device
2. Import the configuration (scan QR code or paste config)
3. Enable the VPN connection
4. Test by pinging the router: `ping 10.0.0.1`

### Test File Sharing

#### Access Read-Only Share (No Password)

**Windows:**
```
\\10.0.0.1\Shared
```

**macOS/Linux:**
```
smb://10.0.0.1/Shared
```

#### Access Upload Share (Password Required)

**Windows:**
```
\\10.0.0.1\Uploads
```
Username: `uploaduser`
Password: *your password*

**macOS/Linux:**
```
smb://uploaduser@10.0.0.1/Uploads
```

### Common Issues

#### USB Drive Not Detected
```bash
# Check if USB modules are loaded
lsmod | grep usb

# Check kernel messages
dmesg | grep -i usb

# Try unplugging and replugging the USB drive
```

#### WireGuard Not Working from Outside
- Verify port forwarding is configured correctly
- Check your public IP hasn't changed: `wget -O - http://ipinfo.io/ip 2>/dev/null`
- Update client config if public IP changed

#### Samba Not Accessible
```bash
# Check if Samba is running
/etc/init.d/samba4 status

# Restart Samba
/etc/init.d/samba4 restart

# Check firewall rules
uci show firewall | grep -i samba
```

#### Can't Connect via SSH After Configuration
- Wait 60 seconds for network to stabilize
- Try connecting to the new IP you configured
- If all else fails, reset router to factory defaults and start over

---

## Security Notes

✅ **Always set a strong root password**
✅ **Keep OpenWrt firmware updated**
✅ **Use strong passwords for Samba upload access**
✅ **WireGuard is extremely secure by design**
✅ **Port forwarding WireGuard is safe** - it's designed for this purpose

---

## Adding More WireGuard Clients

To add additional clients (e.g., laptop, tablet):

```bash
# Generate new client keys
wg genkey | tee /etc/wireguard/client2_private.key | wg pubkey > /etc/wireguard/client2_public.key

# Add to server (increment IP: 10.0.0.3, 10.0.0.4, etc.)
uci add network wireguard_wg0
uci set network.@wireguard_wg0[-1].public_key='NEW_CLIENT_PUBLIC_KEY'
uci set network.@wireguard_wg0[-1].description='Client2'
uci add_list network.@wireguard_wg0[-1].allowed_ips='10.0.0.3/32'
uci commit network
ifup wg0

# Create client config (change Address to 10.0.0.3/24)
```

---

## Credits

Guide created for OpenWrt 23.05.x

**Tested on:** ipq806x architecture (Netgear R7800)
**Should work on:** Any OpenWrt-compatible router with USB support

---

## License

This guide is provided as-is for educational purposes. Use at your own risk.

# 🚀 Build Your Own Private Cloud Server with Any Old Router

*Turn an old router into your personal cloud - Free alternative to Dropbox & paid VPNs*

## Why This Guide?

💰 **Save Money:** No monthly fees for cloud storage or VPN services  
🔒 **Privacy First:** Your data stays on your hardware, not on corporate servers  
🌍 **Access Anywhere:** Connect securely from any device, anywhere in the world  
♻️ **Repurpose Hardware:** Give that old router a new life  

## What You'll Get

- ✅ **VPN Server** - Secure remote access to your home network
- ✅ **Cloud Storage** - Access your files from anywhere
- ✅ **Two Access Levels** - Read-only for viewing, password-protected for uploads
- ✅ **Military-Grade Encryption** - WireGuard VPN protocol

## 📋 Table of Contents
- [Part 0: Installing OpenWrt Firmware](#part-0-installing-openwrt-firmware)
- [Prerequisites](#prerequisites)
- [Part 1: Initial Router Configuration](#part-1-initial-router-configuration)
- [Part 2: USB Storage Setup](#part-2-usb-storage-setup)
- [Part 3: WireGuard VPN Setup](#part-3-wireguard-vpn-setup)
- [Part 4: Samba File Sharing](#part-4-samba-file-sharing)
- [Part 5: Port Forwarding](#part-5-port-forwarding)
- [Testing Your Setup](#testing-your-setup)
- [Comprehensive Troubleshooting](#comprehensive-troubleshooting)
- [Advanced Configuration](#advanced-configuration)

---

## Part 0: Installing OpenWrt Firmware

### Before You Start

⚠️ **WARNING:** Flashing firmware will erase all current router settings. This process can brick your router if done incorrectly. Proceed at your own risk.

### Step 1: Check Router Compatibility

1. Visit the [OpenWrt Table of Hardware](https://openwrt.org/toh/start)
2. Search for your router model (look on the bottom/back of your router)
3. Check if it's supported and note the **device page link**

**What to look for:**
- ✅ Device status: "Supported"
- ✅ Flash size: At least 8MB (16MB+ recommended)
- ✅ RAM: At least 64MB (128MB+ recommended)
- ✅ USB port: Required for this guide

**Common supported routers:** TP-Link Archer C7, Netgear R7800, Linksys WRT series, ASUS RT-AC series

### Step 2: Download OpenWrt Firmware

1. On your router's device page, find the **Installation** section
2. Download the appropriate firmware file:
   - **For first-time installation:** Look for "Factory" or "Sysupgrade" file
   - **File extension:** Usually `.bin` or `.trx`
3. Save it to your computer

**Stuck here?**
- **Can't find your router:** Your router might not be supported. Consider buying a used supported router (~$20-50)
- **Multiple firmware files:** Read the device page carefully - it will specify which file to use
- **Page is confusing:** Look for the "Installation" tab on the device page

### Step 3: Backup Current Router Settings (Optional)

Before flashing:
1. Login to your current router's admin page
2. Export/backup your current configuration
3. Write down your WiFi passwords and settings

### Step 4: Flash OpenWrt Firmware

**Method varies by router manufacturer. Choose your method:**

#### Method A: Via Stock Firmware Web Interface (Easiest)

Most TP-Link, Netgear, and ASUS routers support this:

1. Login to your router's admin page (usually `192.168.1.1` or `192.168.0.1`)
2. Go to **Administration → Firmware Upgrade** (name varies)
3. Select the OpenWrt `.bin` file you downloaded
4. Click **Upload/Update**
5. **Wait 5-10 minutes** - Do NOT unplug during this process
6. Router will reboot automatically

#### Method B: Via TFTP (For bricked routers or specific models)

Some routers require TFTP method:

1. Download a TFTP client:
   - **Windows:** [Tftpd64](https://pjo2.github.io/tftpd64/)
   - **Mac/Linux:** Use built-in `tftp` command
2. Follow the specific TFTP instructions on your router's device page
3. Typical process:
   - Connect computer directly to router via ethernet
   - Set computer IP to `192.168.1.2` (static)
   - Put router in TFTP recovery mode (usually: hold reset while powering on)
   - Use TFTP to upload firmware

**Stuck here?**
- **Upload fails:** Make sure you downloaded the correct file (factory vs sysupgrade)
- **Router won't boot:** Try TFTP recovery method
- **Can't access router after flash:** Wait 10 minutes, then try `192.168.1.1` or perform factory reset

### Step 5: First Access to OpenWrt

1. Connect computer to router via ethernet cable
2. Open browser and go to `http://192.168.1.1`
3. You should see OpenWrt login page (no password set initially)
4. Click "Login" without entering password

**Success!** OpenWrt is now installed.

**Stuck here?**
- **Can't access 192.168.1.1:**
  - Check if computer got IP address: Run `ipconfig` (Windows) or `ifconfig` (Mac/Linux)
  - Manually set computer IP to `192.168.1.2`, subnet `255.255.255.0`, gateway `192.168.1.1`
- **Page won't load:** Clear browser cache or try different browser
- **Router seems dead:** Unplug power for 30 seconds, plug back in, wait 2 minutes

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

**Stuck here?**
- **Password too short error:** Use at least 8 characters with mix of letters, numbers, symbols
- **Can't type password:** Password input is hidden for security - just type and press Enter

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

**Stuck here?**
- **"Connection refused":** Wait another 30 seconds and try again
- **"Network unreachable":** Check that you set the correct gateway IP (your main router's IP)
- **WiFi stopped working:** The router is rebooting, wait 2-3 minutes
- **Can't reconnect at all:** 
  - Try connecting via ethernet cable
  - Check your computer's network adapter - make sure it got an IP from your main router
  - Factory reset OpenWrt router and start over: Hold reset button for 10 seconds while powered on

### Step 4: Fix SSL Certificate Issues

```bash
echo "option check_signature 0" >> /etc/opkg.conf
```

### Step 5: Update Package Lists

```bash
opkg update
```

**Stuck here?**
- **"Failed to download" errors:** 
  - Check internet: `ping -c 3 8.8.8.8`
  - If ping fails, double-check Step 3 configuration (gateway and DNS must be your main router's IP)
- **"wget returned 4" or "wget returned 5" errors:** SSL certificate issue - make sure you completed Step 4
- **Takes very long:** Normal on slow connections, wait up to 5 minutes

---

## Part 2: USB Storage Setup

### Step 1: Install USB Support Packages

```bash
opkg install kmod-usb-storage block-mount kmod-fs-ext4 kmod-fs-vfat kmod-fs-exfat usbutils
```

**Stuck here?**
- **"Unknown package" errors:** Some packages might already be installed or not needed for your hardware
- **"Insufficient space" error:** Your router's flash memory is full
  - Solution: Install fewer packages (skip `kmod-fs-exfat` if you don't use exFAT drives)
  - Or get a router with more flash memory
- **Installation hangs:** Press Ctrl+C, run `opkg update` again, then retry installation

### Step 2: Verify USB Drive Detection

Plug in your USB drive, then check:

```bash
lsusb
ls -l /dev/sd*
```

You should see `/dev/sda1` (or similar).

**Stuck here?**
- **"lsusb: not found":** The `usbutils` package didn't install. Skip this command and just check `/dev/sd*`
- **"No such file or directory" for /dev/sd*:**
  - Unplug USB drive, wait 5 seconds, plug back in
  - Check `dmesg | tail -20` for USB errors
  - Try different USB port
  - Your USB drive might be faulty - try another drive
- **See `/dev/sda` but no `/dev/sda1`:** Drive isn't partitioned
  - Plug drive into computer and format it (FAT32 or exFAT)
  - Or use `fdisk` on OpenWrt to partition it (advanced)

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

**Stuck here?**
- **"wrong fs type" error:** 
  - Check filesystem: `blkid /dev/sda1` (if blkid exists)
  - If it's NTFS: `opkg install kmod-fs-ntfs ntfs-3g`, then retry mount
  - If unknown filesystem: Reformat drive on computer as FAT32 or exFAT
- **"already mounted" error:** Drive is already mounted, check with `mount | grep sda`
- **Mount succeeds but `df -h` shows 0 bytes:** Drive might be corrupted, try reformatting
- **Permission denied:** Run commands as root (you should already be root via SSH)

### Step 4: Auto-Mount on Boot

```bash
block detect > /etc/config/fstab
uci set fstab.@mount[0].enabled='1'
uci set fstab.@mount[0].target='/mnt/usb'
uci commit fstab
/etc/init.d/fstab enable
```

**Stuck here?**
- **"@mount[0]" not found:** Run `cat /etc/config/fstab` to see what was detected
  - If empty: `block detect` didn't find your drive - check if it's still mounted
  - Manually add mount config (see [Advanced Configuration](#advanced-configuration))
- **Drive doesn't mount after reboot:** Check `/etc/config/fstab` syntax, or just manually mount after each reboot

---

## Part 3: WireGuard VPN Setup

### Step 1: Install WireGuard

```bash
opkg install wireguard-tools luci-app-wireguard qrencode
```

**Stuck here?**
- **"Unknown package wireguard-tools":** Wrong OpenWrt version or architecture issue
  - Check version: `cat /etc/openwrt_release`
  - Should be 23.05.x or newer
- **"qrencode" fails to install:** Not critical, you can skip it and manually type config on devices
- **Insufficient space:** Skip `luci-app-wireguard` (web UI) - you can configure via command line only

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

**Stuck here?**
- **"wg: not found":** WireGuard tools didn't install properly - rerun `opkg install wireguard-tools`
- **Directory creation fails:** You might not have write permissions - make sure you're root
- **Keys look weird:** They should be 44 characters of random letters/numbers/symbols - this is normal

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

**Stuck here?**
- **Can't reconnect:** Wait full 2 minutes - network restart takes time
- **"Connection refused":** Firewall might be blocking SSH
  - Connect via ethernet cable if on WiFi
  - Access router web UI at `http://Y.Y.Y.Y` and check firewall settings
- **Lost access completely:** Factory reset and start over, or connect via serial console (advanced)

### Step 5: Verify WireGuard Interface

```bash
ip addr show wg0
```

You should see `10.0.0.1/24`.

**Stuck here?**
- **"Device wg0 does not exist":**
  - Check if interface was created: `uci show network.wg0`
  - If empty: Redo Step 3
  - Try manually bringing up: `ifup wg0`
- **Interface shows but no IP:** Check configuration: `uci show network.wg0 | grep addresses`

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

**Stuck here?**
- **Command hangs or fails:**
  - Try: `curl ifconfig.me` (if curl is installed)
  - Or Google "what is my ip" on another device
  - Or check your main router's WAN IP in its admin page
- **IP starts with 192.168 or 10.:** That's your local IP, not public IP - check main router's WAN IP instead
- **IP keeps changing:** You have dynamic IP - consider using a DDNS service (see [Advanced Configuration](#advanced-configuration))

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

**Stuck here?**
- **"qrencode: not found":** It wasn't installed - manually type the config into your device instead
- **QR code doesn't scan:** Make sure terminal window is large enough to display full QR code
- **Config file has wrong values:** Double-check you replaced all placeholders (CLIENT_PRIVATE_KEY, etc.)

---

## Part 4: Samba File Sharing

### Step 1: Install Samba (if not already installed)

```bash
opkg install samba4-server samba4-utils luci-app-samba4
```

If installation fails, you can configure manually (see below).

**Stuck here?**
- **"Failed to download" errors:**
  - Try: `opkg update` then retry installation
  - Mirror might be down - wait an hour and try again
  - Can skip `luci-app-samba4` if needed (web UI only)
- **"Insufficient space":** Skip `samba4-utils` and `luci-app-samba4` - only `samba4-server` is essential
- **Installation hangs:** Press Ctrl+C and retry, or reboot router and try again

### Step 2: Create Shared Directories

```bash
# Create directories on USB drive
mkdir -p /mnt/usb/shared
mkdir -p /mnt/usb/uploads

# Set permissions
chmod 755 /mnt/usb/shared
chmod 775 /mnt/usb/uploads
```

**Stuck here?**
- **"Read-only file system":** USB drive is mounted read-only
  - Remount: `mount -o remount,rw /mnt/usb`
  - Check drive health on computer - might be failing
- **"No space left on device":** USB drive is full - delete some files or use larger drive
- **Directories already exist:** That's fine, just continue

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

**Stuck here?**
- **"shadow-useradd" not found:** Try `opkg install shadow-common shadow-useradd`
- **"useradd: not found":** Package didn't install
  - Alternative: `opkg install busybox` (might have adduser built-in)
  - Or manually edit `/etc/passwd` and `/etc/shadow` (advanced)
- **"smbpasswd: not found":** Samba not installed correctly - retry Part 4 Step 1
- **Password rejected:** Use at least 8 characters

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

**Stuck here?**
- **"vi" is confusing:** Use `nano` instead if installed, or use the `cat >` method shown above
- **Config file already exists:** Backup first: `cp /etc/samba/smb.conf.template /etc/samba/smb.conf.template.bak`
- **Syntax errors:** Make sure indentation uses spaces (not tabs) and bracket sections `[Name]` are on their own lines

### Step 5: Start Samba Service

```bash
/etc/init.d/samba4 enable
/etc/init.d/samba4 restart
```

**Stuck here?**
- **"samba4: not found":** Samba not installed - redo Part 4 Step 1
- **Service fails to start:** Check logs: `logread | grep samba`
  - Common issue: Config file syntax error - review Step 4
  - Permission issue: Make sure `/mnt/usb` is mounted and writable
- **"smbd dead but pid file exists":** Kill old process: `killall smbd nmbd` then restart

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

**Stuck here?**
- **Firewall restart fails:** Check config: `uci show firewall | grep -i samba`
- **Rules not applying:** Manually restart: `fw3 reload`

---

## Part 5: Port Forwarding

**Only needed if you want to connect to WireGuard from outside your home network** (e.g., from mobile data, coffee shop WiFi).

### On Your Main Router

1. Login to your main router (e.g., `192.168.50.1`)
2. Navigate to **WAN → Virtual Server / Port Forwarding** (location varies by brand)
   - **TP-Link:** Forwarding → Virtual Servers
   - **Netgear:** Advanced → Port Forwarding / Port Triggering
   - **ASUS:** WAN → Virtual Server / Port Forwarding
   - **Linksys:** Security → Apps and Gaming → Single Port Forwarding
3. Add a new rule:
   - **Service Name:** WireGuard
   - **Protocol:** UDP (NOT TCP!)
   - **External Port:** 51820
   - **Internal IP:** `Y.Y.Y.Y` (your OpenWrt router's IP)
   - **Internal Port:** 51820
   - **Source IP:** Leave blank or "All"
4. **Save and Apply**

**Without port forwarding:** WireGuard only works when you're connected to your home WiFi.

**With port forwarding:** WireGuard works from anywhere with internet.

**Stuck here?**
- **Can't find port forwarding settings:**
  - Search router manual for "port forwarding" or "virtual server"
  - Try looking under: Advanced Settings, NAT, Gaming, or Applications
- **Router has UPnP instead:** UPnP won't work for WireGuard - you need manual port forwarding
- **Multiple WAN options:** Forward on the WAN interface that connects to internet (usually WAN1 or Primary WAN)
- **Internal IP not in dropdown:** Make sure OpenWrt router is connected and has the IP you configured
- **Still can't connect from outside:**
  - Check if ISP blocks incoming connections (call ISP)
  - Some ISPs use CGNAT (Carrier-Grade NAT) - port forwarding won't work, need VPN service instead
  - Verify port is open: Use online port checker tool (search "check port 51820")

---

## Testing Your Setup

### Test WireGuard Connection

1. Install WireGuard app on your device:
   - **Android:** [Google Play Store](https://play.google.com/store/apps/details?id=com.wireguard.android)
   - **iOS:** [App Store](https://apps.apple.com/us/app/wireguard/id1441195209)
   - **Windows:** [wireguard.com/install](https://www.wireguard.com/install/)
   - **Mac:** [App Store](https://apps.apple.com/us/app/wireguard/id1451685025)
   - **Linux:** `sudo apt install wireguard` (or equivalent)

2. Import the configuration:
   - **Mobile:** Scan QR code or paste config
   - **Desktop:** Import `/tmp/client1.conf` file

3. Enable the VPN connection

4. Test connectivity:
   - Ping the router: `ping 10.0.0.1`
   - Access router web UI: `http://10.0.0.1`

**Troubleshooting WireGuard:**

**Issue:** VPN shows "Connected" but no internet
- **Solution:** Check `AllowedIPs` in client config - should be `0.0.0.0/0` for full tunnel
- Try: Disable and re-enable VPN connection
- Check: Main router DNS is reachable

**Issue:** VPN won't connect at all
- **From home WiFi:**
  - Check: `ip addr show wg0` on router shows interface is up
  - Check: Client public key was added to server (Step 3.6)
  - Check: Firewall allows WireGuard zone
- **From outside:**
  - Check: Port forwarding is configured correctly
  - Check: Public IP in client config matches current public IP
  - Test: Use online port checker for port 51820 UDP
  - Check: ISP doesn't block incoming connections

**Issue:** Connection drops frequently
- **Solution:** Increase `PersistentKeepalive` in client config to 25 seconds
- Check: Network stability (try different WiFi/cellular)

**Issue:** Can connect but can't access LAN devices
- **Solution:** Check firewall forwarding: `uci show firewall | grep forward`
- Verify: WireGuard zone forwards to LAN zone

### Test File Sharing

#### Access Read-Only Share (No Password)

**Windows:**
1. Open File Explorer
2. Type in address bar: `\\10.0.0.1\Shared`
3. Press Enter

**macOS:**
1. Open Finder
2. Press `Cmd+K`
3. Enter: `smb://10.0.0.1/Shared`
4. Click Connect → Guest

**Linux:**
```bash
smbclient //10.0.0.1/Shared -N
```

**Mobile (Android/iOS):**
1. Install file manager app that supports SMB (e.g., FE File Explorer, Solid Explorer)
2. Add network location: `smb://10.0.0.1/Shared`

#### Access Upload Share (Password Required)

**Windows:**
1. Open File Explorer
2. Type: `\\10.0.0.1\Uploads`
3. Enter credentials:
   - Username: `uploaduser`
   - Password: *your password*

**macOS:**
1. Finder → `Cmd+K`
2. Enter: `smb://uploaduser@10.0.0.1/Uploads`
3. Enter password when prompted

**Linux:**
```bash
smbclient //10.0.0.1/Uploads -U uploaduser
```

**Troubleshooting File Sharing:**

**Issue:** Can't see/access shares
- **Check Samba is running:** `ps | grep smbd`
  - If not: `/etc/init.d/samba4 restart`
- **Check firewall:** `uci show firewall | grep -i samba`
- **Test locally first:** Connect to router WiFi (not via VPN) and try accessing

**Issue:** "Access Denied" or "Permission Denied"
- **For Uploads share:**
  - Verify user exists: `cat /etc/passwd | grep uploaduser`
  - Verify Samba password set: `pdbedit -L`
  - Check permissions: `ls -ld /mnt/usb/uploads`
- **For Shared share:**
  - Check `guest ok = yes` in config
  - Verify directory exists: `ls -la /mnt/usb/shared`

**Issue:** Share is empty or shows wrong files
- **Check USB is mounted:** `df -h | grep usb`
- **Check paths in Samba config match actual directories**

**Issue:** Can't write to Uploads share
- **Check permissions:** `ls -ld /mnt/usb/uploads` (should be 775 or 777)
- **Remount read-write:** `mount -o remount,rw /mnt/usb`
- **Check `read only = no` in Samba config**

**Issue:** Windows can't connect, shows error 0x80004005
- **Enable SMB 1.0 in Windows:**
  - Control Panel → Programs → Turn Windows features on/off
  - Check "SMB 1.0/CIFS File Sharing Support"
  - Restart Windows
- **Or update Samba config to use SMB2:**
  - Add to `[global]` section: `min protocol = SMB2`

---

## Comprehensive Troubleshooting

### General Network Issues

**Can't SSH to router:**
```bash
# From Windows Command Prompt or Mac/Linux Terminal:
ping Y.Y.Y.Y

# If ping works but SSH doesn't:
telnet Y.Y.Y.Y 22

# If telnet fails, SSH is blocked or down:
# - Try accessing web UI: http://Y.Y.Y.Y
# - Factory reset router if all else fails
```

**Internet works on router but not on clients:**
```bash
# On router, check DNS:
nslookup google.com

# If DNS fails:
uci set dhcp.@dnsmasq[0].server='8.8.8.8'
uci set dhcp.@dnsmasq[0].server='1.1.1.1'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

**USB drive stops working after reboot:**
```bash
# Check what happened:
dmesg | grep -i usb
dmesg | grep -i sd

# Manually mount:
mount /dev/sda1 /mnt/usb

# If that works, fstab config is wrong:
block detect > /etc/config/fstab
# Re-run Part 2 Step 4
```

### WireGuard Advanced Troubleshooting

**Check WireGuard status:**
```bash
wg show
```
Should show interface wg0 with your server private key and connected peers.

**No peers showing:**
```bash
# Check peer was added:
uci show network | grep wireguard_wg0

# If empty, re-add peer (Part 3 Step 6)
```

**Check if WireGuard is receiving packets:**
```bash
wg show wg0 transfer
```
`transfer` counters should increase when client tries to connect.

**Debug firewall:**
```bash
# Check if packets are being dropped:
logread | grep -i drop

# Temporarily disable firewall to test:
/etc/init.d/firewall stop
# Try connecting to VPN
# Re-enable after test:
/etc/init.d/firewall start
```

### Samba Advanced Troubleshooting

**Check Samba logs:**
```bash
logread | grep smbd
logread | grep nmbd
```

**Test Samba config syntax:**
```bash
testparm /etc/samba/smb.conf.template
```

**List Samba users:**
```bash
pdbedit -L -v
```

**Manually test SMB connection:**
```bash
# From OpenWrt router:
smbclient -L localhost -N

# Should show list of shares
```

**Reset Samba completely:**
```bash
/etc/init.d/samba4 stop
rm -rf /var/lib/samba/*
rm -rf /var/cache/samba/*
/etc/init.d/samba4 start

# Re-add users:
smbpasswd -a uploaduser
```

### Performance Issues

**Slow file transfers:**
- **Check USB drive speed:** Some old USB drives are very slow
- **Use USB 3.0 port if available**
- **Try different filesystem:** ext4 is usually faster than FAT32

**High CPU usage:**
```bash
top
# Press 'q' to quit

# If Samba using too much CPU:
# Reduce max connections in smb.conf:
# max connections = 5
```

**Out of memory errors:**
```bash
free
# Check available memory

# If low (<10MB free):
# Your router might not have enough RAM
# Disable unused services or get router with more RAM
```

### Factory Reset (Last Resort)

If everything is broken and you want to start over:

1. **Via hardware button:**
   - Unplug router
   - Hold reset button
   - Plug in power while holding reset
   - Keep holding for 10-15 seconds
   - Release when lights flash

2. **Via command line:**
```bash
firstboot -y
reboot
```

**WARNING:** This erases all your configuration!

---

## Advanced Configuration

### Dynamic DNS (DDNS) Setup

If your public IP changes frequently:

```bash
opkg install luci-app-ddns ddns-scripts

# Configure via web UI:
# Services → Dynamic DNS
# Or manually edit /etc/config/ddns
```

Popular DDNS providers: No-IP, DuckDNS, Dynu

Update your WireGuard client config to use your DDNS hostname instead of IP.

### Manual Fstab Configuration

If auto-detection doesn't work:

```bash
cat > /etc/config/fstab << 'EOF'
config mount
    option target '/mnt/usb'
    option device '/dev/sda1'
    option fstype 'auto'
    option options 'rw,sync'
    option enabled '1'
EOF

/etc/init.d/fstab restart
```

### Add Multiple USB Drives

```bash
# Create separate mount points:
mkdir -p /mnt/usb1
mkdir -p /mnt/usb2

# Add to fstab:
uci add fstab mount
uci set fstab.@mount[-1].target='/mnt/usb2'
uci set fstab.@mount[-1].device='/dev/sdb1'
uci set fstab.@mount[-1].enabled='1'
uci commit fstab
```

### Increase Samba Security

```bash
# In /etc/samba/smb.conf.template [global] section:
    restrict anonymous = 2
    min protocol = SMB2
    client min protocol = SMB2
    server signing = mandatory
```

### Backup Your Configuration

```bash
# Backup entire config:
sysupgrade -b /tmp/backup-$(date +%F).tar.gz

# Download to your computer via SCP:
# scp root@Y.Y.Y.Y:/tmp/backup-*.tar.gz ~/
```

### Monitor WireGuard Connections

```bash
# See connected clients:
wg show wg0

# Watch in real-time:
watch -n 1 wg show wg0

# Log connections:
logread -f | grep wireguard
```

### Set Up Wake-on-LAN (WOL)

Wake up computers on your LAN remotely via VPN:

```bash
opkg install etherwake

# Wake a device:
etherwake -i br-lan AA:BB:CC:DD:EE:FF
# Replace AA:BB:CC:DD:EE:FF with target device's MAC address
```

---

## Useful Commands Reference

```bash
# Network
ip addr show                    # Show all network interfaces
ip route show                   # Show routing table
ping -c 4 8.8.8.8              # Test internet connectivity
nslookup google.com            # Test DNS resolution

# USB
lsusb                          # List USB devices
ls -la /dev/sd*                # List storage devices
mount                          # Show mounted filesystems
df -h                          # Show disk space
dmesg | grep -i usb           # USB kernel messages

# WireGuard
wg show                        # Show WireGuard status
wg show wg0 allowed-ips       # Show allowed IPs
wg show wg0 transfer          # Show data transfer stats
ip addr show wg0              # Show WireGuard interface

# Samba
/etc/init.d/samba4 status     # Check if Samba is running
smbclient -L localhost -N     # List shares
pdbedit -L                     # List Samba users
testparm                       # Test Samba config

# System
top                            # Show CPU/memory usage
free                           # Show memory usage
logread                        # Show system logs
reboot                         # Reboot router
poweroff                       # Shutdown router

# Packages
opkg update                    # Update package list
opkg list-installed           # Show installed packages
opkg remove package-name      # Remove a package

# Configuration
uci show                       # Show all config
uci show network              # Show network config
uci changes                    # Show uncommitted changes
uci commit                     # Save changes
```

---

## Security Best Practices

✅ **Always use strong passwords** (15+ characters, mix of types)
✅ **Keep OpenWrt firmware updated:** Check [openwrt.org](https://openwrt.org) regularly
✅ **Disable WPS** on your main router
✅ **Change default SSH port** if exposed to internet (advanced)
✅ **Use key-based SSH authentication** instead of passwords (advanced)
✅ **Monitor logs regularly:** `logread` for suspicious activity
✅ **Backup your configuration** before making changes
✅ **Don't expose Samba to the internet** - only access via VPN
✅ **Use different passwords** for root, Samba, and WiFi

---

## FAQ

**Q: Can I use this with my ISP-provided router?**
A: Yes! This OpenWrt router acts as an access point behind your ISP router.

**Q: Will this slow down my internet?**
A: No. When not using VPN, devices connect directly to your main router. VPN speed depends on your upload speed and router CPU.

**Q: Can I use a bigger USB drive?**
A: Yes! Any size USB drive works. Larger drives = more storage.

**Q: What if my public IP changes?**
A: Use Dynamic DNS (DDNS) - see [Advanced Configuration](#advanced-configuration).

**Q: Can I access this on my TV/game console?**
A: If your device supports SMB file sharing or WireGuard, yes. Most smart TVs support SMB.

**Q: How many devices can connect?**
A: Depends on router hardware. Most can handle 5-10 WireGuard clients easily.

**Q: Is this legal?**
A: Yes! Running a VPN server on your own network is completely legal.

**Q: Can my ISP see what I'm doing?**
A: They can see you're using a VPN, but not what you're accessing through it.

**Q: What if I brick my router?**
A: Most routers have recovery mode. Check your router's OpenWrt device page for unbrick instructions.

**Q: Can I undo this?**
A: Yes, flash back to stock firmware (available from manufacturer's website).

---

## Changelog

**Version 1.0** - Initial release
- OpenWrt installation guide
- WireGuard VPN setup
- Samba file sharing (read-only & upload)
- Comprehensive troubleshooting

---

## Contributing

Found an issue or have improvements? Please:
1. Open an issue describing the problem
2. Submit a pull request with fixes
3. Share your experience and help others!

---

## Credits

Guide tested on OpenWrt 23.05.x

**Hardware tested:** Netgear R7800 (ipq806x)
**Should work on:** Any OpenWrt-supported router with USB

Special thanks to the OpenWrt and WireGuard communities.

---

## License

This guide is provided as-is for educational purposes. Use at your own risk.

MIT License - Feel free to fork, modify, and share!

---

## Support

**Need help?**
- 📖 [OpenWrt Documentation](https://openwrt.org/docs/start)
- 💬 [OpenWrt Forums](https://forum.openwrt.org/)
- 🔧 [WireGuard Documentation](https://www.wireguard.com/)
- 📝 Open an issue in this repository

**Before asking for help:**
1. Read the troubleshooting section
2. Check your router's OpenWrt device page
3. Include output of relevant commands when asking
4. Describe what you've already tried

---

**Happy self-hosting! 🎉**

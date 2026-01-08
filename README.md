# Raspberry Pi Model B+ v1.2 - USB Block Erupter Solo Mining Guide

A comprehensive guide to solo mining SHA256 (Bitcoin) using a USB Block Erupter on a Raspberry Pi Model B+ v1.2 connected via wired Ethernet.

## ⚠️ Important Disclaimers

- **Solo mining Bitcoin is extremely unlikely to be profitable** due to the network's high difficulty. You would need significant hash rate to have any realistic chance of finding a block.
- USB Block Erupters have very low hash rates (typically 330-400 MH/s) compared to modern ASIC miners (10+ TH/s+).
- This guide is for educational purposes, experimentation, and low-hash-rate solo mining.
- **No full Bitcoin node required** - this guide uses Stratum solo mining pools to avoid downloading the blockchain.
- This setup is optimized for a 16GB microSD card.

## Hardware Requirements

### Required Hardware
- **Raspberry Pi Model B+ v1.2** (512MB RAM, single-core ARM processor)
- **USB Block Erupter** (SHA256 USB Bitcoin miner - typically 330-400 MH/s)
- **MicroSD Card** (16GB minimum - sufficient for OS and mining software)
- **Ethernet Cable** (for wired connection)
- **USB Power Supply** (5V, 2A minimum recommended)
- **USB Hub with External Power** (highly recommended for stable Block Erupter operation)
- **MicroSD Card Reader** (for flashing OS)

### Optional but Recommended
- **Heat Sinks** (to prevent thermal throttling)
- **Cooling Fan** (for extended mining operations)
- **Case with Cooling** (to protect and cool the Raspberry Pi)

## Software Requirements

- **Raspberry Pi OS** (32-bit, Lite version recommended - minimal footprint)
- **Mining Software**: BFGMiner or CGMiner (supports Block Erupter via Stratum)
- **SSH Access** (for remote administration)
- **Bitcoin Address** (your own address where block rewards will be sent if you find one)

## Step 1: Prepare Raspberry Pi OS

### 1.1 Download and Flash Raspberry Pi OS

1. Download **Raspberry Pi OS Lite (32-bit)** from [raspberrypi.org](https://www.raspberrypi.org/software/operating-systems/)
2. Use **Raspberry Pi Imager** or **Balena Etcher** to flash the image to your microSD card
3. Before ejecting, enable SSH by creating an empty file named `ssh` in the boot partition
4. For headless setup, create a `wpa_supplicant.conf` file (if needed) or use wired Ethernet as planned

### 1.2 First Boot and Setup

1. Insert microSD card and connect:
   - Ethernet cable
   - USB power supply
   - USB miner (via powered USB hub recommended)

2. Boot the Pi and find its IP address:
   ```bash
   # On your local network router admin page, or use:
   # nmap -sn 192.168.1.0/24 | grep -B 2 "Raspberry Pi"
   ```

3. SSH into the Pi:
   ```bash
   ssh pi@<PI_IP_ADDRESS>
   # Default password: raspberry
   ```

### 1.3 Initial System Configuration

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Change default password (IMPORTANT!)
passwd

# Expand filesystem
sudo raspi-config
# Select: Advanced Options → Expand Filesystem → Finish → Reboot

# After reboot, reconnect via SSH
```

### 1.4 Install Essential Packages

```bash
# Install build tools and dependencies
sudo apt install -y \
  build-essential \
  git \
  autoconf \
  automake \
  libtool \
  pkg-config \
  libcurl4-openssl-dev \
  libudev-dev \
  libusb-1.0-0-dev \
  libjansson-dev \
  libncurses5-dev \
  zlib1g-dev \
  python3-dev \
  python3-pip \
  uthash-dev \
  libevent-dev \
  libmicrohttpd-dev \
  htop \
  screen
```

## Step 2: Connect and Verify USB Miner

### 2.1 Check USB Miner Recognition

```bash
# List USB devices
lsusb

# You should see your miner (e.g., Antminer, Block Erupter, etc.)
# Note the device ID for troubleshooting

# Check USB device details
dmesg | tail -20
```

### 2.2 Create udev Rules for Block Erupter

Block Erupters need udev rules for proper access:

```bash
# Create udev rule file
sudo nano /etc/udev/rules.d/50-block-erupter.rules
```

Add this content (Block Erupter vendor/product IDs):
```
SUBSYSTEM=="usb", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", MODE="0666", GROUP="plugdev"
SUBSYSTEM=="usb", ATTRS{idVendor}=="1d50", ATTRS{idProduct}=="6026", MODE="0666", GROUP="plugdev"
```

Then reload udev rules:
```bash
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Reconnect the Block Erupter after setting up udev rules.

### 2.3 Add User to Plugdev Group

```bash
sudo usermod -a -G plugdev pi
# Log out and back in for changes to take effect
```

## Step 3: Install Mining Software

### 3.1 Install BFGMiner (Recommended for Block Erupter)

BFGMiner has excellent support for Block Erupters and Stratum solo mining:

```bash
# Clone BFGMiner repository
cd ~
git clone https://github.com/luke-jr/bfgminer.git
cd bfgminer

# Configure git to use HTTPS instead of git:// protocol (fixes network issues)
# This prevents connection timeouts when cloning submodules
git config --global url."https://github.com/".insteadOf git://github.com/
git config --global url."https://git.ozlabs.org/".insteadOf git://git.ozlabs.org/

# Configure, compile, and install
./autogen.sh
./configure
make -j1
sudo make install

# Update library cache so system can find the libraries
sudo ldconfig
```
<｜tool▁calls▁begin｜><｜tool▁call▁begin｜>
read_file

**Note**: 
- Compilation may take 30-60 minutes on Raspberry Pi Model B+ due to its single-core processor
- Using `-j1` ensures stable compilation on limited RAM
- BFGMiner has built-in Block Erupter support

### 3.2 Alternative: CGMiner

If BFGMiner doesn't work with your Block Erupter:

```bash
cd ~
git clone https://github.com/ckolivas/cgminer.git
cd cgminer
./autogen.sh
./configure
make -j1
sudo make install
```

### 3.3 Verify Block Erupter Detection

After installation, test if your Block Erupter is detected:

```bash
# List detected devices
bfgminer --ndevices

# Or verbose device detection
bfgminer -D

# Should show "Block Erupter" or similar device
```

## Step 4: Set Up Solo Mining via Stratum Pool

This guide uses **Stratum solo mining** - you mine directly to your own Bitcoin address via a pool's infrastructure, avoiding the need to download the full blockchain. If you find a block, **you keep 100% of the reward**.

### 4.1 Choose a Solo Mining Pool

Popular options for Stratum solo mining:
- **Eligius** (eligius.st) - Long-standing pool with solo mining support
- **ckpool** - Stratum solo mining pool
- Other pools that support Stratum solo mode

### 4.2 Get Your Bitcoin Address

You need your own Bitcoin address where block rewards will be sent. Options:

1. **Generate an address** using a wallet (Electrum, Bitcoin Core, etc.)
2. **Use an existing address** from your wallet
3. **Create a new address** for mining purposes

**Important**: If you find a block, the reward goes to this address. Keep the private key secure!

### 4.3 Pool Configuration Information

For this guide, we'll use **Eligius** as an example. Their Stratum solo mining endpoint:
- **Pool URL**: `stratum+tcp://us-east.eligius.st:4444`
- **Username**: Your Bitcoin address
- **Password**: Can be anything (e.g., `x` or your email)

**Note**: Always check the pool's current documentation for up-to-date endpoints and requirements.

## Step 5: Configure and Start Mining

### 5.1 Create Mining Configuration Script

```bash
nano ~/start-mining.sh
```

Add (replace with your actual Bitcoin address and pool details):

```bash
#!/bin/bash

# Solo Mining Configuration for Block Erupter
# Replace these with your actual values:

# Your Bitcoin address (where block rewards will be sent)
BITCOIN_ADDRESS="YOUR_BITCOIN_ADDRESS_HERE"

# Pool configuration (example: Eligius)
POOL_URL="stratum+tcp://us-east.eligius.st:4444"
POOL_USER="$BITCOIN_ADDRESS"
POOL_PASS="x"

# Start BFGMiner for solo mining via Stratum
bfgminer \
  -o "$POOL_URL" \
  -u "$POOL_USER" \
  -p "$POOL_PASS" \
  --api-listen \
  --api-port 4028 \
  -S blockerupter:all \
  --set-device blockerupter:clock=auto \
  --real-quiet

# Alternative if you need to specify device manually:
# bfgminer \
#   -o "$POOL_URL" \
#   -u "$POOL_USER" \
#   -p "$POOL_PASS" \
#   --api-listen \
#   --api-port 4028 \
#   -S all:auto \
#   --set-device all:clock=auto

# Alternative using CGMiner:
# cgminer \
#   -o "$POOL_URL" \
#   -u "$POOL_USER" \
#   -p "$POOL_PASS" \
#   --api-listen \
#   --api-port 4028
```

Make it executable:

```bash
chmod +x ~/start-mining.sh
```

**Important**: Update `BITCOIN_ADDRESS` in the script with your actual Bitcoin address!

### 5.2 Test Mining Software Detection

Before starting full mining, verify your Block Erupter is detected:

```bash
# Test device detection
bfgminer --ndevices

# Verbose device detection
bfgminer -D

# Should show "Block Erupter" device
# Example output should include your Block Erupter model
```

### 5.3 Test Connection to Pool

Test the connection without full mining:

```bash
# Quick test run (press Q to quit after verifying connection)
bfgminer \
  -o stratum+tcp://us-east.eligius.st:4444 \
  -u YOUR_BITCOIN_ADDRESS \
  -p x \
  -S blockerupter:all \
  --debug
```

If you see connection successful messages, proceed to start mining.

### 5.4 Start Mining

```bash
# Start mining in a screen session (continues after disconnect)
screen -S mining
~/start-mining.sh

# Detach from screen: Press Ctrl+A, then D
# Reattach: screen -r mining
# List screens: screen -ls
```

### 5.5 Monitor Mining

In another SSH session or screen window:

```bash
# View miner statistics via API
curl http://127.0.0.1:4028/api?command=devs
curl http://127.0.0.1:4028/api?command=summary

# Check pool connection status
curl http://127.0.0.1:4028/api?command=pools

# Monitor system resources
htop

# Check Block Erupter USB status
dmesg | grep -i usb | tail -10

# Check CPU temperature
vcgencmd measure_temp
```

### 5.6 View Mining Output

When attached to the mining screen, you should see:
- Hash rate (typically 330-400 MH/s for Block Erupter)
- Shares submitted
- Pool connection status
- Accepted/rejected shares

## Step 6: Optimize for Raspberry Pi Model B+ v1.2

The Model B+ v1.2 has limited resources. Optimize for stability:

### 6.1 Overclock (Optional - Use with Caution)

```bash
sudo nano /boot/config.txt
```

Add (conservative overclock):

```
# Overclock settings for Model B+
arm_freq=950
core_freq=450
sdram_freq=450
over_voltage=2
```

**Warning**: Overclocking can cause instability or damage. Monitor temperature.

### 6.2 Increase USB Current Limit

The Model B+ can provide more current to USB devices:

```bash
sudo nano /boot/config.txt
```

Add:

```
# Increase USB current limit
max_usb_current=1
```

Reboot after changes.

### 6.3 Monitor Temperature

```bash
# Check CPU temperature
vcgencmd measure_temp

# Create temperature monitoring script
nano ~/check-temp.sh
```

Add:

```bash
#!/bin/bash
while true; do
  temp=$(vcgencmd measure_temp | cut -d= -f2)
  echo "$(date): $temp"
  sleep 60
done
```

### 6.4 Monitor Disk Space (16GB Card)

With only 16GB, monitor disk usage regularly:

```bash
# Check disk usage
df -h

# Check largest directories
du -sh ~/* | sort -h

# Clean package cache if needed
sudo apt clean

# Remove old kernels if needed
sudo apt autoremove
```

## Step 7: Troubleshooting

### Block Erupter Not Detected

```bash
# Check USB connection
lsusb -v | grep -i "10c4\|1d50"

# Check USB power
dmesg | grep -i usb | tail -20

# Check if Block Erupter is recognized
lsusb

# Try different USB port or powered hub
# Block Erupters need stable power - use powered USB hub
```

### Low Hash Rate or Instability

- **Use powered USB hub** - Block Erupters require stable power
- Check USB connection stability
- Try different USB cable
- Verify Block Erupter isn't overheating (they can get hot)
- Check USB bandwidth issues
- Try different USB ports (avoid USB 3.0 if available, USB 2.0 is fine)
- Reduce clock speed if unstable: `--set-device blockerupter:clock=200`

### Cannot Connect to Pool

```bash
# Test internet connectivity
ping -c 4 8.8.8.8

# Test DNS resolution
nslookup us-east.eligius.st

# Test pool port
telnet us-east.eligius.st 4444

# Check firewall rules
sudo iptables -L

# Verify pool URL and credentials in start-mining.sh
```

### Missing Dependencies (uthash-dev, libevent-dev, etc.)

If `./configure` fails with errors about missing libraries:

**For uthash-dev error:**
```bash
sudo apt install -y uthash-dev
```

**For libevent error (required):**
```bash
# libevent >= 2.0.3 is required
sudo apt install -y libevent-dev
```

**For libmicrohttpd warning (optional, but recommended):**
```bash
# libmicrohttpd is optional but enables getwork proxy
sudo apt install -y libmicrohttpd-dev
```

**Install all at once:**
```bash
sudo apt install -y uthash-dev libevent-dev libmicrohttpd-dev
```

Then run configure again:
```bash
./configure
```

### Git Submodule Clone Failures (Connection Timeout)

If `./autogen.sh` fails with "fatal: unable to connect" errors when cloning submodules:

```bash
# Configure git to use HTTPS instead of git:// protocol
git config --global url."https://github.com/".insteadOf git://github.com/
git config --global url."https://git.ozlabs.org/".insteadOf git://git.ozlabs.org/

# Clean up and retry
cd ~/bfgminer
rm -rf .git/modules
git submodule deinit --all -f
git submodule update --init --recursive

# Then run autogen.sh again
./autogen.sh
```

**Alternative**: If you're already in the bfgminer directory and autogen.sh failed:

```bash
# Configure git URL rewriting
git config --global url."https://github.com/".insteadOf git://github.com/
git config --global url."https://git.ozlabs.org/".insteadOf git://git.ozlabs.org/

# Manually clone missing submodules with HTTPS
cd ~/bfgminer
git submodule deinit --all -f
git submodule sync
git submodule update --init --recursive

# Then continue with configure
./configure
```

### Shared Library Not Found (libbase58.so.0, etc.)

If you get errors like "error while loading shared libraries: libbase58.so.0: cannot open shared object file":

```bash
# Update the library cache (this should fix it)
sudo ldconfig

# Verify the library is found
ldconfig -p | grep libbase58

# If still not found, check if library exists in standard locations
find /usr/local/lib -name "libbase58*" 2>/dev/null
find /usr/lib -name "libbase58*" 2>/dev/null

# If library exists but not found, add to library path temporarily:
export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH

# Or permanently add to /etc/ld.so.conf.d/
echo "/usr/local/lib" | sudo tee /etc/ld.so.conf.d/usr-local-lib.conf
sudo ldconfig
```

**Note**: After running `sudo make install`, always run `sudo ldconfig` to update the library cache.

### Mining Software Crashes

- Check available memory: `free -h`
- Reduce overclock if applied
- Check system logs: `journalctl -xe`
- Monitor temperature
- Use `screen` or `tmux` to keep session alive

## Step 8: Expected Performance and Realities

### Realistic Expectations

- **Block Erupter Hash Rate**: Typically 330-400 MH/s (0.33-0.4 GH/s)
- **Bitcoin Network Hash Rate**: ~400+ EH/s (400,000,000 TH/s)
- **Your Share**: Approximately 0.0000000001% of network hash rate
- **Time to Find Block**: Statistically millions of years (if ever)
- **Probability**: Near zero for profitable solo mining
- **Average Block Time**: ~10 minutes (if you found blocks, which is extremely unlikely)

### Why This Guide Still Exists

- Educational purposes
- Understanding Bitcoin mining
- Testing hardware
- Experimentation and learning

### Alternative Options

If you want actual rewards or different mining experiences:

1. **Join a Regular Mining Pool**: Pool mining allows you to receive regular payouts based on your contribution (not solo mining, but you'll get small payouts)
2. **Altcoin Solo Mining**: Mine SHA256 coins with lower difficulty (some altcoins support Stratum solo mining)
3. **Testnet Mining**: Some pools support testnet solo mining for testing purposes
4. **Different Algorithms**: Some USB miners support other algorithms with better profit potential

## Additional Resources

- [BFGMiner Documentation](https://github.com/luke-jr/bfgminer)
- [Eligius Pool Documentation](http://eligius.st/)
- [Raspberry Pi Documentation](https://www.raspberrypi.org/documentation/)
- [Block Erupter Information](https://en.bitcoin.it/wiki/Block_Erupter)
- [Stratum Mining Protocol](https://en.bitcoin.it/wiki/Stratum_mining_protocol)

## Maintenance

### Regular Tasks

1. **Monitor Temperature**: Ensure Pi and Block Erupter don't overheat
2. **Check Disk Space**: Monitor 16GB card usage (should be plenty for this setup)
3. **Update Software**: Keep mining software updated
4. **Monitor Mining**: Check hash rate and pool connection status regularly
5. **Check Pool Status**: Verify pool is operational and your address is configured correctly

### Auto-start on Boot (Optional)

```bash
sudo nano /etc/systemd/system/usb-miner.service
```

Add:

```ini
[Unit]
Description=USB Block Erupter Solo Miner
After=network.target

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi
ExecStart=/home/pi/start-mining.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable:

```bash
sudo systemctl enable usb-miner
```

## License

This guide is provided as-is for educational purposes. Use at your own risk.

## Block Erupter Specific Notes

### Expected Performance
- **Hash Rate**: 330-400 MH/s per Block Erupter
- **Power Consumption**: ~2.5W per device
- **Heat Output**: Moderate - ensure adequate ventilation
- **USB Requirements**: USB 2.0 compatible, benefits from powered USB hub

### Multiple Block Erupters
If you have multiple Block Erupters, BFGMiner can handle them:
```bash
# The -S blockerupter:all option automatically detects all Block Erupters
# Or specify individually:
-S blockerupter:/dev/ttyUSB0,blockerupter:/dev/ttyUSB1
```

### Clock Speed Options
Adjust clock speed for stability or performance:
```bash
--set-device blockerupter:clock=200  # Lower (more stable)
--set-device blockerupter:clock=250  # Default
--set-device blockerupter:clock=auto # Auto-detect
```

---

**Good luck with your Block Erupter solo mining experiment! Remember: this is primarily for educational purposes and understanding Bitcoin mining. The odds of finding a block are extremely low, but it's a great learning experience!**

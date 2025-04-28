# pi-usb-otg

Modify `/boot/firmware/cmdline.txt`:

```bash
sudo nano /boot/firmware/cmdline.txt
```

Add the following right after `rootwait`:

```
modules-load=dwc2,g_ether
```

Modify `/boot/firmware/config.txt`:

```bash
sudo nano /boot/firmware/config.txt
```

Add at the end of the file:

```
dtoverlay=dwc2
```

Create the USB gadget script:

```bash
sudo nano /usr/local/bin/usb-gadget.sh
```

Add the following content:

```bash
#!/bin/bash
set -e

modprobe libcomposite
cd /sys/kernel/config/usb_gadget/
mkdir -p g1
cd g1

echo 0x1d6b > idVendor
echo 0x0104 > idProduct
echo 0x0100 > bcdDevice
echo 0x0200 > bcdUSB

mkdir -p strings/0x409
echo "fedcba9876543210" > strings/0x409/serialnumber
echo "Raspberry Pi" > strings/0x409/manufacturer
echo "Pi Zero USB" > strings/0x409/product

mkdir -p configs/c.1/strings/0x409
echo "Config 1: ECM network" > configs/c.1/strings/0x409/configuration
echo 250 > configs/c.1/MaxPower

mkdir -p functions/ecm.usb0

# Check for existing symlink and remove if necessary
if [ -L configs/c.1/ecm.usb0 ]; then
    rm configs/c.1/ecm.usb0
fi
ln -s functions/ecm.usb0 configs/c.1/

# Ensure the device is not busy before listing available USB device controllers
max_retries=10
retry_count=0

while ! ls /sys/class/udc > UDC 2>/dev/null; do
    if [ $retry_count -ge $max_retries ]; then
        echo "Error: Device or resource busy after $max_retries attempts."
        exit 1
    fi
    retry_count=$((retry_count + 1))
    sleep 1
done

# Check if the usb0 interface is already configured
if ! ip addr show usb0 | grep -q "172.20.2.1"; then
    ifconfig usb0 172.20.2.1 netmask 255.255.255.0
else
    echo "Interface usb0 already configured."
fi
```

Make the script executable:

```bash
sudo chmod +x /usr/local/bin/usb-gadget.sh
```

Create the systemd service:

```bash
sudo nano /etc/systemd/system/usb-gadget.service
```

Add:

```ini
[Unit]
Description=USB Gadget Service
After=network.target

[Service]
ExecStartPre=/sbin/modprobe libcomposite
ExecStart=/usr/local/bin/usb-gadget.sh
Type=simple
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

Configure `usb0`:

```bash
sudo vi /etc/network/interfaces
```

Add:

```bash
allow-hotplug usb0
iface usb0 inet static
    address 172.20.2.1
    netmask 255.255.255.0
```

Reload the services:

```bash
sudo systemctl daemon-reload
sudo systemctl enable systemd-networkd
sudo systemctl enable usb-gadget
sudo systemctl start systemd-networkd
sudo systemctl start usb-gadget
```

# If you want to give me a tip i accept monero:

```
462ZrXQjmJnD9hpp55ckEMccGGrLrknSFSxesChuPz2FJ4MeYyyaVkYVrynU1tn2ZgSJGJBHm9ZAMA2jzck5RWhK2aUQKA2
```

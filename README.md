# basic-station-setup

Step 1: Boot & Access Raspberry Pi

Purpose: Start Raspberry Pi and access the terminal.
Command:
bash
ssh pi@<raspberry-pi-ip>
# enter username: ______ | enter Password:_____

Step 2: Update System Packages
Purpose: Update existing software to the latest versions.
Command:
sudo apt-get update
sudo apt-get upgrade

Step 3: Enable SPI Interface
Purpose: Enable SPI communication with the LoRa concentrator.
Command:

sudo raspi-config
# Interface Options → SPI → Enable → Exit


Step 4: Install Required Packages**

Purpose: Install tools needed to compile the packet forwarder.
Command:

sudo apt-get install git gcc make


Step 5: Clone & Build LoRa Basic Station
Purpose: Download and compile the LoRa Basics Station software.
Command:
git clone https://github.com/lorabasics/basicstation
cd basicstation
make platform=rpi variant=std ARCH=$(gcc --print-multiarch)

---

Step 6: Verify & Install Binary

Purpose: Ensure the binary is working and place it in its directory.
Command:
./build-rpi-std/bin/station --version
sudo mkdir -p /opt/ttn-station/bin
sudo cp ./build-rpi-std/bin/station /opt/ttn-station/bin/station


Step 7: Derive Gateway EUI

Purpose: Generate a unique EUI based on MAC address.
Command:
export MAC=$(cat /sys/class/net/eth0/address)
export EUI=$(echo $MAC | awk -F: '{print $1$2$3 "fffe" $4$5$6}')
echo "The Gateway EUI is $EUI"

Step 8: Register Gateway on TTN Console

Purpose: Register the gateway on The Things Stack using the EUI.
Command: Done via TTN web interface.

* Use EUI from previous step
* Enable Require authenticated connection

Step 9: Create API Key

Purpose: Generate a secure key for gateway authentication.
**Command**: *Done via TTN web interface.*

* Rights: Write uplink, Read downlink
* Save the key for next step


Step 10: Create Configuration Directory

Purpose: Prepare folder for configuration files.
Command:
sudo mkdir -p /opt/ttn-station/config


Step 11: Set LNS Server Address

Purpose: Define LNS WebSocket server URL.
Command:
echo 'wss://eu1.cloud.thethings.network:8887' | sudo tee /opt/ttn-station/config/tc.uri


Step 12: Save API Key in "tc.key"
Purpose: Add authentication header for LNS connection.
Command:

export API_KEY="NNSXS.XXXX.YYYY"
echo "Authorization: Bearer $API_KEY" | perl -p -e 's/\r\n|\n|\r/\r\n/g' | sudo tee -a /opt/ttn-station/config/tc.key

Step 13: Set Trusted Certificate
Purpose: Allow secure HTTPS connection with LNS.
Command:

sudo ln -s /etc/ssl/certs/ca-certificates.crt /opt/ttn-station/config/tc.trust


Step 14: Create "station.conf" Configuration
Purpose: Set gateway and concentrator hardware settings.
Command:
echo '{
  "SX1301_conf": {
    "lorawan_public": true,
    "clksrc": 1,
    "radio_0": {
      "type": "SX1257",
      "rssi_offset": -166.0,
      "tx_enable": true,
      "antenna_gain": 0
    },
    "radio_1": {
      "type": "SX1257",
      "rssi_offset": -166.0,
      "tx_enable": false
    }
  },
  "station_conf": {
    "routerid": "'"$EUI"'",
    "log_file": "stderr",
    "log_level": "DEBUG",
    "log_size": 10000000,
    "log_rotate": 3,
    "CUPS_RESYNC_INTV": "1s"
  }
}' | sudo tee /opt/ttn-station/config/station.conf

Step 15: Create "start.sh" Script

Purpose: Reset the LoRa concentrator and start the forwarder.
Command:

echo '#!/bin/bash
SX1301_RESET_BCM_PIN=25
echo "$SX1301_RESET_BCM_PIN" > /sys/class/gpio/export
echo "out" > /sys/class/gpio/gpio$SX1301_RESET_BCM_PIN/direction
echo "0" > /sys/class/gpio/gpio$SX1301_RESET_BCM_PIN/value
sleep 0.1
echo "1" > /sys/class/gpio/gpio$SX1301_RESET_BCM_PIN/value
sleep 0.1
echo "0" > /sys/class/gpio/gpio$SX1301_RESET_BCM_PIN/value
sleep 0.1
echo "$SX1301_RESET_BCM_PIN" > /sys/class/gpio/unexport

while [[ $(ping -c1 google.com | grep " 0% packet loss") == "" ]]; do
  echo "[TTN Gateway]: Waiting for internet connection..."
  sleep 30
done

/opt/ttn-station/bin/station
' | sudo tee /opt/ttn-station/bin/start.sh

sudo chmod +x /opt/ttn-station/bin/start.sh


Step 16: Test the Gateway Manually

Purpose: Start the gateway and verify it's receiving packets.
Command:

cd /opt/ttn-station/config
sudo RADIODEV=/dev/spidev0.0 /opt/ttn-station/bin/start.sh

Step 17: Create Systemd Service

Purpose: Automatically start gateway on boot.
Command:

echo '
[Unit]
Description=The Things Network Gateway

[Service]
WorkingDirectory=/opt/ttn-station/config
ExecStart=/opt/ttn-station/bin/start.sh
SyslogIdentifier=ttn-station
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
' | sudo tee /lib/systemd/system/ttn-station.service


Step 18: Enable and Start the Service
Purpose: Run gateway as a background service.
Command:

sudo systemctl enable ttn-station
sudo systemctl start ttn-station

Step 19: Monitor Logs

Purpose: Check gateway status and logs.
Command:

sudo journalctl -f -u ttn-station

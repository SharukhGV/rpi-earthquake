# Raspberry Pi Sensor Monitoring Application

This guide explains how to set up a Raspberry Pi to monitor a vibration sensor and a raindrop sensor. The application is built with Flask and exposed to the internet using ngrok. All steps are detailed for clarity. While not a seismograph, the vibration sensor can pick up anomalous vibrations. If this project is created and placed in multiple locations, it can verify if a precursor tremor was initiated by a potential earthquake, allowing time to prepare for the catastrophe.

**Why Combine Flask and ngrok?**

Flask handles the local data processing and API creation, while ngrok bridges the gap between the local application and the internet.

This combination allows you to quickly deploy and access a local IoT project remotely, even in networks where traditional port forwarding or static IP solutions are not feasible.

---

## Prerequisites

1. A Raspberry Pi with GPIO capabilities.
2. **Vibration Sensor** and **Raindrop Sensor**.
3. Internet access for the Raspberry Pi.
4. SD card for the rpi you are using
5. Other computer if using SSH
6. Any other hardware needed
---

## Hardware Setup

To set up a Raspberry Pi Lite (without a desktop environment) with SSH enabled, you need to prepare the system, configure SSH, and set up Wi-Fi. Below is a detailed guide, including creating the required files for SSH and Wi-Fi configuration before even booting the Raspberry Pi.

## Setting Up a Raspberry Pi Lite with SSH (Headless Configuration)

This guide explains how to set up Raspberry Pi OS Lite for headless operation, enabling SSH and optionally configuring Wi-Fi before the first boot.

---

## Step 1: Download and Flash Raspberry Pi OS Lite

1. **Download Raspberry Pi OS Lite**:
   - Go to the [Raspberry Pi Downloads page](https://www.raspberrypi.com/software/operating-systems/).
   - Download the "Raspberry Pi OS Lite" image.

2. **Flash the OS to a microSD card**:
   - Use a tool like [Raspberry Pi Imager](https://www.raspberrypi.com/software/) or [balenaEtcher](https://www.balena.io/etcher/).
   - Select the downloaded image and the microSD card.
   - Start the flashing process.


NOTE: If using raspberry pi imager, you DO NOT need to install the OS beforehand.


## Wiring the Sensors

1. **Vibration Sensor:**
   - Connect **VCC** to **5V** pin on the Raspberry Pi.
   - Connect **GND** to a **GND** pin.
   - Connect **OUT** to **GPIO 17** (BCM mode).

2. **Raindrop Sensor:**
   - Connect **VCC** to **5V** pin.
   - Connect **GND** to a **GND** pin.
   - Connect **OUT** to **GPIO 27** (BCM mode).


---


## Step 2: Enable SSH and Configure Wi-Fi

1. **Access the boot partition**:
   - Once the flashing is complete, insert the microSD card into your computer (should already be in the computer if you just flashed the SD card)
   - Open the **boot** partition (it should be the only accessible partition, unless on a linux system. Sometimes the *bootfs* folder).

2. **Enable SSH & Configure Wifi**:
   - Create an empty file named `ssh` (no extension) in the `boot` directory:
     - Create and save an empty file named `ssh` in the `boot` directory (no file extension).

   - Then create a file in the same boot directory named **wpa_supplicant.conf** and add the following content, replacing placeholders with your country code, SSID, and password:
     ```plaintext
     country=US
     ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
     update_config=1

     network={
         ssid="YourWiFiSSID"
         psk="YourWiFiPassword"
     }
     ```
---

## Step 3: Boot the Raspberry Pi

1. **Insert the microSD card**:
   - Insert the prepared microSD card into the Raspberry Pi.

2. **Power on the Raspberry Pi**:
   - Connect power to the Raspberry Pi. It will boot and automatically enable SSH.

---

<!-- ## Step 4: Find the Raspberry Pi’s IP Address

1. **Access your router’s admin interface**:
   - Look for connected devices to find the Raspberry Pi’s IP address.

2. **Use network discovery tools** (if router access isn’t possible):
   - Run the following command on another computer in the same network:
     ```bash
     nmap -sn 192.168.1.0/24
     ```
   - Replace `192.168.1.0/24` with your local network's IP range.

--- -->

## Step 4: SSH into the Raspberry Pi
**(Basically control Raspberry Pi from another computer)**

1. **Open an SSH terminal**:
   - On Linux/macOS:
     ```bash
     ssh pi@<raspberry_pi_ip>
     ```
   - On Windows, use a tool like [PuTTY](https://www.putty.org/) or the Windows Terminal:
     ```bash
     ssh pi@<raspberry_pi_ip>
     ```



2. **Default Credentials**:
   - Username: `pi`
   - Password: `raspberry`


NOTE: if you setup the rpi when creating the SD card, just use the credentials from there. I recommend writing this information down or taking a screenshot.

Example: if you setup your hostname to be raspberrypi.local, and your username is computer instead of pi, you will do ssh computer@raspberrypi.local, then enter your password when prompted.



## Step 5: All Software Setup Using the SSH Setup

Step 1: Enable SSH (Optional for Remote Access) & then enter password when prompted

    ssh pi@<your_raspberry_pi_ip>

Step 2: Install Dependencies

Update system packages:

    sudo apt update && sudo apt upgrade -y

Install required software:

    sudo apt install python3 python3-pip -y

Step 3: Set Up Python Virtual Environment

Install venv module if not already installed:

    sudo apt install python3-venv -y

Create and activate a virtual environment:

    python3 -m venv myenv
    source myenv/bin/activate

Install required Python packages:

    pip install flask flask-cors RPi.GPIO

Step 4: Create the Flask Application

Create the project directory:

    mkdir ~/sensor-monitoring && cd ~/sensor-monitoring

Create app.py using nano:

    nano app.py

Paste the following code into app.py:

```
from flask import Flask, jsonify
import RPi.GPIO as GPIO
import time
from flask_cors import CORS

# GPIO Pin Configurations
VIBRATION_PIN = 17
RAINDROP_PIN = 27

# Initialize GPIO
GPIO.setmode(GPIO.BCM)
GPIO.setup(VIBRATION_PIN, GPIO.IN)
GPIO.setup(RAINDROP_PIN, GPIO.IN)

# Initialize Flask app
app = Flask(__name__)
CORS(app)

# Vibration Sensitivity Parameters
TREMOR_THRESHOLD = 3
TREMOR_TIME_FRAME = 2

def detect_tremors():
    start_time = time.time()
    vibration_count = 0
    while time.time() - start_time < TREMOR_TIME_FRAME:
        if GPIO.input(VIBRATION_PIN):
            vibration_count += 1
            time.sleep(0.05)
    return vibration_count >= TREMOR_THRESHOLD

def read_raindrop_sensor():
    return "Rain detected" if GPIO.input(RAINDROP_PIN) == 0 else "No rain"

@app.route('/sensor-data', methods=['GET'])
def sensor_data():
    try:
        return jsonify({
            "vibration": detect_tremors(),
            "raindrop": read_raindrop_sensor()
        })
    except Exception as e:
        return jsonify({"error": str(e)})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)

```


Save and exit nano (Ctrl+X, Y, Enter).

Test the Flask app:

    python3 app.py

---

## Step 6: Create the Flask Service File to start the python script on startup: ##

Open a terminal on the Raspberry Pi.

Use a text editor to create a new service file:

    sudo nano /etc/systemd/system/flaskapp.service

Add the following content to the file, replacing placeholders with your specific details:

```
[Unit]
Description=Flask App for Earthquake Sensor
After=network.target

[Service]
ExecStart=/bin/bash -c "source /home/computer/myenv/bin/activate && python3 /home/computer/sensor_earthquak>
User=computer
Group=computer
WorkingDirectory=/home/computer/
Restart=always
Environment=PATH=/usr/bin:/usr/local/bin:/home/computer/myenv/bin
Environment=VIRTUAL_ENV=/home/computer/myenv

[Install]
WantedBy=multi-user.target
```

Replace /path/to/your/flask/app with the directory of your Flask application (the python script for your rpi).

Ensure the ExecStart path points to your Python executable and Flask app.

2. Reload and Enable the Service

After creating the service file, reload the systemd manager and enable the service:

Reload systemd to recognize the new service:

    sudo systemctl daemon-reload

Enable the Flask service to start on boot:

    sudo systemctl enable flaskapp.service

Start the service immediately:

    sudo systemctl start flaskapp.service

Check the status to ensure it’s running correctly:

    sudo systemctl status flaskapp.service

3. Test the Automation

Reboot the Raspberry Pi to test if the Flask application starts automatically:

    sudo reboot

After the system boots up, check if the Flask app is running:

    sudo systemctl status flaskapp.service


## Step 7: Install and Configure Ngrok

Download and install ngrok:

      curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc \
     | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null \
     && echo "deb https://ngrok-agent.s3.amazonaws.com buster main" \
     | sudo tee /etc/apt/sources.list.d/ngrok.list \
     && sudo apt update \
     && sudo apt install ngrok

NOTE: visit this link for latest install: https://download.ngrok.com/linux?tab=install


Authenticate ngrok: (Auth code and script for it can be found on your online ngrok account)

    ngrok config add-authtoken <your-auth-token>

Run ngrok manually:

    ngrok http 5000

## Step 8: Automate Ngrok with Systemd

Create a systemd service file:

    sudo nano /etc/systemd/system/ngrok.service

Paste the following content:

```
[Unit]
Description=Ngrok Tunnel for Flask App
After=network.target

[Service]
ExecStart=/usr/local/bin/ngrok http 5000
Restart=on-failure
User=computer
WorkingDirectory=/home/pi
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target

```
    
**NOTE: make sure your username is pi, otherwise replace pi with your username**

Save and exit (Ctrl+X, Y, Enter).


Enable and start the service:

```
sudo systemctl daemon-reload
sudo systemctl enable ngrok.service
sudo systemctl start ngrok.service
```


Check the status:

    sudo systemctl status ngrok.service

## Step 9: Access Sensor Data

Go to your Ngrok dashboard where you obtained the Auth token. Navigate to Endpoints, and your public URL should be there. If you automate the scripts to run on startup and if you restart the pi, this url will change. Keep in mind to prevent the initial "Visit Site" page from popping up, you will need a subscription.

Use the ngrok URL (e.g., https://<ngrok-id>.ngrok.io/**sensor-data**) to fetch sensor data.
# Contents

1. [ Flashing IoT Devices with Tasmota using Tuya-Convert ](#tuyaconvert)
2. [ Home Assistant ](#homeassistant)

<a name="tuyaconvert"></a>

# Flashing IoT Devices with Tasmota Using Tuya-Convert

### Sources

https://github.com/ct-Open-Source/tuya-convert  
https://www.youtube.com/watch?v=dt5-iZc4_qU  
https://tasmota.github.io/docs/About/

https://tasmota.github.io/docs/Upgrading/  
http://ota.tasmota.com/tasmota/

https://templates.blakadder.com/index.html  
https://templates.blakadder.com/howto.html  
https://templates.blakadder.com/gosund_UP111.html

### Device Compatibility

This documents what I did to successfully flash the [Gosund (UP111) UK smart plugs](https://www.amazon.co.uk/gp/product/B0856T6TJC/) over-the-air (OTA) with Tasmota firmware.

These smart plugs were bought from Amazon on January 2021. Devices purchased later may have had their firmware updated which prevents this method from working, so do your own research online to check your device's compatibility.

### Installing Tuya-Convert

Carry out this process on a fresh Raspberry Pi OS install by swapping out your micro SD card, or re-formatting with a fresh OS for this purpose.

Perform a `sudo apt update` and install git, `sudo apt install git -y`.

Clone the Tuya-Convert git repo, `git clone https://github.com/ct-Open-Source/tuya-convert`, then `cd tuya-convert`, and run `./install_prereq.sh`.

### Flashing the IoT device

Start the flashing process by running `./start_flash.sh`. You'll be presented with a warning: when you are happy to proceed, type "yes" and enter. It'll ask you if it can terminate dnsmasq process to free up port 53, type "y". It'll ask you to terminate mosquitto to free up port 1883, type "y". The script should have turned the raspberry pi into a WiFi access point with SSID "vtrust-flash".

Connect a smartphone (or any other device) to the "vtrust-flash" WiFi access point.

Plug your IoT device in and put it in autoconfig/smartconfig/pairing mode - you should know it's in the right mode when the LED starts blinking. This is usually done by pressing and holding the primary button of the device. In my case, I pressed the power button of the plug for 5 seconds.

Once the device is in pairing mode, press enter to start flashing.

After some setup, it'll then prompt you to ask which image you'd like to load onto the device. For Tasmota, select the tasmota.bin option by typing "2", then confirm with "y". After it's done, we have Tasmota on our device! Now we can start configuring it.

> **Remember to keep your device plugged in for the next step.**

### Configure WiFi on Tasmota

The device should now broadcast a Wifi access point with SSIP `tasmota-xxxx`. Connect to it with your phone or another device to configure Tasmota.

One connected to the Tasmota Wifi, you can configure your home WiFi credentials. Make sure to double check the credentials before pressing "Save".

After pressing "Save", the device should restart and automatically connect to your home network. Find the IP address of the IoT device on your home network via your router or a network analyser app like Fing and connect to it to see the device's Tasmota web page.

### Upgrade Tasmota Firmware if required

Now on my device is Tasmota v8.1.0.2. However the configuration template (see next section) for my device is for Tasmota v9.1+. So next we'll need to upgrade the firmware. Note that we should be cautious of upgrading firmware on an already working device.

There is a [defined upgrade path](https://tasmota.github.io/docs/Upgrading/#upgrade-flow) which we must follow, so we'll upgrade in steps from v8.1.0 to v8.5.1 to v9.1 to current latest. Download the files by clicking on those versions on the [Tasmota Upgrade page](https://tasmota.github.io/docs/Upgrading/#upgrade-flow).

On the device's Tasmota web page, click on "Firmware Upgrade", upload the downloaded v8.5.1 `tasmota-lite.bin` file in the "Upgrade by file upload" section, and "Start upgrade". After some time, you should see it say "Upload Successful". It will then reboot.

Repeat the above for v9.1. Note that the downloaded v9.1 file `tasmota-lite.bin.gz` is gzip compressed (new feature).

Lastly, either copy the URL to the non-lite and latest `tasmota.bin.gz` file, which can be found [here](http://ota.tasmota.com/tasmota/), in the "Upgrade by web server" section, or download it and uploading the file as before, then clicking "Start upgrade".

### GPIO Configuration on Tasmota

> See this link for screenshots of the below steps: https://templates.blakadder.com/howto.html

On the device's Tasmota web page, go to "Configuration", then "Configure Other".

Search for your specific device's GPIO configuration on [Tasmota templates](https://templates.blakadder.com/index.html). My smart plug's configuration template is found [here](https://templates.blakadder.com/gosund_UP111.html).

Paste in the device's GPIO configuration template string in "Templates" input field, check "Activate", then press "Save". The device should now reboot with the module name specified in the template already selected: in this case `Gosund UP111 Module`.

### Troubleshooting: Issue connecting to Tuya-Convert's vtrust-flash access point

https://github.com/ct-Open-Source/tuya-convert/issues/551

Never really solved it. The first connect to vtrust-flash worked for me so I flashed all the plugs at once because the subsequent attempts to connect to vtrust-flash continually fails.

### Troubleshooting: Tasmota has issues connecting to Asus Routers

https://github.com/arendst/Tasmota/issues/7770

Not fully solved but [adding Wireless Mac Filters](https://github.com/arendst/Tasmota/issues/7770#issuecomment-767198267) seem to be work for a while but is still not 100% stable.

Also changed RTS Threshold from 2347 to 2000 but not sure if that has a real difference.

<a name="homeassistant"></a>

# Home Assistant

### Sources

https://www.home-assistant.io/getting-started/  
https://www.home-assistant.io/hassio/installation/

### Home Assistant OS

The more beginner friendly way to get started with home assistant is installing the Home Assistant OS on the micro-SD card for your Raspberry Pi. It has lots of community created add-ons available where you would otherwise have to configure yourself if you didn't use the OS. Follow the [instructions here](https://www.home-assistant.io/getting-started/). Instead of balenaEtcher to write the image to the SD card, I used [Raspberry Pi Imager](https://www.raspberrypi.org/software/).

> The alternative is to install Home Assistant core on your Raspberry Pi OS using, say, Docker Compose by following [these instructions](https://www.home-assistant.io/docs/installation/docker/#docker-compose). This is not for beginners because you will have to set up everything else yourself (such as an MQTT broker), where it might be available as an Addon in Home Assistant OS.

### Mosquitto MQTT broker add-on

https://github.com/home-assistant/addons/blob/master/mosquitto/DOCS.md

### File Editor add-on

To edit configuration.yml, install the File Editor add-on from the store: https://www.home-assistant.io/getting-started/configuration/


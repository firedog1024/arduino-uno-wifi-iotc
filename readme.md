# Arduino Uno WiFi Rev.2 with Azure IoT Central

## Whats this about

We have been experimenting with a number of Arduino boards and getting them connected to Azure IoT Central to send sensor data to Azure.  We did this with the MKR1000 and MKR1010 with good success (see https://github.com/firedog1024/mkr1000-iotc).  Then we asked ourselves can we go smaller?  The Arduino Uno WiFi Rev2 is an Arduino Uno class device with an added U-blox WiFi chip, the specs look like this:

|   |   |
|---|---|
| Microcontroller  | ATMEGA4809  |
| Flash Memory  | 48 KB   |
| SRAM  | 6,144 Bytes   |
| EEPROM  | 256 Bytes   |
| Clock Speed  | 16 MHz  |
|   |   |


As you can see the specs on this board are not great so the challenge is to enable the device to talk to IoT Central and/or IoT Hub via MQTT over WiFi and support sending telemetry, sending and recieving Digital Twin updates, and responding to commands from the cloud all within 6KB of SRAM.

The code here enables all that functionality in about 380 lines of core code.  When built the code consumes about 40KB (81%) of flash and 670 bytes (10%) of dynamic memory.  

## Purchasing an Arduino Uno WiFi Rev2

If you dont have an Arduino Uno WiFi Rev2 you can purchase one from Arrow 

![Arrow logo](https://github.com/firedog1024/arduino-uno-wifi-iotc/blob/master/assets/arrow-logo.png)

* Arduino Uno WiFi Rev2 https://www.arrow.com/en/products/abx00021/arduino-corporation

## Features

* Uses the onboard LSM6DS IMU chip to get accelermoter and gyroscopic sensor data along with temperature data.  
* Uses simple MQTT library to communicate to Azure IoT Central
* Simple code base designed to illustrate how the code works and encourage hacking (~400 lines of core code w/ comments)
* IoT Central features supported
    * Telemetry data - Accelerometer and Gyroscopic (X,Y,Z axis), and temperature
    * Properties - Device sends a die roll number every 15 seconds
    * Settings - Change the fan speed value and see it displayed in the serial moitor and acknowledged back to IoT Central
    * Commands - Send a message to the device and see it displayed as morse code on the device LED

## Installation

Run:

```
git clone https://github.com/firedog1024/arduino-uno-wifi-iotc.git   
```

## Prerequisite

Install the Arduino IDE and the necessary drivers for the Arduino Uno WiFi Rev2 board and ensure that a simple LED blink sketch compiles and runs on the board. Follow the getting started guide here https://www.arduino.cc/en/Guide/ArduinoUnoWiFiRev2.

This code requires a couple of libraries to be installed for it to compile. To install an Arduino library open the Arduino IDE and click the "Sketch" menu and then "Include Library" -> "Manage Libraries". In the dialog filter by the library name below and install the latest version. For more information on installing libraries with Arduino see https://www.arduino.cc/en/guide/libraries.

* Install library "WiFiNINA"
* Install library "Accelerometer And Gyroscope LSM6DS3" by Seeed Studio
* Install library "PubSubClient"

Note - We need to increase the payload size limit in PubSubClient to allow for the larger size of MQTT messages from the Azure IoT Hub. Open the file at %HomePath%\Documents\Arduino\libraries\PubSubClient\src\PubSubClient.h in your favorite code editor. Change the line (line 26 in current version):

```
#define MQTT_MAX_PACKET_SIZE 128
```

to:

```
#define MQTT_MAX_PACKET_SIZE 2048
```

Save the file and you have made the necessary fix. The size probably does not need to be this large but I have not found the crossover point where the size causes a failure. Fortunately the buffer size does not cause an out of memory event, but if you modify the code and consume more memory this may become an issue and you might consider adjusting down the buffer size.

To connect the device to Azure IoT Central you will need to provision an IoT Central application. This is free for seven days but if you already have signed up for an Azure subscription and want to use pay as you go IoT Central is free as long as you have no more than five devices and do not exceed 1MB per month of data.

Go to https://apps.azureiotcentral.com/ to create an application (you will need to sign in with a Microsoft account identity you may already have one if you use Xbox, office365, Windows 10, or other Microsoft services).

* Choose Trial or Pay-As-You-Go.
* Select the Sample DevKits template (middle box)
* Provide an application name and URL domain name
If you select Pay-As-You-Go you will need to select your Azure subscription and select a region to install the application into. This information is not needed for Trial.
* Click "Create"

You should now have an IoT Central application provisioned so lets add a real device. Click Device Explorer on the left. You will now see three templates in the left hand panel (MXChip, Raspberry Pi, Windows 10 IoT Core). We are going to use the MXChip template for this exercise to prevent having to create a new template. Click "MXChip" and click the "+V" icon on the toolbar, this will present a drop down where we click "Real" to add a new physical device. Give a name to your device and click "Create".

You now have a device in IoT Central that can be connected to from the Arduino Uno WiFi device. Proceed to wiring and configuration.

## Wiring

Nothing to wire as we can use the builtin LSM6DS sensor for our telemetry data.

![wiring diagram for Arduino Uno WiFi Rev2](https://github.com/firedog1024/arduino-uno-wifi-iotc/blob/master/assets/uno_wifi_rev2.jpg)

## Configuration

We need to copy some values from our new IoT Central device into the configure.h file so it can connect to IoT Central. 

Due to current issues using the Azure Device Provisioning Service with this device we need to generate an IoT Hub connection string using an external process. We have a tool called DPS KeyGen that given device and application information can generate a connection string to the IoT Hub. Lets grab the tool:

```
git clone https://github.com/Azure/dps-keygen.git
```

in the cloned directory there is a bin folder and inside that three folders for the OS's Windows, OSX, and Linux. Go into the correct folder for your operating system (for Windows you will need to unzip the .zip file in the folder). Using the command line UX type:

cd dps-keygen\bin\windows\dps_cstr
dps_cstr <scope_id> <device_id> <primary_key>
for the values in <> substitute in values taken from clicking the device you created at the end of the Prerequisite step and click the "Connect" link to get the connection information for the device. We can then copy "Scope ID', Device ID", and "Primary Key" values into the respective positions in the command line.

After executing the command above with the substituted values you should see a connection string displayed on the command line.

```
.\dps_cstr 0ne0003D8B4 mkr1000 zyGZtz6r5mqta6p7QXOhlxR1ltgHS0quZPgIYiKb9aE=
...
Registration Information received from service: iotc-fc41f2e1-fc58-40c0-ac56-f7b05c53f70e.azure-devices.net!
Connection String:
HostName=iotc-fc41f2e1-fc58-40c0-ac56-f7b05c53f70e.azure-devices.net;DeviceId=mkr1000;SharedAccessKey=zyGZtz6r5mqta6p7QXOhlxR1ltgHS0quZPgIYiKb9aE=
```

We need to copy this value from the command line to the configure.h file and paste it into the iotConnStr[] line, the resulting line will look something like this.

```
static char iotConnStr[] = "HostName=iotc-fc41f2e1-fc58-40c0-ac56-f7b05c53f70e.azure-devices.net;DeviceId=mkr1000;SharedAccessKey=zyGZtz6r5mqta6p7QXOhlxR1ltgHS0quZPgIYiKb9aE=";
```

You will also need to provide the Wi-Fi SSID (Wi-Fi name) and password in the configure.h

```
// Wi-Fi information
static char wifi_ssid[] = "<replace with Wi-Fi SSID>";
static char wifi_password[] = "<replace with Wi-Fi password>";
```

## Compiling and running

Now that you have configured the code with IoT Central and Wi-Fiinformation we are ready to compile and run the code on the device.

Load the uno_wifi.ino file into the Arduino IDE and click the Upload button on the toolbar. The code should compile and be uploaded to the device. In the output window you should see:

```
Sketch uses 40090 bytes (81%) of program storage space. Maximum is 49152 bytes.
Global variables use 673 bytes (10%) of dynamic memory, leaving 5471 bytes for local variables. Maximum is 6144 bytes.
Uploading...
[Done] Uploaded the sketch: uno_wifi.ino
```

Note: You may see some warnings from the wire.h include, these appear to be harmless and unrelated to the .ino code as far as I can tell.

The code is now running on the device and should be sending data to IoT Central. We can look at the serial port monitor by clicking the Tool menu -> Serial Monitor (you may need to change the baud rate to 115200). You should start to see output displayed in the window.

### Telemetry:

If the device is working correctly you should see output like this in the serial monitor that indicates data is successfully being transmitted to Azure IoT Central:

```
Firmware version is 1.2.1
Attempting to connect to Wi-Fi SSID: AZUREIOTS 
Starting IoT Hub connection
===> mqtt connected
Current state of device twin:
{
  "desired": {
    "setCurrent": {
      "value": 100
    },
    "setVoltage": {
      "value": 100
    },
    "fanSpeed": {
      "value": 200
    },
    "activateIR": {
      "value": false
    },
    "$version": 15
  },
  "reported": {
    "dieNumber": 4,
    "fanSpeed": {
      "value": 200,
      "statusCode": 200,
      "status": "completed",
      "desiredVersion": 15
    },
    "$version": 2714
  }
}

Sending telemetry ...
	{"temp": 24.56, "accelerometerX": 23.00, "accelerometerY": 4.00, "accelerometerZ": 2087.00, "gyroscopeX": 42.00, "gyroscopeY": -49.00, "gyroscopeZ": -59.00}
Sending telemetry ...
	{"temp": 24.63, "accelerometerX": 19.00, "accelerometerY": -1.00, "accelerometerZ": 2091.00, "gyroscopeX": 42.00, "gyroscopeY": -53.00, "gyroscopeZ": -62.00}
Sending digital twin property ...
Sending telemetry ...
	{"temp": 24.63, "accelerometerX": 16.00, "accelerometerY": -8.00, "accelerometerZ": 2089.00, "gyroscopeX": 41.00, "gyroscopeY": -53.00, "gyroscopeZ": -62.00}
--> IoT Hub acknowledges successful receipt of twin property: 1
Sending telemetry ...
	{"temp": 24.69, "accelerometerX": 14.00, "accelerometerY": -4.00, "accelerometerZ": 2090.00, "gyroscopeX": 42.00, "gyroscopeY": -49.00, "gyroscopeZ": -62.00}
```

Now that we have data being sent lets look at the data in our IoT Central application. Click the device you created and then select the temperature and humidity telemetry values in the Telemetry column. You can turn on and off telemetry values by clicking on the eyeballs. We are only sending temperature and humidity so no other telemetry items will be active. You should see a screen similar to this:

![telemetry on Arduino Uno WiFi Rev2](https://github.com/firedog1024/arduino-uno-wifi-iotc/blob/master/assets/telemetry.png)

### Properties:

The device is also updating the property "Die Number", click on the "Properties" link at the top and you should see the value in the Die Number change about ever 15 seconds.

![properties on Arduino Uno WiFi Rev2](https://github.com/firedog1024/arduino-uno-wifi-iotc/blob/master/assets/properties.png)

### Settings:

The device will accept settings and acknowledge the receipt of the setting back to IoT Central. Go to the "Settings" link at the top and change the value for Fan Speed (RPM), then click the "Update" button the text below the input box will briefly turn red then go green when the device acknowledges receipt of the setting. In the serial monitor the following should be observed:

```
Fan Speed setting change received with value: 200
{"fanSpeed":{"value":200,"statusCode":200,"status":"completed","desiredVersion":9}}
--> IoT Hub acknowledges successful receipt of twin property: 1
```

The settings screen should look something like this:

![settings on Arduino Uno WiFi Rev2](https://github.com/firedog1024/arduino-uno-wifi-iotc/blob/master/assets/settings.png)

### Commands:

We can send a message to the device from IoT Central. Go to the "Commands" link at the top and enter a message into the Echo - Value to display text box. The message should consist of only alpha characters (a - z) and spaces, all other characters will be ignored. Click the "Run" button and watch your device. You should see the LED blink morse code. If you enter SOS the led should blink back ...---... where dots are short blinks and dashes slightly longer :-)

![commands on Arduino Uno WiFi Rev2](https://github.com/firedog1024/arduino-uno-wifi-iotc/blob/master/assets/commands.png)


The morse code blinking LED is here on the Arduino Uno WiFi Rev2

![morse code LED on Arduino Uno WiFi Rev2](https://github.com/firedog1024/arduino-uno-wifi-iotc/blob/master/assets/uno_wifi_rev2_morse.jpg)


## What Now?

You have the basics now go play and hack this code to send other sensor data to Azure IoT Central. If you want to create a new device template for this you can learn how to do that with this documentation https://docs.microsoft.com/en-us/azure/iot-central/howto-set-up-template.

How about creating a rule to alert when the temperature or humidity exceed a certain value. Learn about creating rules here https://docs.microsoft.com/en-us/azure/iot-central/tutorial-configure-rules.

For general documentation about Azure IoT Central you can go here https://docs.microsoft.com/en-us/azure/iot-central/.

Have fun!
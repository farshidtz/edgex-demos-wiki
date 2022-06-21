## Overview

The demo involves setting up EdgeX to collect ambient sensing measurements from an MQTT broker. It will showcase the following aspects of EdgeX:
- Deployment using Snaps, allowing automatic updates out of the box
- Devices (sensors) and their resources (sensor measurement classes) added to EdgeX's Metadata store
- Sensor measurements ingested into EdgeX, for storage and further processing

Required hardware:
- Raspberry Pi 4 or another arm64/amd64 computer
- DHT22/DHT11 temperature and humidity sensor modules

We use the following:
- Raspberry Pi 4 running arm64 Ubuntu Core (hostname: `jupiter.local`)
- Raspberry Pi Zero W running armhf Raspberry Pi OS (hostname: `pluto.local`)
- DHT22 sensor connected to the Raspberry Pi Zero W

To interact with the APIs, we use the [HTTPie HTTP Client](https://snapcraft.io/httpie) which provides the `http` command. It is easy to use, prints pretty JSON output, and is available as a Snap! You can use any other HTTP client.

## Installing the EdgeX platform

The [EdgeX platform snap](https://snapcraft.io/edgexfoundry), called `edgexfoundry`, contains the core and several other components. To install the latest stable:
```bash
sudo snap install edgexfoundry
```

After installation, a set of Secret Store tokens get created which are valid for 1 hour. 
This means if these tokens aren't used and refreshed within 1 hour, they will get expired.

Use the following commands to extend the validity to 72 hours while working on the demo:
```bash
sudo snap set edgexfoundry app-options=true
sudo snap set edgexfoundry apps.security-secretstore-setup.config.tokenfileprovider-defaulttokenttl=72h
sudo snap restart edgexfoundry.security-secretstore-setup
```

## Setting up sensing input

We assume that sensors have been setup and the measurements for temperature and humidity are published to an MQTT broker at an interval. Setting up sensors and publishing their data to the broker is beyond the scope of this demo. 

We use the following:
- [Mosquitto](https://snapcraft.io/mosquitto) MQTT broker
- [DHT-MQTT](https://github.com/farshidtz/dht-mqtt) for reading data DHT11/DHT22 sensors on a Raspberry Pi and publishing to the broker

Let's subscribe to the broker to verify the flow of sensing data:
```bash
mosquitto_sub -h jupiter.local -t "#" -v
```
```
pluto/dht22/temperature 26.0
pluto/dht22/humidity 49.2
pluto/dht22/temperature 26.0
pluto/dht22/humidity 49.5
...
```

The topics have `.../<device>/<resource>` format. The payloads are the raw measurements, without any envelop object. That's exactly how we want them to be!

## Setting up the EdgeX MQTT service

Install the [EdgeX Device MQTT](https://snapcraft.io/edgex-device-mqtt) service:
```bash
sudo snap install edgex-device-mqtt
```

The service is NOT started by default. We need to configure and then start it.

Enter the snap's resource directory:
```bash
cd /var/snap/edgex-device-mqtt/current/config/device-mqtt/res
```
All following commands are relative to this path, so make sure you don't change directory.

Remove the default profile and device so that they aren't loaded by the service:
```bash
sudo rm profiles/mqtt.test.device.profile.yml
sudo rm devices/mqtt.test.device.toml
```
Those files are still available as read-only under `/snap/edgex-device-mqtt/current/config/device-mqtt/res/`.

Add `profiles/temperature-humidity-sensor.yml`:
```yml
name: "temperature-humidity-sensor"
deviceResources:
- name: temperature
  properties:
    valueType: "Float32"
    readWrite: "R"
- name: humidity
  properties:
    valueType: "Float32"
    readWrite: "R"
```

Add `devices/dht22.toml`:
```toml
[[DeviceList]]
  Name = "dht22"
  ProfileName = "temperature-humidity-sensor"
  [DeviceList.Protocols]
    [DeviceList.Protocols.mqtt]
       CommandTopic = "CommandTopic"
```

Update `configuration.toml` with snap option:
```bash
sudo snap set edgex-device-mqtt app-options=true
sudo snap set edgex-device-mqtt config.mqttbrokerinfo-usetopiclevels=true
sudo snap set edgex-device-mqtt config.mqttbrokerinfo-incomingtopic=pluto/#
```

This is equivalent to updating the following entries in the config file before service has started for the first time:
```toml
UseTopicLevels = true
IncomingTopic = "pluto/#"
```

With the configurations in place, we can now start the service:
```bash
sudo snap start edgex-device-mqtt
```

> 🛑 **Debug**  
> Check the logs to see if there are errors:
> ```bash
> sudo snap logs -f edgex-device-mqtt
> ```
> To see if all messages pass through, enable the debugging first as below and then query the logs:
> ```bash
> sudo snap set edgex-device-mqtt config.writable-loglevel=DEBUG
> sudo snap restart edgex-device-mqtt
> ```

> ℹ **Tip**  
> To change the device/device profile after service has started: Update the local files, then delete the device/profile from core-metadata, and restart as below:
>
> ```bash
> # Delete device:
> http DELETE http://localhost:59881/api/v2/device/name/example-camera
>
> # Delete profile, if modified:
> http DELETE http://localhost:59881/api/v2/deviceprofile/name/USB-Camera-General
>
> # Restart:
> sudo snap restart edgex-device-usb-camera
> ```

## Querying store data from EdgeX
Let's query EdgeX Core Data to check if measurements (readings) are being added via Device MQTT. We use the `readings` endpoint and query just 2 records:
```bash
http http://localhost:59880/api/v2/reading/device/name/dht22?limit=2
```
```
HTTP/1.1 200 OK
Content-Length: 494
Content-Type: application/json
Date: Tue, 21 Jun 2022 06:55:36 GMT
X-Correlation-Id: a94fadad-ad47-46de-bcb6-b4c2da56e7ab

{
    "apiVersion": "v2",
    "readings": [
        {
            "deviceName": "dht22",
            "id": "7601bcfe-0ef2-4480-a2c9-812f7b97e11c",
            "origin": 1655794520619530651,
            "profileName": "temperature-humidity-sensor",
            "resourceName": "humidity",
            "value": "5.050000e+01",
            "valueType": "Float32"
        },
        {
            "deviceName": "dht22",
            "id": "1db797f6-27b6-48f5-bae2-8265d204baf6",
            "origin": 1655794520617060775,
            "profileName": "temperature-humidity-sensor",
            "resourceName": "temperature",
            "value": "2.560000e+01",
            "valueType": "Float32"
        }
    ],
    "statusCode": 200,
    "totalCount": 97781 <-- This thing has been running for a while!
}
```

## Visualizing sensor data with Grafana
Using pre-setup dashboard

## Creating an OS image with all the above
Well, except for the sensing part.

To add:
- UC Model assertion for the above and a config provider
  - Show config provider for device-mqtt
  - Show model assertion
  - Build image

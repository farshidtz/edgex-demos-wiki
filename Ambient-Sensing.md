## Overview

The demo involves setting up EdgeX to collect ambient sensing measurements from an MQTT broker.

1. Install the platform

The [platform snap](https://snapcraft.io/edgexfoundry), called `edgexfoundry`, contains the core and several other components. To install the latest stable:
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

2. Sensing input

We assume that sensors have been setup and the measurements for temperature and humidity are published to an MQTT broker at an interval. Setting up sensors and publishing their data to the broker is beyond the scope of this demo. 

We use the following:
- [Mosquitto](https://snapcraft.io/mosquitto) MQTT broker
- [DHT-MQTT](https://github.com/farshidtz/dht-mqtt) for reading data DHT11/DHT22 sensors on a Raspberry Pi and publishing to the broker

Let's subscribe to the broker to verify the flow of sensing data:
```
$ mosquitto_sub -h 192.168.0.129 -t "#" -v
pluto/dht22/temperature 26.0
pluto/dht22/humidity 49.2
pluto/dht22/temperature 26.0
pluto/dht22/humidity 49.5
...
```

The topics are `pluto/dht22/temperature` and `pluto/dht22/humidity`. The payloads are the raw measurements, without any envelop object. That's exactly how we want them to be!

3. Setup device mqtt (install, configure)

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

Remove the default profile and device so that they aren't loaded:
```bash
sudo rm profiles/mqtt.test.device.profile.yml
sudo rm devices/mqtt.test.device.toml
```
Don't worry about backing up. Those files are still available as read-only under `/snap/edgex-device-mqtt/current/config/device-mqtt/res/`.

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

> **Debug**  
> Check the logs to see if there are errors:
> ```bash
> sudo snap logs -f edgex-device-mqtt
> ```
> To see if all messages pass through, enable the debugging first as below and then query the logs:
> ```bash
> sudo snap set edgex-device-mqtt config.writable-loglevel=DEBUG
> sudo snap restart edgex-device-mqtt
> ```

> **Tip**  
> To change the device/device profile after service has started: Update the local files, then delete the device/profile from core-metadata, and restart as below:
>
> ```bash
> # Delete device:
> curl -X DELETE http://localhost:59881/api/v2/device/name/example-camera
>
> # Delete profile, if modified:
> curl -X DELETE http://localhost:59881/api/v2/deviceprofile/name/USB-Camera-General
>
> # Restart:
> sudo snap restart edgex-device-usb-camera
> ```

4. Query core data
5. Show Grafana (pre-setup dashboard)

6. UC Model assertion for the above and a config provider
     - Show config provider for device-mqtt
     - Show model assertion
     - Build image

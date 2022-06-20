## Overview

The demo involves setting up EdgeX to collect ambient sensing measurements from an MQTT broker.

1. Install the platform
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

We assume that sensors have been setup and the measurements for temperature and humidity are published to an MQTT broker at an interval. Setting up sensors and publishing their data to the broker is beyond the scope of this demo. An example for reading data DHT11/DHT22 sensors on a Raspberry Pi is available [here](https://github.com/farshidtz/dht-mqtt).


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

Install [EdgeX Device MQTT](https://snapcraft.io/edgex-device-mqtt) service:
```bash
sudo snap install edgex-device-mqtt
```

The service is NOT started by default. We need to configure and then start it.



4. Query core data
5. Show Grafana (pre-setup dashboard)

6. UC Model assertion for the above and a config provider
     - Show config provider for device-mqtt
     - Show model assertion
     - Build image

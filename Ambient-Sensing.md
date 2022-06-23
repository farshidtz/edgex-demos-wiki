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
- [Mosquitto](https://snapcraft.io/mosquitto) MQTT broker on `jupiter.local`
- [DHT-MQTT](https://github.com/farshidtz/dht-mqtt) for reading data DHT11/DHT22 sensors on `pluto.local` and publishing to the broker. The configuration is as follows:
```bash
MQTT_BROKER=jupiter.local
MQTT_BROKER_PORT=1883
MQTT_TOPIC_PREFIX=pi/pluto/pluto-dht22
MQTT_CLIENT_ID_PREFIX=pluto
PIN=4
SENSOR=DHT22
```

Let's subscribe to the broker to verify the flow of sensing data:
```bash
mosquitto_sub -h jupiter.local -t "#" -v
```
```
pi/pluto/pluto-dht22/temperature 25.3
pi/pluto/pluto-dht22/humidity 40.4
pi/pluto/pluto-dht22/temperature 25.3
pi/pluto/pluto-dht22/humidity 40.6
...
```

The topics have `../<device>/<resource>` format; we only care about the last two parts to map measurements to the EdgeX device and resource setup in the next step. The payloads are the raw measurements, without any envelop object.

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
  Name = "pluto-dht22"
  ProfileName = "temperature-humidity-sensor"
  [DeviceList.Protocols]
    [DeviceList.Protocols.mqtt]
       CommandTopic = "CommandTopic"
```

Update `configuration.toml` with snap option:
```bash
sudo snap set edgex-device-mqtt app-options=true
sudo snap set edgex-device-mqtt config.mqttbrokerinfo-usetopiclevels=true
sudo snap set edgex-device-mqtt config.mqttbrokerinfo-incomingtopic=pi/pluto/#
```

This is equivalent to updating the following entries in the config file before service has started for the first time:
```toml
UseTopicLevels = true
IncomingTopic = "pi/pluto/#"
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
> http DELETE http://localhost:59881/api/v2/device/name/pluto-dht22
>
> # Delete profile, if modified:
> http DELETE http://localhost:59881/api/v2/deviceprofile/name/temperature-humidity-sensor
>
> # Restart:
> sudo snap restart edgex-device-mqtt
> ```

## Querying stored data from EdgeX
Let's query EdgeX Core Data to check if measurements (readings) are being added via Device MQTT. We use the `readings` endpoint and query just 2 records:
```bash
http http://localhost:59880/api/v2/reading/device/name/pluto-dht22?limit=2 --body
```
```json
{
    "apiVersion": "v2",
    "readings": [
        {
            "deviceName": "pluto-dht22",
            "id": "94e2fbf8-4da5-403b-97a0-6b9cd7fab00e",
            "origin": 1655806229236796266,
            "profileName": "temperature-humidity-sensor",
            "resourceName": "humidity",
            "value": "4.120000e+01",
            "valueType": "Float32"
        },
        {
            "deviceName": "pluto-dht22",
            "id": "b5588876-cef9-4fde-bf29-87ab0a92f187",
            "origin": 1655806229234657153,
            "profileName": "temperature-humidity-sensor",
            "resourceName": "temperature",
            "value": "2.560000e+01",
            "valueType": "Float32"
        }
    ],
    "statusCode": 200,
    "totalCount": 16
}
```

## Visualizing sensor data with Grafana
We use Grafana to query the readings from EdgeX Core Data. We use a Grafana plugin called [JSON API](https://grafana.com/grafana/plugins/marcusolsson-json-datasource/)to query and pick the needed information.

#### Install
```bash
sudo snap install grafana --channel=rock/edge
sudo snap start --enable grafana
```
Open UI: http://jupiter.local:3000

Default username/password: admin/admin

#### Install JSON API Plugin
Install the [JSON API](http://localhost:3000/plugins/marcusolsson-json-datasource?page=overview) plugin via "configuration->plugin":
[http://localhost:3000/plugins/marcusolsson-json-datasource?page=overview](http://localhost:3000/plugins/marcusolsson-json-datasource?page=overview))

#### Add a datasource
Select JSON API and set the following parameters:
* name: core-data  
* URL: http://localhost:59880/api/v2/reading

Save and test. You should see Not Found as follows, meaning that the server was set correctly but there is no resource at the given URL. To resolve this, we will later on set the HTTP path in the query editor.


#### Create a dashboard
To do so, go follow: + -> Create / Dashboard

Set the query refresh rate to `5s`

> ℹ **Tip**  
> The range can be shorted by manually entering the from value such as: now-1m

#### Setup the panel
a. Add an empty panel. Set the title to Temperature.

b. Setup query and transformation:
-   Name `Pluto`
-   Field `$.readings[:].value`, Type `Boolean`, Alias `Value`
-   Field `$.readings[:].origin`, Type `String`, Alias `Time(ns)`
-   Path: `/device/name/pluto-dht22/resourceName/temperature` (this gets appended to the server URL set in datasource configuration to construct the core-data reading endpoint as `http://localhost:59880/api/v2/reading/device/name/pluto-dht22/resourceName/temperature`)
-   Param key `limit`, value `100` (this is the number or readings queried from core-data)
-   Cache time: `0s` (otherwise, the datasource won’t query the core-data on refresh!)

At this point, we should be able to see data on a table.

To visualize time series as line graph, we need to add two transformation:

c. In the Transform tab in the query editor:
-   Select "Add field from calculation":
-   Binary operation, `Time(ns)`/`1000000`, Alias `Time`. This converts the time in nanos to seconds
-   Add transformation -> Select "Convert field type"
-   `Time` as `Time`. This converts the time from Number to Time format.


Auto refresh doesn’t work in the query editor, but only on the dashboard. Refresh manually here to see new results.

Save and go back to the dashboard view. It should auto refresh every 5s as previously configured.

Setup a panel for humidity by cloning the Temperature panel and changing the relevant fields.

## Creating an OS image with all the above
Well, except for the sensing part.

To add:
- UC Model assertion for the above and a config provider
  - Show config provider for device-mqtt
  - Show model assertion
  - Build image

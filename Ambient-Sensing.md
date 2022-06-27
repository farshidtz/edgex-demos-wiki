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
Now, let's create an OS image that includes all the above (except for the sensing part) and can be easily flashed on storage medium to create a bootable drive. This will make it very easy to onboard gateway devices pre-loaded with the EdgeX stack!

### Create a config provider for device-mqtt
The EdgeX Device MQTT service cannot be fully configured using environment variables / snap options. Because of that, we need to package the modified config files and replace the defaults.

Since we want to create an OS image pre-loaded with the configured system, we need to make sure the configurations are there without any manual user interaction. We do that by creating a snap which provides the configuration files prepared in the previous steps to the Device MQTT snap:
- configuration.toml
- devices/dht22.toml
- profiles/temperature-humidity-sensor.yml

This snap should be build and uploaded to the store. We use `edgex-demo-ambient-sensing-config` as the snap name. 

The source code is available at: https://github.com/canonical/edgex-demos/tree/main/ambient-sensing/config-provider

Build and upload (release to `latest/edge` channel):
```
snapcraft
snapcraft upload --release=latest/edge ./edgex-demo-ambient-sensing-config_demo_amd64.snap
```

### Create an Ubuntu Core model assertion
Refer to the following article for details on how to sign the model assertion:
https://ubuntu.com/core/docs/custom-images#heading--signing

1. Create and register a key if you don't already have one:
```bash
snap login
snap keys
# continue if you have no existing keys
# you’ll be asked to set a passphrase which is needed before signing
snap create-key edgex-demo
snapcraft register-key
```


2. Now, create the model assertion. 
Get familiar with model assertion: https://ubuntu.com/core/docs/reference/assertions/model

Unlike the official documentation which uses JSON, we use YAML serialization for the model. This is for consistency with all the other serialization formats in this tutorial. Moreover, it allows us to comment out some parts or add comments to describe the details inline.

Create `model.yaml` with the following content:
```yaml
type: model
series: '16'

# authority-id and brand-id must be set to your developer-id
authority-id: SZ4OfFv8DVM9om64iYrgojDLgbzI0eiL
brand-id: SZ4OfFv8DVM9om64iYrgojDLgbzI0eiL

model: ubuntu-core-20-amd64
architecture: amd64

# timestamp should be within your signature’s validity period
timestamp: '2022-06-21T10:45:00+00:00'
base: core20

# grade is set to dangerous because the gadget is not signed nor from the store
grade: dangerous

snaps:
- # This is our custom, dev gadget snap
  # It has no channel and id, because it isn't in the store.
  # We’re going to build it locally and pass it to the image builder. 
  name: pc
  type: gadget
  # default-channel: 20/stable
  # id: UqFziVZDHLSyO3TqSWgNBoAdHbLI4dAH

- name: pc-kernel
  type: kernel
  default-channel: 20/stable
  id: pYVQrBcKmBa0mZ4CCN7ExT6jH8rY1hza

- name: edgexfoundry
  type: app
  default-channel: latest/stable
  id: AZGf0KNnh8aqdkbGATNuRuxnt1GNRKkV

- name: edgex-device-mqtt
  type: app
  default-channel: latest/stable
  id: AeVDP4oaKGCL9fT0u7lbNKxupwXrGiMX

# Config provider for edgex-device-mqtt
- name: edgex-demo-ambient-sensing-config
  type: app
  default-channel: latest/edge
  id: 5riI41SdX1gJYFdFXC5eoKzzBUEzSgqq

# MQTT Broker
- name: mosquitto
  type : app
  default-channel: latest/stable
  id: mDxT0cGOHKSs62MOHSK5Ype0Na5UU2LB
```

> ℹ **Tip**  
> Find your developer ID:
> ```
> $ snapcraft whoami
> email:        farshid.tavakolizadeh@canonical.com
> developer-id: SZ4OfFv8DVM9om64iYrgojDLgbzI0eiL
> 
> Or from https://dashboard.snapcraft.io.
> ```

3. Sign the model
We sign the model using the “edgex-demo” key created and registered earlier. 

The snap sign command takes JSON as input and produces YAML as output! We use yq to convert our model assertion to JSON before passing it in for signing.

```bash
# if you don’t already have it
sudo snap install yq

# sign
yq eval model.yaml -o=json | snap sign -k edgex-demo > model.signed.yaml

# check
cat model.signed.yaml
```

Note: You need to repeat the signing every time you change the input model, because the signature is applied to a copy of the model.

Refer to the one of following to do so:
- Ubuntu Startup Disk Creator: https://ubuntu.com/tutorials/create-a-usb-stick-on-ubuntu
- Raspberry Pi Imager: https://www.raspberrypi.com/software/
- `dd` command: https://ubuntu.com/download/iot/installation-media

### Setup defaults using a Gadget snap
Setting up default snap options and connection is possible via a Gadget snap.

Use one of the following as basis:
- https://github.com/snapcore/pc-amd64-gadget for amd64 computers. This is what we'll use for the remaining steps.
- https://github.com/snapcore/pi-gadget for Raspberry Pi

Add the following to `gadget.yml`:
```yml
# Add default config options
defaults:
  # edgex-device-mqtt
  AeVDP4oaKGCL9fT0u7lbNKxupwXrGiMX:
    # automatically start the service
    autostart: true

connections:
   -  # Connect edgex-device-mqtt's plug (consumer) to 
      #   edgex-demo-ambient-sensing-config's slot (provider) to override the
      #   default configuration files by bind-mounting provider's "res" directory
      #   on top of the provider's "res" directory.
      plug: AeVDP4oaKGCL9fT0u7lbNKxupwXrGiMX:device-config
      slot: 5riI41SdX1gJYFdFXC5eoKzzBUEzSgqq:device-config
```

Build:
```bash
$ snapcraft
...
Snapped pc_20-0.4_amd64.snap
```

Note: You need to rebuild the snap every time you change the gadget.yaml file.

### Build the Ubuntu Core image
We use ubuntu-image and set the following:
- Path to signed model assertion YAML file
- Path to gadget snap that we built in the previous steps

```bash
# install if you don’t already have it
$ sudo snap install ubuntu-image --beta --classic

# build the image
$ ubuntu-image snap edgex-model-signed.yaml --validation=enforce --snap pc_20-0.4_amd64.snap
Fetching pc-kernel
Fetching core20
Fetching edgexfoundry
...
WARNING: "pc" installed from local snaps disconnected from a store cannot be refreshed subsequently!
Copying "pc_20-0.4_amd64.snap" (pc)

# check the image file
$ file pc.img
pc.img: DOS/MBR boot sector, extended partition table (last)
```

**The image file is now ready to be flashed on a medium to create a bootable drive with the needed applications and configuration!**

### Run in an emulator
x86 emulator - for amd64 gadget and image
- https://ubuntu.com/core/docs/testing-with-qemu
- https://ubuntu.com/core/docs/image-building#heading--testing

Run the following command and wait for the boot to complete:
```bash
sudo qemu-system-x86_64 -smp 4 -m 4096  -net nic,model=virtio -net user,hostfwd=tcp::8022-:22  -drive file=/usr/share/OVMF/OVMF_CODE.fd,if=pflash,format=raw,unit=0,readonly=on  -drive file=pc.img,cache=none,format=raw,id=disk1,if=none  -device virtio-blk-pci,drive=disk1,bootindex=1 -machine accel=kvm -serial mon:stdio -vga virtio
```
The above command forwards the port 22 of the emulator to 8022 on the host. Refer to the above references on how to SSH.

To forward additional ports, add more: hostfwd=tcp::PORT_ON_HOST-:PORT_IN_EMULATOR values to -net.

Once the boot is complete, it will prompt for your email address to deploy your public key. This manual step can be avoided by pre-loading the image with user data.

Connect to it via SSH
```bash
ssh <user>@localhost -p 8022
```


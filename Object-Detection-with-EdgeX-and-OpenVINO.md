## Architecture

![overview.drawio.svg](https://raw.githubusercontent.com/canonical/edgex-demos/main/openvino-object-detection/figures/overview.drawio.svg)

## Steps
### 1. (EdgeX) Install platform
```bash
sudo snap install edgexfoundry --channel=latest/stable
```

**[tip]** [Extend the default secret store tokens TTL](https://docs.edgexfoundry.org/2.2/getting-started/Ch-GettingStartedSnapUsers/#secret-store-token) from 1h to 72h to avoid running into expired tokens while preparing the demo.
Note that tokens will expire if some components are stopped for a period longer than the validity. The restart command in the given instructions can be used to issue a fresh set of tokens.

### 2. (EdgeX) Setup Device USB Camera:
install:
```bash
sudo snap install edgex-device-usb-camera --channel=latest/edge/pr-30
connect edgex-device-usb-camera’s edgex-secretstore-token and camera interfaces:
sudo snap connect edgexfoundry:edgex-secretstore-token edgex-device-usb-camera:edgex-secretstore-token
sudo snap connect edgex-device-usb-camera:camera :camera
```
configure and start:
```bash
sudo mv /var/snap/edgex-device-usb-camera/current/config/device-usb-camera/res/devices/general.usb.camera.toml.example \
/var/snap/edgex-device-usb-camera/current/config/device-usb-camera/res/devices/general.usb.camera.toml
```
**[optional]** 
set the right video device (default is /dev/video0)
we assume that the device name stays as default “example-camera” in the rest of this document:
```bash
sudo nano /var/snap/edgex-device-usb-camera/current/config/device-usb-camera/res/devices/general.usb.camera.toml
```
```bash
sudo snap start --enable edgex-device-usb-camera
```
trigger streaming for usb camera:
```bash
curl -X PUT -d '{
    "StartStreaming": {
      "OutputFps": "5",
      "InputImageSize": "320x240",
      "OutputVideoQuality": "31"
    }
}' http://localhost:59882/api/v2/device/name/example-camera/StartStreaming

```
**[debug]** The usb camera could be stopped by:
```bash
curl -X PUT -d '{"StopStreaming": true
}' http://localhost:59882/api/v2/device/name/example-camera/StopStreaming
```
Note that stopping the stream will cause openvino’s container to exit! The container will automatically restart if there is a restart policy, but that may take up to a minute.

**[debug]** Check the video stream:
Test URI with VLC. You would see a video window:
if you don’t already have it:
```bash
sudo snap install vlc
vlc rtsp://localhost:8554/stream/example-camera
```
If that didn’’t work, use mplayer:
```bash
mplayer rtsp://localhost:8554/stream/example-camera
```
**[tip]** Need to change the device/device profile after service has started? Update the local files, delete from core-metadata, and restart:

delete device:
```bash
curl -X DELETE http://localhost:59881/api/v2/device/name/example-camera
```
delete profile, if modified:
```bash
curl -X DELETE http://localhost:59881/api/v2/deviceprofile/name/USB-Camera-General
```
restart:
```bash
sudo snap restart edgex-device-usb-camera
```
query the above URLs to make sure the changes have been reflected.

**[tip]** Turn on device-usb-camera’s auto streaming:
```bash
sudo snap set edgex-device-usb-camera app-options=true
sudo snap set edgex-device-usb-camera config.devicelist-protocols-usb-autostreaming=true
sudo snap restart edgex-device-usb-camera.device-usb-camera
```
### 3. (Mosquitto) Setup MQTT Broker
Install the mosquitto broker, or any other MQTT broker. We use port 1883 for MQTT (without TLS).
```bash
sudo snap install mosquitto
```
The broker is started automatically, but just in case you have disabled it:
```bash
sudo snap start --enable mosquitto
```
### 4. (OpenVINO) Setup OpenVINO
Install docker, if you don’t already have it:
```bash
sudo snap install docker
```
Run the object detection container:

The DISPLAY environment variable is optional for viewing the predictions as annotations on the video stream. It should open a new window on the host. It may not work and can be omitted safely.

Let’s try by starting a temporary container in the foreground:
```bash
sudo docker run --network=host -e DISPLAY=$DISPLAY --rm ghcr.io/monicaisher/edgex-openvino-object-detection:latest
```
Ctrl+C or closing the real time video window will stop openvino.

If everything worked without error, stop and run again in background (detached), with a restart policy, name, and without the display env variable:
```bash
sudo docker run --network=host --name=openvino --restart=unless-stopped --detach ghcr.io/monicaisher/edgex-openvino-object-detection:latest
```
To stop when detached:
```bash
sudo docker stop openvino
```
**[debug]** Subscribe to the broker and see the predictions:
```bash
mosquitto_sub -t "openvino/MQTT-test-device/prediction"
```
```bash
{"objects":[{"detection":{"bounding_box":{"x_max":1.0,"x_min":0.1194220781326294,"y_max":0.9418730139732361,"y_min":0.06846112012863159},"confidence":0.6409656405448914,"label":"dog","label_id":11},"h":210,"roi_type":"dog","w":282,"x":38,"y":16}],"resolution":{"height":240,"width":320},"timestamp":9031881447638}
...
```
**[debug]** Query core-data to check if raw predictions are being added via Device MQTT:
```bash
curl http://localhost:59880/api/v2/reading/device/name/MQTT-test-device
```
### 5. (EdgeX) Setup Device MQTT

a) Install the device service:
```bash
sudo snap install edgex-device-mqtt
```
b) update configuration.toml with snap option:
```bash
sudo snap set edgex-device-mqtt app-options=true
sudo snap set edgex-device-mqtt config.mqttbrokerinfo-usetopiclevels=true
sudo snap set edgex-device-mqtt config.mqttbrokerinfo-incomingtopic=openvino/#
```
c) replace the whole mqtt.test.device.profile.toml:
```bash
sudo nano /var/snap/edgex-device-mqtt/current/config/device-mqtt/res/profiles/mqtt.test.device.profile.yml
```
  
```
name: "Test-Device-MQTT-Profile"
manufacturer: "Canonical"
model: "MQTT-2"
labels:
- "openvino"
description: "device mqtt profile for openvino prediction"
deviceResources:
-
name: prediction
isHidden: false
description: "prediction in JSON"
properties:
valueType: "Object"
readWrite: "R"
mediaType: "application/json"
```
d) start device-mqtt:
```bash
sudo snap start --enable edgex-device-mqtt
```
**[debug]** Verify that the right profile has been uploaded:
```bash
curl http://localhost:59881/api/v2/deviceprofile/name/Test-Device-MQTT-Profile
```
**[debug]** Check the logs to see if there are errors:
```bash
sudo snap logs -f edgex-device-mqtt
```
to see if all messages pass through, enable the debugging first:
```bash
sudo snap set edgex-device-mqtt config.writable-loglevel=DEBUG
sudo snap restart edgex-device-mqtt
```
**[tip]** Need to change the device/device profile after service has started? Update the local files, delete from core-metadata, and restart:

delete the device:
```bash
curl -X DELETE http://localhost:59881/api/v2/device/name/MQTT-test-device
```
delete the profile, if modified:
```bash
curl -X DELETE http://localhost:59881/api/v2/deviceprofile/name/Test-Device-MQTT-Profile
```
restart:
```bash
sudo snap restart edgex-device-mqtt
```
query the above URLs to make sure the changes have been reflected.
### 6. (EdgeX) Setup eKuiper
eKuiper filters prediction results and sends them back to edgex message bus.
Install eKuiper:
```bash
sudo snap install edgex-ekuiper
```
Configure eKuiper:
```bash
sudo nano /var/snap/edgex-ekuiper/current/etc/sources/edgex.yaml
```

In the default section, change as below:
~~topic: rules-events~~
topic: edgex/events/#
messageType: request
```bash
sudo snap restart edgex-ekuiper
```
create a stream:
```
edgex-ekuiper.kuiper-cli create stream deviceMqttStream '() WITH (FORMAT="JSON",TYPE="edgex")'
```
create a rule:
```
edgex-ekuiper.kuiper-cli create rule filterPeople '
{
  "sql":"SELECT regexp_matches(prediction, \"person\") AS person FROM deviceMqttStream WHERE meta(deviceName)=\"MQTT-test-device\"",
 "actions": [
     {
       "log":{}
     },
    {
      "edgex": {
       "connectionSelector": "edgex.redisMsgBus",
        "topicPrefix": "edgex/events/device",
        "messageType": "request",
        "deviceName": "people",
        "contentType": "application/json"
      }
    }
  ]
}'

```
**[debug]** Query results from EdgeX Core Data:
```bash
curl http://localhost:59880/api/v2/reading/device/name/people
```
### 7. (Grafana) Visualize OpenVINO predictions
We use Grafana to query the filtered results from EdgeX Core Data. We use a Grafana plugin called JSON API ([https://grafana.com/grafana/plugins/marcusolsson-json-datasource/](https://grafana.com/grafana/plugins/marcusolsson-json-datasource/)) to query and pick the needed information.
install:
```bash
sudo snap install grafana --channel=rock/edge
sudo snap start --enable grafana
```
TBD
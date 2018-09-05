# Full IoT product: smart light on Mongoose OS

This repository contains the implementation of the full, functional commercial IoT product under a commercial-friendly Apache 2.0 license.
It utilises the power of [Mongoose OS](https://mongoose-os.com) and can be used as a reference for creating similar smart products.

This project implements a smart light. For the hardware, we use a development board with an LED, which serves as a light. The devboard can be
"shipped" to a customer. A customer provisions it using a mobile app.
You, as a vendor, have full control
over all "shipped" products, including device
dashboard with remote firmware updates, remote management and usage statistics.

This short video demonstrates the use case:

TBD

## Step-by-step usage guide

1. Get a hardware device. We simulate a real smart lite with one of the
   supported development boards - choose one from https://mongoose-os.com/docs/quickstart/devboards.md. The built-in LED
   on the devboard will act as a light. Alternatively, you can put together
   your own hardware setup, just make sure to alter `firmware/mos.yml` to set
   the GPIO pin number for the LED.
2. Follow https://mongoose-os.com/software.html to
   install `mos`, a Mongoose OS command-line tool.
3. Clone this repository:
   ```
   git clone https://github.com/cesanta/mongoose-os-smart-light
   ```
5. Install [Docker Compose](https://docs.docker.com/compose/) and
   start the backend on your workstation (or any other machine):
   ```
   cd backend
   docker-compose build
   docker-compose up
   ```
   NOTE: on MacOS, make sure to use Docker for Mac (not Docker toolbox),
   see https://docs.docker.com/docker-for-mac/docker-toolbox/. That is
   required cause Docker toolbox installation on Mac requires extra steps
   to forward opened ports.
6. Connect your device to your workstation via a USB cable. Build and
   flash the device:
   ```
   cd mongoose-os-smart-light/firmware
   mos build --platform YOUR_PLATFORM  # esp32, cc3220, stm32, esp8266
   mos flash
   ```
8. Register a new device on a management dashboard, obtain access token:
   ```
   $ curl -d '{}' -u admin:admin http://YOUR_WORKSTATION_IP:8009/api/v2/devices
   {
     ...
     "id": "...........",
     "token": "..........",
     ...
   }
   ```
   If you login to the dash at http://YOUR_WORKSTATION_IP:8009 with
   username/password `admin/admin`, you should be able to see your new device.
9. Factory-configure your device, and pre-provision it on a dashboard:
   ```
   mos config-set --no-reboot device.id=GENERATED_DEVICE_ID
   mos config-set --no-reboot dash.token=ACCESS_TOKEN
   mos config-set --no-reboot dash.server=ws://YOUR_WORKSTATION_IP:8009/api/v2/rpc
   mos config-set --no-reboot conf_acl=wifi.*,device.*,dash.enable
   mos call FS.Rename '{"src": "conf9.json", "dst": "conf5.json"}'
   ```
   The `mos config-set` commands generates `conf9.json` file on a device.
   The `mos call FS.Rename` renames it to `conf5.json`, in order to make this
   configuration immune to factory reset and OTA. The only way to re-configure
   these settings is to reflash the device, or remove `conf5.json`.


## General Architecture

The backend is installed on your workstation (so called on-premises
installation). It is completely self-contained, not requiring any external
service to run, and run as a collection of Docker images (docker-compose).
Thus, such backend could be run on any server, e.g. as a AWS EC2 instance,
Google Cloud instance, etc.


Device management backend is mDash (the same that runs on
https://dash.mongoose-os.com), the frontend is a PWA (progressive web app).
Both are behind Nginx, which terminates SSL from devices and mobile apps.

<img src="media/a1.png" class="mw-100" />

The mobile app talks with the API server over WebSocket, sending and
receiving JSON events. Switching the light on/off sends
`{"name:"on", "data":{"id":.., "on": true/false}}` event. An API server catches it, and modifies the device shadow
object for the device with corresponding ID: `{"desired": {"on": true/false}}`.
The device shadow generates a delta, which is sent to a device. A device code
reacts to the delta, switches the light on or off, and updates the shadow.
Shadow update clears the delta, and triggers a notification from mDash.
API server catches the notification, and forwards it to the mobile app. 
A mobile app reacts, and sets the on/off control according to the device shadow.

That implements a canonic pattern for using a device shadow - the same logic
can be used with backends like AWS IoT device shadow,
Microsoft Azure device twin, etc. 


## Backend

The mDash comes pre-configured with a single administrator user `admin`
(password `admin`). That was done with the following command:

```
docker-compose run dash /dash --config-file /data/dash_config.json --register-user admin admin
```

The resulting `backend/data/db.json` mDash database was committed to
the repo. The API key, automatically created for the admin user, is used
by the API Server for all API Server <-> mDash communication, and specified
as the `--token` flag in the `backend/docker-compose.yml` file. Thus,
the API Server talks to the mDash with the administrative privileges.

## Device provisioning process

Adding new device is implemented by the Mobile app (PWA) in 3 steps:

1. Customer is asked to join the WiFi network called `Mongoose-OS-Smart-Light`
   and set device name. A new device, when shipped to the customer,
   starts a WiFi access point, and has a pre-defined IP address `192.168.4.1`.
   The app calls device's RPC function `Config.Set`, saving entered
   device name into the `device.password` configuration variable.
2. Customer is asked to enter WiFi name/password. 
   The app calls device's RPC function `Config.Set` to set
   `wifi.sta.{ssid,pass,enable}` configuration variables, and then calls
   `Config.Save` function to save the config and reboot the device.
   After the reboot, a device joins home WiFi network, and starts the
   DNS-SD service, making itself visible as `mongoose-os-smart-light.local`.
3. Customer is asked to join home WiFi network and press the button to
   finish registration process. The app calls `Config.Set` and `Config.Save`
   RPCs to disable local webserver on a device, and the DNS-SD service.
   Then it sends `pair` Websocket message to the API server, asking to
   associate the device with the particular mobile APP (via the generated app ID).
   The API server registers the app ID as a user on mDash,
   and sets the `shared_with` device attribute equal to the app ID.

Thus, all devices are owned by the admin user, but the pairing process
shares a device with the particular mobile app. Therefore, when an API
server lists devices on behalf of the mobile app, all shared devices are
returned back.


## Mobile app

The mobile app is a Progressive Web App (PWA). It is written in
[preact](https://preactjs.com/) and [bootstrap](https://getbootstrap.com/).
The main app logic is in a signle source file, `backend/mobile-app/js/app.jsx`.
In order to avoid a separate build step, the app uses a prebuilt babel
transpiler.

When first downloaded and run on a mobile phone or desktop browser,
an app generates a unique ID and sets an `app_id` cookie. The `app_id`
cookie is used to authenticate the mobile phone with the
API server. The API server creates a user on the mDash for that `app_id`.
Basically, an API server trusts each new connection with a new `app_id`
that it is a new mobile app client, and creates a user for it. This simple
authentication schema allows to avoid user login/password step, but
is also suboptimal, cause it binds a user to a specific device. If,
for some reason, cookies get cleared, then all devices must be re-paired.

That was done deliberately to skip the user login step, as it is not
crucial for this reference implementation. Those who want to implement
password based user auth, can easily do so, for it is well known and understood.

When started, the app creates a WebSocket connection to the API Server, and
all communication is performed as an exchange of WebSocket messages. Each
message is an "event", which is a single JSON object with two attributes:
`name` and `data`. The API Server receives events, and may send events in
return. There is no request/response pattern, however. The communication
is "fire and forget" events.

The events sent by the app are:

- `{"name": "list"}` - request to send device list
- `{"name": "pair", "data":{"id":...}}` - request to pair a device with the app


## Mongoose OS - based firmware

## Device dashboard

## Usage statistics and analytics

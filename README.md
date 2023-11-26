# Bluetooth Low Energy (BLE) Communication with BRIO Smart Tech Sound train

```
Notes on BLE communication with BRIO Smart Tech Sound train
Author: Chris Petrich, 26 Nov 2023.
License: CC-BY, https://creativecommons.org/licenses/by/4.0/
This repository is not affiliated with BRIO.
```

## About

This document is about BRIO Smart Tech Sound as sold around 2023.
The trains can be controlled

- via Bluetooth Low Energy (BLE) with the BRIO Smart Tech Sound App,
- 13.56 MHz RFID Action Tunnels such as those found in sets 33974 and 33972, or
- three buttons on the train (forward, stop, reverse).

## Results

A valid control event results in a sequence of one to ca. five BLE
notifications in conjunction with a sequence of sound, light, and
motion effects. These sequences are part of the firmware of the train,
which can be updated with the app.

The trains are BLE peripherals (i.e. servers) that advertise themselves
as `Smart 2.0`.
The service UUID is `b11b0001-bf9b-4a20-ba07-9218fec577d7`,
and within this, the characteristic for commands to the train is
`b11b0002-bf9b-4a20-ba07-9218fec577d7`, while
the characteristic for notifications from the train back to the BLE
client is `b11b0003-bf9b-4a20-ba07-9218fec577d7`.

BLE commands and notifications are a sequence of unsigned 8-bit integers
framed by start byte `0xAA` and single-byte checksum, `<C>`, at the end. The
checksum is calculated from the sum of the bytes of the command or notification
(i.e. without the start byte `0xAA`) as follows:

`<C> = (0x100 - (Sum & 0xFF)) & 0xFF`

Unknown commands or commands with checksum errors are silently ignored (i.e.
no notifications are sent).

Among the list of recognized commands is

| Command             | Meaning |
| ------------------- | ------- |
| `02:01:<speed>`     | Set speed of train and play appropriate light and sound effects. |
| `03:56:AA:<effect>` | Stop train, set sound effect theme, and play sound. |
| `03:52:<b1>:<b2>`   | Unknown. |

Train speed, `<speed>`, is coded as 

- `0x00` for "off", 
- values `0x01` to `0x08` for forward, and
- values `0x11` to `0x18` for backward.

Currently, power with values `0x01`and `0x11` is too low to actually move the train.

The sound effect themes are coded depending on context:

| Effect    | `<effect>` | `<eff1>` | `<eff2>` | `<eff3>` | `<eff4>` |
| --------- | ---------- | -------- | -------- | -------- | -------- |
| Honk      | `0xf0`  | `0x15` | `0x1d` | `0x11` | `0x19` |
| Whistle   | `0xf1`  | `0x16` | `0x1e` | `0x12` | `0x1a` |
| Horn      | `0xf2`  | `0x17` | `0x1f` | `0x13` | `0x1b` |
| Spaceship | `0xf3`  | `0x18` | `0x20` | `0x14` | `0x1c` |

The **first** notification message after pressing a button on
the train is:

| Button   | First Notification |
| -------- | ------------------ |
| Stop     | `02:80:01`   |
| Forward  | `02:80:02`   |
| Backward | `02:80:03`   |

The **first** notification when passing through an Action Tunnel is:

| Action Tunnel               | First Notification |
| --------------------------- | ------------------ |
| "stop" from forward         | `08:81:02:00:00:<eff1>:91:3a:03` |
| "stop" from backward        | `08:81:02:00:00:ef:0f:00:0f` |
| "change direction"          | `08:81:03:00:00:ef:0f:00:0f` |
| "break" down from forward   | `08:81:07:40:00:<eff2>:6f:31:20` |
| "break" down from backward  | `08:81:07:40:00:<eff2>:6f:31:30` |
| "fix" without repair        | `08:81:0c:00:00:<eff3>:f9:9a:<speed>` |
| "fix" with repair, forward  | `08:81:0c:00:00:ef:0f:00:03` |
| "fix" with repair, backward | `08:81:0c:00:00:ef:0f:00:0f` |
| double tunnel, top          | `08:81:25:00:00:ef:df:aa:<speed>` |
| "sound" (orange)            | `08:81:39:00:00:87:01:3a:<speed>` |

Action tunnel notifications are sent only if the train is in motion.

Apparently, notifications gets sent after the actual action has already been
initiated by the train, i.e. it does not seem to be possible to *cleanly*
override the pre-programmed behavior by sending BLE commands (there may be
train jerking and aborted sounds). Sending a command in response to a
notification from an Action Tunnel may cause the train to detect the Action
Tunnel again, leading to loop that lasts until the train is out of range
of the tunnel.

## Examples

### BLE Commands

| Sequence | Meaning |
| -------- | ------- |
| `aa:02:01:08:f5` | Full speed forward |
| `aa:02:01:02:fb` | Slowly forward |
| `aa:02:01:00:fd` | Stop the train |
| `aa:02:01:18:e5` | Full speed backward |
| `aa:03:56:aa:f0:0d` | Set sound theme to "honk" (default) |
| `aa:03:56:aa:f1:0c` | Set sound theme to "whistle" |

In JavaScript, commands could be sent to the BLE peripheral like so:
`characteristic.writeValue( new Uint8Array([0xaa,0x02,0x01,0x08,0xf5]) )`.

It follows a minimal code example to make the train move forward
using the Web Bluetooth API.
**Read the Disclaimer at the end of this page before proceeding.**
Copy text to a file with extension `.html`
and open in Chrome or Edge.
Turn on the *BRIO Smart Tech Sound* train, enable BLE on the computer
and click the button on the web page to initiate the pairing process.
Note that **this will work only on Chrome and Edge**.
```html
<!DOCTYPE html>
<html>
<head>
<title>BLE test</title>
<meta name="author" content="Chris Petrich" />
</head>
<body>
<h1>BRIO Smart Tech Sound example for Chrome and Edge</h1>
<button onclick="connectBLE()">Click to Connect to Train and Start Driving</button>
<script>
const serviceUUID = "b11b0001-bf9b-4a20-ba07-9218fec577d7";
const commandCharacteristicUUID = "b11b0002-bf9b-4a20-ba07-9218fec577d7";
function connectBLE() {
  navigator.bluetooth.requestDevice(
    { filters: [{ services: [serviceUUID] }] })
    .then(device => device.gatt.connect())
    .then(server => server.getPrimaryService(serviceUUID))
    .then(service => service.getCharacteristic(commandCharacteristicUUID))
    .then(characteristic => characteristic.writeValue( new Uint8Array([0xaa,0x02,0x01,0x08,0xf5]) ));
}
</script>
</body>
</html>
```

### BLE Notifications

Example notification sequence of a train going backward through the
"fix" tunnel while still being damaged from the "broken" tunnel.
Initial speed `0x18`, and sound effect `0xf0` (default).

```
aa:08:81:0c:00:00:ef:0f:00:0f:5e
aa:08:81:0c:00:01:ef:0f:00:03:69
aa:08:81:0c:00:02:21:c1:91:0f:e7
aa:08:81:0c:00:03:00:3f:3a:18:d7
aa:08:81:00:00:00:00:3f:3a:18:e6
```

## Disclaimer

I am not affiliated with BRIO. This is not an official documentation by BRIO.
Information on this page are the result of some basic reverse engineering
effort on my part, and results may contain errors. Information may be out of
date as implementation and firmware of the trains could change at any time.
Note that your train may get permanently unresponsive if you inadvertently
initiate a firmware update. Use information in this repository
**at your own risk**.

## Sources

- Inspiration for this work came from the blog post *Smart Track vs Smart Tech* by `JohnM` from January 2020 at
https://woodenrailway.info/blog/products/smart-track-vs-smart-tech
- BLE sniffing for UUIDs and commands followed the instructions of [Nordic Semiconductor](https://www.nordicsemi.com/) for the nRF52840 Dongle.
- Notifications were received after connecting to a train with the [Web Bluetooth API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API).

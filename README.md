# Cromocon LRV Meter web app

A single-page replacement for the Cromocon LRV Meter phone app. It talks to the meter over Bluetooth Low Energy from the browser (Web Bluetooth).

- **Android:** open the page in Chrome.
- **iPhone / iPad:** Safari has no Web Bluetooth. Install the free **Bluefy** browser and open the page there.
- **Windows / Mac / ChromeOS:** Chrome or Edge, on a computer with Bluetooth.
- **No meter to hand:** add `?demo` to the address to use a simulated meter.

The page must be served over https (or from `localhost`). It does not work inside a claude.ai artifact, because that frame blocks device APIs.

## The hardware

The Cromocon LRV Meter is a rebadged **Palette Pico** colorimeter. The official Android app is `com.palette.picocromocon` v1.0.8 (June 2022, made by Palette). The micro-USB port is for charging only, so all data goes over Bluetooth.

## Bluetooth protocol

Worked out from the decompiled Android app. The meter advertises with the name `PICO`.

| Purpose | Service | Characteristic | Direction |
|---|---|---|---|
| Colour data | `54AA5561-9FA7-417A-90AA-AF6EAA357506` | `54AA5563-…` | notify: 3 × float32 big-endian (L\*, a\*, b\*), sent when the meter's own button is pressed |
| Charging status | `54AAFAF1-…` | `54AAFAF2-…` | notify: byte 0 = `C0` not charging, `C1` charging, `F1` charged |
| Command channel (Palette) | `54AAB9A1-…` | `54AAB9A2-…` notify (replies), `54AAB9A3-…` write (commands) | |
| Command channel (older meters) | Nordic UART `6E400001-B5A3-F393-E0A9-E50E24DCCA9E` | `6E400003-…` notify, `6E400002-…` write | |

All the custom UUIDs share the suffix `-9FA7-417A-90AA-AF6EAA357506`. Enable notifications on all three notify characteristics (CCCD `2902`) before sending commands.

Commands go to the write characteristic. The reply arrives on the notify characteristic, and its byte 1 echoes the command code. The app sends one command at a time and gives up after 15 s.

| Command | Bytes sent | Reply |
|---|---|---|
| Take a reading | `4C 40` + 12 × `00` | `?? 40` + 3 × float32 BE (L\*, a\*, b\*) from byte 2 |
| Zero (calibrate) | `40 05` | `?? 05` when finished. The meter must be pushed fully into its cap first. |
| Battery level | `41 07 00` | `?? 07 NN`, where NN is 0–100 %. Palette channel only. |
| Charging state | `41 0E 00` | `?? 0E SS`, where SS = `B1` charged, `B2` not charging, `B3` charging. Palette channel only. |

**LRV** is calculated from L\* as the CIE luminance factor: `Y = ((L* + 16) / 116)^3 × 100`. For L\* ≤ 8 this app uses the exact CIE linear segment `L* / 903.3 × 100`, where the original app used the cube formula throughout (the two differ by less than 1 LRV point). A multi-reading LRV is the mean of the individual LRVs.

## Why the official Android app fails on newer phones

It targets Android 12 (API 32) but only checks the Location permission before it starts scanning, and it uses the deprecated `startLeScan` API. On Android 12 and later, scanning also needs the **Nearby devices** permission. If that permission isn't granted, the scan fails or finds nothing. A possible workaround with the old app: in Android Settings → Apps → Cromocon LRV Meter → Permissions, allow both **Nearby devices** and **Location**, and turn Location on.

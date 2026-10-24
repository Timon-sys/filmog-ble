# FilMog Control

A single HTML page that controls a Ulanzi FilMog smoke machine over Bluetooth. You don't need an app or an install, just Chrome or Edge. It reacts faster than the original remote, fast enough for 100 ms puffs.

> **Warning:** the fogger has no watchdog of its own. Once it gets "smoke on", it fogs until it gets "smoke off". The page sends that stop when you let go, when the window loses focus, when you switch tabs, when you close it and when it reconnects, and a watchdog stops any smoke the page didn't ask for. If your PC crashes in the middle of a burst, though, nothing is left to send the stop. Keep the battery within reach.

## Use it

Web Bluetooth only runs in a secure context. On your own PC, `localhost` counts as one, so a local web server is enough. From any other machine, the page has to be served over https.

1. Switch the original remote off. The fogger accepts only one Bluetooth connection at a time.
2. In this folder, run:
   ```
   python -m http.server 8765 --bind 127.0.0.1
   ```
3. Open <http://localhost:8765/filmog.html> in Chrome or Edge. Firefox doesn't support Web Bluetooth.
4. Click Connect and pick `CH9141BLE2U`.

The big button fogs while you hold it, and so does the Space key. Either way it stops after 15 seconds at most. There are fixed 1, 2 and 5 second bursts, plus a field for any length from 50 to 10000 ms. Two sliders (with number fields for exact values) set the coil power and the air pump, and they follow whatever the fogger reports. The status panel shows whether it's smoking, the current power and fan, both batteries as bars, and how old the last status frame is.

By default the page keeps to safe limits: power 8000 to 40000 like the remote, and the fan between 22 and 100. Before any smoke it checks what the fogger has stored and corrects anything outside those limits, and it stops the smoke if the pump isn't running while the coil heats. At the bottom, "Unlock all values" removes those limits (any power, fan 0 to 255, holds and bursts up to 60 s) and turns the pump guard into a warning. It always starts locked.

## How we got here

I reverse-engineered this in one long night together with [Claude Code](https://claude.com/claude-code).

### 1. Is it even BLE?

A scan with only the remote powered found nothing new. That fits a remote that acts as the central: it scans and connects, but never advertises itself. With only the fogger powered, a strong new device showed up, `CH9141BLE2U`.

### 2. A transparent UART bridge

The CH9141 is a WCH chip that bridges Bluetooth to a serial port, and this one runs WCH's stock firmware. It has a service `FFF0` with `FFF1` (notify), `FFF2` (write) and `FFF3` (config), plus a second service `FFE0`. There's no pairing and no encryption. Subscribing produced a 20-byte status frame starting with `aa 55`, about ten times a second. The fogger's microcontroller just talks serial, and the CH9141 passes it through.

### 3. Pretending to be the fogger

To learn the commands, we had to see what the remote sends. Windows can host a GATT server, so the PC offered the same services, and the remote ignored it completely. It only connects to its own fogger's MAC address, and Windows has no way to fake one.

### 4. An ESP32-S3 clone

An ESP32 can set its own MAC address, so it became the fake fogger. The first attempt came up one address off. The S3 derives its Bluetooth address as base + 1, while the classic ESP32 uses base + 2. Once that was fixed, the remote connected and immediately showed a full-screen "oil tank connected" popup, again and again. The status frame Claude was replaying had been captured right after the fogger powered up, and one byte in it is a one-off "tank inserted" event. We were announcing a fresh tank ten times a second.

### 5. Man in the middle

Next came a relay: remote to ESP clone, ESP to PC over USB serial, PC to the real fogger. It works because the real CH9141 stops advertising the moment the PC connects to it, which leaves the clone as the only device on air with that MAC. Two things went wrong at first. On the first subscribe, the CH9141 dumps kilobytes of status it buffered while nobody was listening, and that flooded the remote. And the fogger sends its status on `FFE1`, while the remote only listens on `FFF1`.

### 6. Hard-coded handles

Even with real, live status flowing through, the remote never sent a single command. It turned out the remote doesn't look attributes up by UUID when it writes. It uses the fixed handle numbers of the real CH9141 (the `FFF2` value sits at handle 37). The clone had a different layout, so the remote's writes hit read-only attributes and were rejected without a trace. Once the clone's attribute table matched the real one handle for handle, the remote started talking, and the relay passed its first command through. First smoke!

### 7. The runaway

The fogger then didn't stop, and the battery had to come out, The remote's "smoke on" had made it through the relay, but its "release" got lost when the shaky link to the ESP dropped. That's how we learned there's no watchdog. We had a second close call right after: the safety stop Claude had guessed at would actually have been the start command.

### 8. Decoding with known presses

One more capture, with a known sequence of presses (smoke on and off a few times, then power down, then fan down), lined each command up with a change in the status frame. With the protocol known, the PC could skip the remote and talk to the fogger directly. A Python script ran a 5 second burst and confirmed the stop, and this Web Bluetooth page replaced the script soon after.

## Protocol

The CH9141 exposes the fogger microcontroller's serial port. Write commands to `FFF2` in service `FFF0`. Status arrives as notifications on `FFE1` in service `FFE0`, and sometimes on `FFF1`.

### Commands

Every command starts with `55 aa`, followed by a command byte and its payload.

| Bytes | Meaning |
|---|---|
| `55 aa 00` | smoke on |
| `55 aa 02` | smoke off, the only way to stop it |
| `55 aa 04 <u32 big-endian>` | fog power; the remote goes from 8000 (its lowest setting) to 40000, and we've also seen 15000, 25000, 30000 and 35000 |
| `55 aa 05 <fan %> 64` | air pump; any byte is accepted, the pump tops out at 100, and the remote's steps of 13 are only its own UI |
| `55 aa 07 00 00` | sent by the remote when it connects; purpose unknown, not needed |
| `55 aa 08 <n> 00` | remote keepalive, about every 0.5 s; `n` drifts slowly (the remote's battery?); not needed |

Smoke starts about 130 ms after `00` and stops about 100 ms after `02`. The fogger acknowledges each write on `FFF3` with `88 00 04 00 01 10 11`.

The fogger keeps its power and fan settings when the remote or the PC disconnects. It also keeps fogging when the Bluetooth link drops: in a test, the PC disconnected 3 seconds into a burst, and the smoke ran for 25 seconds until the PC reconnected and sent `02`.

### Status frame

20 bytes, about ten times a second:

```
aa 55 14 | ?? | power (u32 BE) | fan % | pump | tank | ?? | batt L | batt R | 00 00 | fogging | ff ff ff
 0  1  2 |  3 |   4  5  6  7   |   8   |  9 |  10  | 11 |   12   |   13   | 14 15 |   16    | 17 18 19
```

The FilMog is basically a heated coil with an air pump. "Power" is the coil and "fan" is the pump. Byte 8 is the fan setting, and byte 9 is what the pump actually does: it reads `64` (100) while the pump runs and `00` when the fan is set to 0. Fogging with the fan at 0 heats the coil with no airflow, and the wick crackles. Below power 8000 the coil doesn't get hot enough to make any smoke, and power 0 turned the fogger's LED red for the duration of the burst.

Byte 10 is `02` once after power-up and `00` from then on. That's the "oil tank connected" event. Byte 16 is `01` while the fogger is smoking. Bytes 12 and 13 slowly drop with use, so they're almost certainly the two batteries. Their scale isn't confirmed, and the page assumes they're percentages.

### Still unknown

We don't know the battery scale, or what bytes 3 and 11 mean (byte 11 showed `08` while the battery values were rising, so maybe a charging flag). We also don't know what power above 40000 does, or what `07` and the `08` keepalive tell the fogger. If you have a FilMog and figure any of these out, please open an issue.

## Notes

This project isn't affiliated with or endorsed by Ulanzi. It's a hobby project for talking to a fogger you own. There's a heater inside a smoke machine, so use this at your own risk and don't leave it fogging unattended.

Released under the MIT License, see [LICENSE](LICENSE).

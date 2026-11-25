# Bluetooth: phone or tablet streaming into the Phoniebox speakers

Decision: the Phoniebox becomes a **Bluetooth speaker**. A phone or tablet pairs with it and its audio comes out of the HiFiBerry speakers. Headphone output is not planned (notes for that are at the end for reference).

This is a Raspberry Pi OS feature (BlueZ as an A2DP sink), not a Jukebox feature, so it works on version 2.4 or version 3. The audio routing step differs because v3 uses PulseAudio and 2.4 uses bare ALSA.

Hardware: the Pi 4's onboard Bluetooth is enough. Nothing to buy.

## 1. BlueZ: make the Pi a pairable speaker

Edit `/etc/bluetooth/main.conf` and set, under `[General]`:

```ini
Name = Aurora's Phoniebox
Class = 0x200414
DiscoverableTimeout = 120
JustWorksRepairing = always
```

- `Class = 0x200414` advertises the box as a loudspeaker so phones show the speaker icon and offer audio.
- `DiscoverableTimeout = 120` means the box is only visible for two minutes after pairing mode is turned on (section 4). Set `0` for always visible, but see the safety note.
- `JustWorksRepairing = always` lets a phone that forgot the box pair again without an error.

Then:

```bash
sudo systemctl restart bluetooth
```

Pairing without a keyboard needs an agent that auto-accepts. Install `bluez-tools` and run `bt-agent` as a service:

```bash
sudo apt install -y bluez-tools
```

`/etc/systemd/system/bt-agent.service`:

```ini
[Unit]
Description=Bluetooth auto-accept pairing agent
After=bluetooth.service
Requires=bluetooth.service

[Service]
ExecStart=/usr/bin/bt-agent -c NoInputNoOutput
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now bt-agent
```

## 2. Audio routing

### Version 3 (PulseAudio)

```bash
sudo apt install -y pulseaudio-module-bluetooth
```

PulseAudio loads `module-bluetooth-discover` and `module-bluetooth-policy` by default once the module package is present. The policy module automatically loops an incoming A2DP source to the default sink, which the Jukebox has already set to the HiFiBerry. Restart the user PulseAudio and check:

```bash
systemctl --user restart pulseaudio
```

After a phone connects and plays:

```bash
pactl list sources short
```

A `bluez_source.XX_XX_XX_XX_XX_XX.a2dp_source` line should appear and audio should already be coming from the speakers. If not, load the loopback by hand to confirm routing, then fix `default.pa`:

```bash
pactl load-module module-loopback source=bluez_source.XX_XX_XX_XX_XX_XX.a2dp_source
```

### Version 2.4 (ALSA, no PulseAudio)

Use bluez-alsa:

```bash
sudo apt install -y bluez-alsa-utils
```

Run the BlueALSA daemon with only the sink profile, and `bluealsa-aplay` to pump any connected phone into the HiFiBerry device. Override the packaged units:

`/etc/systemd/system/bluealsa.service.d/override.conf`:

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/bluealsa -p a2dp-sink
```

`/etc/systemd/system/bluealsa-aplay.service.d/override.conf`:

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/bluealsa-aplay -D hifiberry 00:00:00:00:00:00
```

`00:00:00:00:00:00` means "any connected device". `hifiberry` must be the ALSA device name in `/etc/asound.conf`; if the softvol cap from the headphone component is used, point at that device so the cap applies to phone audio too.

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now bluealsa bluealsa-aplay
```

## 3. Pair from the phone

Turn on pairing mode (section 4), open Bluetooth settings on the phone or tablet, tap "Aurora's Phoniebox", play anything. Set the phone's media volume to about 70 % and leave the box's own volume at its usual level.

Pairing is remembered on both sides. Next time the phone just reconnects when Bluetooth is on and the box is powered.

## 4. Kid-proofing

**Pairing on demand, not always on.** An always-discoverable speaker can be paired by anyone within range, including a neighbour's phone. Instead, a script toggles discoverability for two minutes:

`/home/pi/bt-pair-mode.sh`:

```bash
#!/bin/bash
bluetoothctl discoverable on
bluetoothctl pairable on
```

With `DiscoverableTimeout = 120` BlueZ turns it off again by itself. Bind the script to a special RFID figure (a "pairing" figure kept by the grown-ups) or to a long press of one of the five buttons:

- 2.4: add a `functionCall` entry in `settings/gpio_settings.ini`, or a card command in `rfid_trigger_play.conf`.
- v3: bind the script in the GPIO config or as a card action in the web UI.

**Pause the Jukebox when a phone connects.** Otherwise both play at once. A small watcher does it:

- v3: a script subscribing to `pactl subscribe` for new `bluez_source` sources, calling the Jukebox pause RPC, and resuming nothing on disconnect (the kid swipes a figure to go back).
- 2.4: the same idea with `bluealsa-aplay`'s connection events or a `dbus-monitor` on `org.bluez.Device1 Connected`, calling `playout_controls.sh -c=playerpause`.

**Volume cap.** On v3 the Jukebox's max-volume setting applies to the PulseAudio sink, so phone audio is capped too. On 2.4 phone audio bypasses MPD's volume entirely, so cap it with an ALSA softvol device as described in the headphone component's README, and point `bluealsa-aplay` at that device.

**Auto-disconnect.** Optional: a timer that disconnects an idle phone after an hour so a forgotten tablet in another room cannot wake the box at night.

## 5. Known issues

- **Wi-Fi and Bluetooth share the Pi 4 antenna.** Streaming while the web UI is uploading music over Wi-Fi can stutter. Use 5 GHz Wi-Fi if the router has it, or add a cheap USB Bluetooth dongle and disable the onboard radio with `dtoverlay=disable-bt`.
- **Latency.** A2DP has 100 to 200 ms of delay. Fine for music, noticeable for video from a tablet. Nothing to fix; it is the protocol.
- **Battery.** Streaming from a phone keeps the Pi and Amp fully awake, so it draws the same as Jukebox playback. The idle shutdown timer will not see phone audio as activity unless the pause watcher above pokes the Jukebox; worth a check once the battery is in.
- **Codec.** The Pi will use SBC. Quality is fine for a kid's speaker. AAC and aptX are not worth chasing.

## 6. Test plan

1. Pair one phone, play music, confirm it comes out of both speakers at a sane volume.
2. Forget the box on the phone, re-pair without touching the Pi (tests `bt-agent` and `JustWorksRepairing`).
3. Pair a second device (the tablet). Confirm only one can play at a time and the second can take over after the first disconnects.
4. Walk away with the phone and come back: auto-reconnect.
5. Start a Jukebox playlist, then connect the phone: Jukebox pauses.
6. Wait out the two-minute window and confirm the box is no longer visible to a fresh phone.

## Reference: headphones out (not planned)

If headphone output is ever wanted, version 3 has it as a core "secondary output" with automatic switching: pair with `bluetoothctl`, re-run the audio config tool, restart `jukebox-daemon`. On 2.4 it is the `components/bluetooth-sink-switch` add-on, which needs a second ALSA device, a second MPD output, and a volume cap fix. Receiving from a phone (this document) and sending to headphones can coexist, but not at the same moment.

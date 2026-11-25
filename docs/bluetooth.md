# Bluetooth on Aurora's Phoniebox

Two different features, two different setups. Both use the Pi 4's built-in Bluetooth radio, so no extra hardware is needed. The HiFiBerry Amp stays the primary output in every case.

The box runs classic **version 2.4** today. Bluetooth headphone output exists there as a community component with manual ALSA and MPD wiring. In version 3 it is a core feature with automatic switching. If the upgrade in [upgrade-to-v3.md](upgrade-to-v3.md) happens, skip straight to section A2.

## A. Play to Bluetooth headphones or a Bluetooth speaker

### A0. Pairing (same for both versions)

Put the headphones in pairing mode, then on the Pi:

```bash
sudo bluetoothctl
```

```
scan on
scan off
pair AA:BB:CC:DD:EE:FF
trust AA:BB:CC:DD:EE:FF
connect AA:BB:CC:DD:EE:FF
exit
```

`trust` is what makes the headset reconnect on its own later. Pairing a new device needs an SSH session once; after that, trusted devices reconnect automatically.

### A1. Version 2.4: the `bluetooth-sink-switch` component

What it does: toggles MPD's output between the sound card and Bluetooth headphones from a GPIO button, an RFID card, or the web UI. It is classic-edition only because it relies on MPD having two outputs.

1. Install bluealsa (the classic edition does not use PulseAudio) and add a second ALSA device in `/etc/asound.conf`, keeping the existing HiFiBerry entry:

   ```
   pcm.btheadphone {
       type plug
       slave.pcm {
           type bluealsa
           service "org.bluealsa"
           device "AA:BB:CC:DD:EE:FF"
           profile "a2dp"
           delay -20000
       }
   }
   ```

   Test both outputs:

   ```bash
   aplay -D btheadphone /usr/share/sounds/alsa/Front_Center.wav
   ```

   ```bash
   aplay -D hifiberry /usr/share/sounds/alsa/Front_Center.wav
   ```

2. In `/etc/mpd.conf` add a second `audio_output` block for `btheadphone` with `enabled "no"`, after the HiFiBerry block. Order matters. Restart MPD and check with `mpc outputs`.

3. Run the component installer:

   ```bash
   cd ~/RPi-Jukebox-RFID/components/bluetooth-sink-switch && ./install-bt-sink-switch.sh
   ```

4. Bind a toggle. For a sixth GPIO button, in `settings/gpio_settings.ini`:

   ```ini
   [BluetoothToggle]
   enabled: True
   Type: Button
   Pin: 5
   pull_up: True
   hold_time: 0.3
   functionCall: functionCallBluetoothToggle
   ```

   Or assign an RFID figure with `CMDBLUETOOTHTOGGLE="<card id>"` in `rfid_trigger_play.conf`. An optional status LED on GPIO 13 shows which output is active.

Caveats from the component's README:

- Output is always the speakers after boot. Automatic switch-on-connect only works together with the `buttons-bluetooth-headphone` component.
- Headphone and speaker volume differ a lot. The suggested fix is an ALSA softvol block capping the speakers at -20 dB, which renames the mixer to "Master" and requires matching edits in `mpd.conf` and `settings/Audio_iFace_Name`. Undoing softvol is awkward.
- Debug with `bt-sink-switch.py toggle debug`.

### A2. Version 3: built-in secondary output

Version 3 has a primary output (the HiFiBerry) and an optional secondary output aimed at Bluetooth headsets. Audio moves between them on user input or automatically when the Bluetooth device connects.

1. Pair as in A0, then confirm PulseAudio sees the sink:

   ```bash
   pactl list sinks short
   ```

   Expect a line like `bluez_sink.AA_BB_CC_DD_EE_FF.a2dp_sink` beside the HiFiBerry sink.

2. Re-run the Jukebox audio config tool so the headset is registered as the secondary output, then:

   ```bash
   systemctl --user restart jukebox-daemon
   ```

3. Optional: headset buttons (play, pause, next) are supported through a Bluetooth buttons component, so the kid can control playback from the headphones.

### Battery note

When audio goes to headphones the HiFiBerry draws almost nothing, so battery runtime goes up noticeably.

## B. Let a phone stream to the box's speakers (not a Jukebox feature)

This turns the Pi into a Bluetooth receiver (an A2DP sink). It is a plain Raspberry Pi OS task and runs underneath the Jukebox, in either version.

1. Make the Pi discoverable and pairable, with a no-PIN agent so a phone can pair without a keyboard:

   ```bash
   sudo bluetoothctl
   ```

   ```
   power on
   discoverable on
   pairable on
   agent NoInputNoOutput
   default-agent
   ```

   For a permanent setting, add `DiscoverableTimeout = 0` and `Class = 0x200414` (audio speaker) under `[General]` in `/etc/bluetooth/main.conf`.

2. Route the incoming stream to the HiFiBerry. On version 3 (PulseAudio) install `pulseaudio-module-bluetooth` and the A2DP source is routed to the default sink. On version 2.4 (ALSA) use `bluealsa-aplay` as a service, pointing it at the HiFiBerry device.

3. Pair from the phone. Audio then plays through the Amp alongside Jukebox playback. A small script can pause the Jukebox when a phone connects, if the overlap is annoying.

Trade-offs:

- A permanently discoverable speaker means any phone nearby can pair. Limit discoverability to a button press or a time window.
- The Pi 4's onboard radio shares an antenna with Wi-Fi. Heavy web UI use over Wi-Fi while streaming can cause stutter. A cheap USB Bluetooth dongle fixes it.
- Received Bluetooth audio bypasses the Jukebox volume limits unless routed through the same chain, so a child-safe volume cap needs checking.

## Open questions

- Which direction is wanted first: headphones out, phone in, or both?
- Stay on 2.4 or upgrade first? The upgrade makes section A much simpler.
- Should pairing be locked down (pair only via a button hold) for a kid's device?

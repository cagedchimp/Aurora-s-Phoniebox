# Bluetooth on Aurora's Phoniebox

Two different features, two different setups. Both use the Pi 4's built-in Bluetooth radio, so no extra hardware is needed. The HiFiBerry Amp stays the primary output in every case.

## A. Play to Bluetooth headphones or a Bluetooth speaker (supported by Jukebox v3)

The Jukebox version 3 software has a primary output (the HiFiBerry) and an optional secondary output, aimed at Bluetooth headsets and speakers. Audio is moved between them on user input, or automatically when the Bluetooth device connects.

1. Put the headphones in pairing mode, then on the Pi:

   ```bash
   bluetoothctl
   ```

   ```
   scan on
   scan off
   pair AA:BB:CC:DD:EE:FF
   trust AA:BB:CC:DD:EE:FF
   connect AA:BB:CC:DD:EE:FF
   exit
   ```

   Use `sudo bluetoothctl` if it complains about permissions. `trust` is what makes the headset reconnect on its own later.

2. Confirm PulseAudio sees the new sink:

   ```bash
   pactl list sinks short
   ```

   Expect a line like `bluez_sink.AA_BB_CC_DD_EE_FF.a2dp_sink` next to the HiFiBerry sink. Test it:

   ```bash
   paplay -d bluez_sink.AA_BB_CC_DD_EE_FF.a2dp_sink /usr/share/sounds/alsa/Front_Center.wav
   ```

3. Re-run the Jukebox audio config tool so the headset is registered as the secondary output, then restart the daemon:

   ```bash
   systemctl --user restart jukebox-daemon
   ```

4. Optional: the Jukebox has a component for Bluetooth headset buttons (play, pause, next) so the kid can control playback from the headphones. Its doc page could not be fetched from this session, so the exact enable step is still to be confirmed.

Notes:

- Pairing a new device needs a keyboard or SSH session once. After that, trusted devices reconnect automatically.
- When audio goes to headphones the HiFiBerry draws almost nothing, so battery runtime goes up noticeably.
- If the box runs the classic version 2.x software, Bluetooth output is a community add-on rather than a core feature, and the steps differ. Confirm the version first.

## B. Let a phone stream to the box's speakers (not a Jukebox feature)

This turns the Pi into a Bluetooth receiver (an A2DP sink). It is a plain Raspberry Pi OS task and runs underneath the Jukebox.

Outline:

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

2. Ensure PulseAudio's Bluetooth modules are present (`pulseaudio-module-bluetooth`) and loaded, so an incoming A2DP stream shows up as a source that PulseAudio routes to the default sink, the HiFiBerry.

3. Pair from the phone. Audio from the phone then plays through the Amp alongside, or on top of, Jukebox playback. A small script or Jukebox RPC call can pause the Jukebox when a phone connects, if the overlap is annoying.

Trade-offs:

- A permanently discoverable speaker means any phone nearby can pair. Limit discoverability to a button press or a time window if that matters.
- The Pi 4's onboard radio shares an antenna with Wi-Fi. Streaming over Bluetooth while the web UI is in heavy use over Wi-Fi can stutter. A cheap USB Bluetooth dongle on a short extension solves it if it shows up.
- Received Bluetooth audio bypasses the Jukebox volume limits and equalizer unless routed through the same PulseAudio chain, so a child-safe volume cap needs checking.

## Open questions

- Which direction is wanted first: headphones out, phone in, or both?
- Jukebox version installed (v3 or classic 2.x)?
- Should pairing be locked down (pair only via a button hold) for a kid's device?

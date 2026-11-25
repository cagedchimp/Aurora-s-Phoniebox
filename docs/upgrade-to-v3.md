# Upgrading from version 2.4 to version 3

## Why

Both planned features are built into version 3 and bolt-ons in 2.4:

| Feature | Version 2.4 | Version 3 |
| --- | --- | --- |
| Battery level and safe shutdown | custom service needed | INA219 battery monitor component |
| Bluetooth headphone output | `bluetooth-sink-switch` component, manual ALSA and MPD edits | core secondary output, auto-switch on connect |
| Idle shutdown timer | yes | yes |
| Project direction | maintenance mode per the v2.9.0 release notes | active development |

## State of version 3 (checked 2026-11-25)

- Newest tagged version 3 release is v3.6.0 ("Full Bookworm support, 64-bit"), with v3.7.0-alpha as a pre-release.
- GitHub still marks v2.9.0 as "Latest". Version 3 is not labelled stable, but the v2.9.0 notes say v3 "will soon be the new default".
- Raspberry Pi OS Bookworm Lite is the recommended base. Trixie is not yet supported. Pi 4 is more than enough.

## What the upgrade actually involves

There is no in-place upgrade. The install guide covers a fresh image only.

1. Back up from the running 2.4 box: the audio folders (`shared/audiofolders`), the card assignments (`shared/shortcuts`), `settings/gpio_settings.ini`, `/etc/asound.conf`, and `/boot/config.txt` for the HiFiBerry overlay line.
2. Image a **new** SD card with Raspberry Pi OS Lite (Bookworm). Keep the 2.4 card untouched as the fallback.
3. Run the version 3 installer:

   ```bash
   cd; bash <(wget -qO- https://raw.githubusercontent.com/MiczFlor/RPi-Jukebox-RFID/future3/main/installation/install-jukebox.sh)
   ```

4. Configure the HiFiBerry as the primary output (the installer asks about the sound card; the overlay line in `config.txt` is the same as today).
5. Copy the audio folders back. Card assignments in v3 use a different format, so the figures get re-registered through the web UI, one swipe each. With a handful of 3D-printed figures this is a few minutes.
6. Recreate the five GPIO buttons in the v3 GPIO config (YAML instead of the 2.4 ini file).
7. Test playback, buttons, and reader. Only then put the card in the box.

## Risks

- Version 3 is newer and some corners are rougher than 2.4. Keeping the old card means a five-minute rollback.
- Spotify support, if ever used, should be checked before switching. It was not mentioned for this build.
- Expect an evening of work including re-registering figures.

## Recommendation

Upgrade on a fresh card before the battery and Bluetooth work. It removes the custom battery service and the fiddly ALSA/MPD Bluetooth wiring, and it is where the project is heading.

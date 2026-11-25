# Aurora's Phoniebox

A kid-friendly RFID jukebox built on the [RPi-Jukebox-RFID](https://github.com/MiczFlor/RPi-Jukebox-RFID) (Phoniebox) project.

## Hardware as built

| Part | What is in the box |
| --- | --- |
| Computer | Raspberry Pi 4 (to be confirmed) |
| Software | RPi-Jukebox-RFID classic, version 2.4 (upgrade to version 3 under consideration, see docs) |
| Audio | HiFiBerry Amp (Amp2 class, 12 to 24 V input, powers the Pi from its onboard regulator) driving two speakers |
| RFID | USB RFID reader; NFC tags embedded in 3D-printed figures |
| Controls | 5 GPIO push buttons |
| Power | 12 V DC wall supply into the HiFiBerry Amp |
| Housing | [Phoniebox housing by Doblist on Printables](https://www.printables.com/model/461626-phoniebox-jukebox-for-the-raspberry-pi-playing-aud), with the flat side panels laser cut instead of printed |

## Documents

- [Battery / portability plan](docs/battery-plan.md)
- [Bluetooth: phone/tablet streaming into the box](docs/bluetooth.md)
- [Upgrade 2.4 to version 3](docs/upgrade-to-v3.md)

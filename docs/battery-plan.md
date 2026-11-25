# Battery plan: making Aurora's Phoniebox portable

Status: draft for discussion. Nothing here has been bought or wired yet.

## 1. What we are powering

The HiFiBerry Amp2-family boards take a single 12 to 24 V DC input and generate the Pi's 5 V on board. HiFiBerry's datasheet gives a recommended operating range of 12 to 20 V, and 12 to 18 V with 4 ohm speakers. There is no separate 5 V input to the Pi today, so the battery must deliver a HiFiBerry-safe voltage, not 5 V. The "step down" in the current build is almost certainly the Amp's own regulator, not a separate module (to be confirmed by opening the box).

Consequences:

- A 5 V Pi UPS HAT (PiSugar, PiJuice, Waveshare UPS) does **not** fit this build. The HAT slot is taken by the Amp, and the Amp needs 12 V anyway.
- Any battery must stay above roughly 12 V while discharging, or sit behind a regulated 12 V converter.
- The pack must handle the Pi 4's draw plus audio peaks.

## 2. Power budget (estimate, to be measured)

| Load | Typical | Peak |
| --- | --- | --- |
| Pi 4 with USB RFID reader, playing audio | 4 W | 7 W |
| Amp regulator losses (about 85 % efficient) | 1 W | 1.5 W |
| Audio at child-safe volume, two speakers | 1 to 3 W | 20 W (brief) |
| **Total at the 12 V input** | **7 to 9 W, about 0.7 A** | **~30 W, about 2.5 A** |

Design target: 10 W average, 3 A peak at 12 V.

Runtime rule of thumb: pack watt-hours divided by 10 gives hours of play.

| Pack | Energy | Approx. runtime |
| --- | --- | --- |
| 4S1P 18650, 3000 mAh cells | 43 Wh | 4 h |
| 4S2P 18650, 3000 mAh cells | 86 Wh | 8 h |
| 3S 11.1 V 6000 mAh commercial pack | 67 Wh | 6 h if voltage held |

Step 1 of the build is to measure the real draw with a USB-C or barrel inline power meter on the existing 12 V supply at normal and loud volume, then resize.

## 3. Options

### Option A: Regulated 12 V battery pack into the existing barrel jack (lowest effort)

Buy a lithium pack with a **regulated** 12 V DC output rated for at least 3 A and plug it into the Amp's existing power jack. The box behaves exactly as it does on the wall supply today.

- Pros: no soldering, no change to the Pi or Amp, charger included, built-in on/off switch.
- Cons: the Pi cannot see the battery level or shut down safely; most packs do not play while charging; many "12 V" packs are really 3S lithium that sags from 12.6 V to 9 V and are **not** regulated. The common Talentcell YB1206000, for example, is 11.1 V nominal with an output that drops to 9 V, which is below the HiFiBerry minimum for most of its discharge. Avoid those.
- Must check before buying: output is regulated 12 V, 3 A continuous, supports charging while in use (UPS mode) if we want it to still work plugged in.

### Option B: 4S lithium-ion pack with BMS, direct to the Amp (recommended)

Four lithium-ion cells in series run from 16.8 V full to 12.0 V empty. That is exactly inside the HiFiBerry 12 to 18 V window for 4 ohm speakers, so no converter is needed and nothing is wasted in conversion.

- Cells: 4 or 8 × 18650 (4S1P or 4S2P) in a holder or a prebuilt 4S pack with a protection board (BMS) rated 10 A or more.
- Charging: a 16.8 V 2 A CC/CV charger through a second DC jack, or a 4S charge/power-path module so the box plays while charging.
- Gauge: an INA219 current/voltage sensor on the pack output, wired to the Pi's I2C pins (SDA, SCL, 3.3 V, GND). The Jukebox v3 battery monitor plugin reads it, shows state of charge in the web UI, and shuts the Pi down at 3.0 V per cell (12.0 V pack), which matches the Amp's lower limit.
- Switch: a latching power switch on the pack output, kid-proof (recessed or on the back).
- Pros: longest runtime per volume, Pi knows battery level, safe shutdown, keeps wall operation.
- Cons: some soldering and wiring; must respect lithium safety (protected cells or a BMS, fused output, no loose cells in a kid's toy).

### Option C: 4S LiFePO4 pack (safest chemistry)

12.8 V nominal, 14.6 V full, 10 V empty. Safer and longer-lived than lithium-ion, but it drops below 12 V with roughly 20 % capacity left. Workable if the cutoff is set at 12 V and we accept losing the tail of the capacity. Heavier for the same energy.

### Recommendation

Option B. It fits the Amp's electrical needs with no extra converter, and the Jukebox software already supports the INA219 so the battery level and safe shutdown come almost for free. Option A is the fallback if soldering is off the table, with the explicit caveat that the pack must be a regulated 12 V output.

## 4. Safe shutdown and software

The box currently runs the **classic version 2.4**. That version has no battery monitor, so there are two routes.

### Route 1 (preferred): upgrade to version 3, use the built-in battery monitor

RPi-Jukebox-RFID version 3 ships a `battery_monitor` component with drivers for the INA219 and ADS1015 I2C boards. Behaviour, from the source:

- Polls every 5 s and publishes state of charge, voltage, and charging state to the web UI.
- Below 3.3 V per cell it fires the configured `warning_action` (for example play a "battery low" sound).
- Below 3.0 V per cell it calls the host shutdown. This threshold is hardcoded.
- `scale_to_phy_num` / `scale_to_phy_denom` scale the measured pack voltage down to a single-cell value, so a 4S pack is reported as pack voltage divided by 4.

Sketch of the config addition in `jukebox.yaml`:

```yaml
modules:
  named:
    battmon: battery_monitor.batt_mon_i2c_ina219
battmon:
  scale_to_phy_num: 1
  scale_to_phy_denom: 4
  warning_action:
  all_clear_action:
```

The exact scaling values need to be verified against the plugin code before use. Version 3 also has an idle shutdown timer, which matters on battery. See [upgrade-to-v3.md](upgrade-to-v3.md) for the upgrade itself.

### Route 2: stay on 2.4, add a small battery service

Write a short Python systemd service that reads the INA219 over I2C every few seconds, plays a warning sound through the existing Phoniebox scripts at about 13.2 V (3.3 V per cell), and runs the Phoniebox shutdown script at 12.0 V. About 60 lines of code, no changes to the Jukebox itself. Version 2.4 has an idle shutdown option in the web settings that should be turned on as well.

Either way, a hard power switch alone is not enough: cutting power without a shutdown risks SD card corruption.

## 5. Mechanical

Open questions about the housing, because the Printables page could not be read from this session:

- Exterior dimensions and how much free interior volume is left after the Pi, Amp, speakers and reader.
- Whether a 4S1P 18650 holder (about 78 × 80 × 22 mm) or a flat 4S pouch pack fits better.
- Where the charge jack and power switch go, ideally on the back panel, which is laser cut and therefore easy to re-cut.
- Weight budget for a child carrying it.

## 6. Bill of materials (Option B, draft)

| Item | Qty | Notes |
| --- | --- | --- |
| 18650 lithium-ion cells, 3000 mAh or better, from a reputable brand | 4 or 8 | or a prebuilt 4S pack with BMS |
| 4S BMS / protection board, 10 A or more, with balancing | 1 | skip if pack has one |
| 4S cell holder or pack enclosure | 1 | |
| 16.8 V 2 A lithium-ion charger | 1 | CC/CV, barrel plug |
| Panel-mount DC barrel jack for charging | 1 | |
| Latching rocker or recessed power switch, 5 A | 1 | |
| Inline fuse holder and 5 A fuse | 1 | on the pack output |
| INA219 breakout board | 1 | I2C to the Pi |
| Silicone wire, 18 AWG, connectors (XT30 or JST) | | |
| Optional 4S power-path / charge-while-playing module | 1 | if wall + battery at the same time is required |

## 7. Steps

1. Open the box and confirm: Pi model, exact HiFiBerry board, whether a separate step-down exists, free interior space, and which Jukebox version is installed.
2. Measure real power draw at the 12 V input, quiet and loud.
3. Choose pack size from the measurement and the runtime target.
4. Order parts.
5. Bench test the pack with the Amp outside the box, including full discharge to confirm the 12 V cutoff.
6. Wire the INA219, enable the battery monitor plugin, confirm UI readout and shutdown.
7. Re-cut the back panel for the charge jack and switch, mount the pack, strain-relieve all wires.
8. Document the final wiring and config in this repo.

## 8. Open questions for the owner

- Target runtime on one charge?
- Must it play while charging, or is charge-then-play acceptable?
- Comfort with soldering and lithium pack assembly?
- Budget ceiling?
- Upgrade to version 3 before the battery work, or stay on 2.4?
- Housing free space, or a photo of the inside.

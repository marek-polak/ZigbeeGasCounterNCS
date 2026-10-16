# Hardware

This directory contains the 3D-printable parts used to assemble the gas counter enclosure.

The enclosure is designed to hold:

- The main electronics board (Seeed Studio XIAO nRF52840)
- Two 18650 batteries connected in parallel (wired directly to the backside `BAT+` and `BAT-` pads on the XIAO, using the board's built-in `P0.14` / `P0.31 AIN7` battery voltage divider)
- The main push button (wired between `3V3` and `D0 / P0.02`, using the nRF52840 internal `GPIO_PULL_DOWN`)
- The [KY-021 magnetic reed sensor module](https://www.elecom.sk/magneticky-senzorovy-modul-ky-021/) (with onboard `10 kΩ` resistor, wired to `3V3`, `GND`, and `D1 / P0.03`) used to detect gas meter pulses

See [`xiao_gpio_wiring.html`](xiao_gpio_wiring.html) in this directory for the interactive color-coded SVG schematic and logic analyzer.

The main body has an opening at the top for inserting a 5 mm × 10 mm threaded insert, which is used to attach the enclosure to the gas meter. The triangular-looking part with a cylindrical section in the middle is designed to hold the reed sensor in the correct position.

## Power Source

The device is powered by two 18650 batteries connected in parallel.

The reason for using two batteries is to make battery maintenance easier: one battery can be removed and charged externally while the device continues running from the other battery.

Always keep at least one battery installed while replacing or charging the other one. This prevents the device from losing power and helps ensure that no reed sensor pulses are missed during battery maintenance.

## Materials

All parts were printed in PETG.

PETG was chosen because it provides a good balance between mechanical strength, durability, and ease of printing.

## Magnets

Some parts include small cylindrical holes measuring approximately 4.75 mm × 1.65 mm. These holes are intended for small magnets.

The magnets are used to hold the battery holder, main body, and cover plate together without requiring screws for those parts.

## License

The models are shared in STEP format so they can be adapted to different setups or modified for your own needs.

These hardware models are licensed under the **Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License**.

You are free to copy, modify, and adapt the models for personal or non-commercial use, provided that you give appropriate credit and share any derived work under the same license.

Commercial use is not allowed.

## Downloads

- [Gas counter - Main body](Gas%20counter%20main%20body.step)
- [Gas counter - Top cover](Gas%20counter%20top%20cover.step)
- [Battery holder - Top side](Battery%20holder%20-%20top%20side.step)
- [Battery holder - Bottom side](Battery%20holder%20-%20bottom%20side.step)
- [Battery holder - Lock for hole under negative pole](Battery%20holder%20-%20lock%20for%20hole%20under%20negative%20pole.step)
- [Battery holder - Lock for hole under positive pole](Battery%20holder%20-%20lock%20for%20hole%20under%20positive%20pole.step)

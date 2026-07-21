# Corne Wireless + Prospector dongle — clean setup

This is a from-scratch Prospector dongle integration for the KeyboardHoarders
Corne Wireless (42-key, 6-column). It starts from the stock
`KeyboardHoarders/zmk-config-cornekbh` config and adds a `corne_dongle` central
shield plus the `prospector-zmk-module`.

## Why the keys were shifted last time

The Corne halves you flash use ZMK's **built-in `corne` shield**
(`corne_left` / `corne_right`). That shield defines **two** physical layouts —
a 5-column and a 6-column one — plus a position map that ties them together
(`app/boards/shields/corne/corne.dtsi` + `dts/layouts/foostan/corne/*.dtsi`).

The previous dongle overlay only included the **6-column** layout and only the
`default_transform`:

```
#include <layouts/foostan/corne/6column.dtsi>   // only one layout
&foostan_corne_6col_layout { transform = <&default_transform>; };
// ...only default_transform defined
```

ZMK's dongle guide warns about exactly this:

> It is very important that all keyboard parts have the exact same physical
> layouts and matrix transforms with the same devicetree node names. For
> keyboards with multiple physical layouts, make sure to include all of them in
> the dongle shield, even if you only use one of them.

Because the dongle (central) and the halves (peripherals) had a **different set
of physical layouts**, ZMK's position-map logic renumbered incoming key
positions — so every keypress landed on the wrong keymap slot. That's the
"shifted keys" symptom.

## The fix

`boards/shields/corne_dongle/corne_dongle.overlay` now mirrors the built-in
corne shield **exactly** — both layouts, both transforms, mock kscan:

```
#include <layouts/foostan/corne/5column.dtsi>
#include <layouts/foostan/corne/6column.dtsi>

&foostan_corne_6col_layout { transform = <&default_transform>; };
&foostan_corne_5col_layout { transform = <&five_column_transform>; };

/ {
    chosen {
        zmk,kscan = &mock_kscan;
        zmk,physical-layout = &foostan_corne_6col_layout;
    };
    default_transform:      keymap_transform_0 { ... };  // 12x4
    five_column_transform:  keymap_transform_1 { ... };  // 10x4
    mock_kscan: mock_kscan_0 { compatible = "zmk,kscan-mock"; ... };
};
```

The transforms are byte-for-byte identical to
`zmkfirmware/zmk@v0.3 app/boards/shields/corne/corne.dtsi`. The position maps
come along automatically because `5column.dtsi` and `6column.dtsi` each include
`position_map.dtsi`.

Nothing about your keymap changed — `config/corne.keymap` is your existing one.

## Files in this config

```
.github/workflows/build.yml              standard ZMK build action (@v0.3)
build.yaml                               dongle + 2 peripherals + settings_reset
zephyr/module.yml                        makes boards/shields discoverable
config/
  west.yml                               adds carrefinho + prospector-zmk-module
  corne.conf                             shared config + Prospector display options
  corne.keymap                           your keymap (unchanged)
boards/shields/corne_dongle/
  Kconfig.shield
  Kconfig.defconfig                      central role, 2 peripherals, BT_MAX_CONN 7
  corne_dongle.overlay                   FIXED — both layouts + both transforms
```

## Board / branch notes

- `west.yml` pins ZMK to **v0.3** and pulls `prospector-zmk-module@main`, which
  targets ZMK v0.3 / Zephyr 3.5. The XIAO board id for that branch is
  `seeeduino_xiao_ble` (used in `build.yaml`). If you ever move to ZMK `main`,
  you must switch to the module's `feat/new-status-screens` branch and build the
  dongle with board `xiao_ble//zmk` instead.
- Ambient light sensor: `corne.conf` leaves
  `CONFIG_PROSPECTOR_USE_AMBIENT_LIGHT_SENSOR` at its default (`y`). If your
  Prospector has **no** APDS9960 sensor, uncomment the `=n` line — then
  `CONFIG_PROSPECTOR_FIXED_BRIGHTNESS=80` takes effect.

## Build & flash order

1. Push to GitHub and let the **Build ZMK firmware** action run, or build
   locally. You'll get these `.uf2` artifacts:
   - `corne_dongle prospector_adapter` → the Prospector/XIAO dongle
   - `corne_left ...` → left half (now a peripheral)
   - `corne_right ...` → right half (now a peripheral)
   - `settings_reset` (nice_nano_v2 and seeeduino_xiao_ble)

2. **Reset bonds first.** Flash the matching `settings_reset.uf2` to **all
   three** devices (both halves + dongle). Old pairing data is not cleared by a
   normal flash and will cause connection problems.

3. Flash the real firmware: `corne_left` → left, `corne_right` → right,
   `corne_dongle` → the Prospector.

4. **Pair in left-to-right order.** Power on the dongle, then pair the **left**
   half first, then the **right**. The Prospector battery widget orders its bars
   by pairing order, so left-then-right keeps them in the right place.

5. ZMK Studio: plug the **dongle** into USB (the studio snippet is on the dongle
   build) and connect at https://zmk.studio. Unlock via `&studio_unlock` on your
   System layer.

## Reverting to no-dongle

Remove the `prospector-zmk-module` entry from `west.yml`, drop the dongle rows
and the `-DCONFIG_ZMK_SPLIT_ROLE_CENTRAL=n` cmake-args from `build.yaml`, run a
settings reset on the halves, and reflash.

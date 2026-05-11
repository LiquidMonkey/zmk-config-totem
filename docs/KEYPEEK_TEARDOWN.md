# Removing the KeyPeek HUD setup

Once you no longer need the live on-screen layer overlay, back out the firmware-side changes and return the keyboard to a normal ZMK build. The static keymap-drawer cheatsheet (workflow + config file) is harmless and can stay — it costs nothing at runtime and re-renders the SVG on every push.

## What was added (for reference)

| File | Purpose | Safe to keep? |
|------|---------|---------------|
| `config/west.yml` | adds `zmk-raw-hid` + `zmk-keypeek-layer-notifier` modules | **No** — slows builds, pulls extra code |
| `build.yaml` | adds `raw_hid_adapter` shield, `studio-rpc-usb-uart` snippet, `CONFIG_ZMK_STUDIO=y` on `totem_left` | **No** — bloats central firmware |
| `config/totem.conf` | adds `CONFIG_ZMK_STUDIO_LOCKING=y` | **No** — only useful while Studio enabled |
| `config/boards/shields/totem/totem.dtsi` | adds `#include <physical_layouts.dtsi>`, `default_layout` node, `zmk,physical-layout` chosen entry | **Partial** — see below |
| `.github/workflows/draw-keymap.yml` | runs keymap-drawer in CI | **Yes** — static SVG cheatsheet, no runtime cost |
| `keymap_drawer.config.yaml` | keymap-drawer rendering config | **Yes** — see above |

## Teardown steps

### 1. `config/west.yml` — remove module remotes + projects

Revert to the original minimal manifest:

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
  self:
    path: config
```

### 2. `build.yaml` — drop Studio + raw_hid_adapter

```yaml
include:
  - board: xiao_ble//zmk
    shield: totem_left
  - board: xiao_ble//zmk
    shield: totem_right
# there is no settingsreset (needed) for the XIAO
```

### 3. `config/totem.conf` — drop Studio locking

Remove `CONFIG_ZMK_STUDIO_LOCKING=y` and its comment. The file can shrink back to:

```conf
CONFIG_ZMK_USB_LOGGING=n
```

### 4. `config/boards/shields/totem/totem.dtsi` — strip the physical-layout

If you want to keep using keymap-drawer (it can also read the layout from the dtsi for nicer SVG output), **leave the `default_layout` node in place** and only strip what ZMK Studio specifically needs:

- Keep `#include <physical_layouts.dtsi>` and the `default_layout` node — keymap-drawer benefits from them and the runtime cost is zero on flash.
- In `chosen`, you may keep `zmk,physical-layout = &default_layout;` (no harm with Studio disabled).
- Optionally restore `zmk,matrix_transform` (legacy underscore form) if you want bit-for-bit parity with the pre-KeyPeek file. The hyphenated `zmk,matrix-transform` is current ZMK and works fine.

If you want a full revert to the original dtsi, remove the `#include`, the entire `default_layout: default_layout { … };` block, and the `zmk,physical-layout` line from `chosen`. The matrix-transform and kscan blocks must stay.

### 5. Re-flash + re-pair

1. Push the teardown commit — CI builds new `totem_left.uf2` and `totem_right.uf2`.
2. Flash **both halves** (zmk pin moved when removing modules, so the peripheral rebuilds too).
3. On Windows: Settings → Bluetooth → remove the existing Totem pairing → re-pair. The HID descriptor changes when raw HID is dropped, so the cached Windows pairing will misbehave otherwise.
4. Uninstall `keypeek.exe` from Win11 (or leave it — it's harmless without matching firmware).

### 6. Verify

- [ ] CI green on the teardown PR.
- [ ] Build artifacts smaller than before (raw HID + Studio dropped a few KB).
- [ ] All 38 keys type correctly on both halves.
- [ ] Bluetooth re-pair succeeds; keyboard reconnects cleanly after sleep.
- [ ] If keymap-drawer kept enabled: SVG in `keymap-drawer/` folder still updates on every push to `master`.

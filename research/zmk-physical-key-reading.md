# Can a browser or desktop app read physical key presses or the active layer from a ZMK keyboard?

Research for [#4](https://github.com/pauldavidkelly/velvet/issues/4), part of the map [#1](https://github.com/pauldavidkelly/velvet/issues/1). Researched 2026-10-10 against ZMK `main` at `4dd95d7` and `zmk-studio-messages` at `6cb4c28`.

## Answer

**Not with stock firmware, so it doesn't belong in the MVP.** Nothing in stock ZMK or MoErgo's Glove80 firmware reports key positions or layer state to the host. The only standard channel to the host, ZMK Studio RPC, has no key or layer notifications.

**It is possible later, at the cost of custom firmware.** You'd build upstream ZMK for the Glove80 with a community Raw HID module plus a small reporting module, then read it in the browser with **WebHID**. WebHID works only in Chromium browsers (Chrome, Edge). On Linux it also needs a udev rule. This needs no desktop shell: it can be an optional "physical key mode" on top of the plain browser app. The MVP stays as planned: it gets characters and works out which key and layer produced them from the keymap.

## Findings

### 1. ZMK Studio RPC can't report key presses or layers

- Studio talks to the keyboard over USB CDC-ACM serial or over a custom BLE GATT service, using framed protobuf messages ([Studio RPC protocol docs](https://zmk.dev/docs/development/studio-rpc-protocol)).
- The full message set lives in [`zmk-studio-messages/proto/zmk`](https://github.com/zmkfirmware/zmk-studio-messages/tree/main/proto/zmk). It has three request subsystems: `core`, `behaviors` and `keymap`. It has only **two notifications**:
  - `zmk.core.Notification.lock_state_changed` ([core.proto](https://github.com/zmkfirmware/zmk-studio-messages/blob/main/proto/zmk/core.proto))
  - `zmk.keymap.Notification.unsaved_changes_status_changed` ([keymap.proto](https://github.com/zmkfirmware/zmk-studio-messages/blob/main/proto/zmk/keymap.proto))
- No message carries key positions, key events or the active layer. A search of ZMK PRs found no open work to add one.
- **Side finding:** Studio *can read the keymap* (`keymap.get_keymap` returns layers, names and `BehaviorBinding {behavior_id, param1, param2}`, and `get_physical_layouts` returns key geometry). That's a possible way to import a keymap later. It needs a Studio-enabled build ([Studio feature docs](https://zmk.dev/docs/features/studio): `studio-rpc-usb-uart` snippet, `CONFIG_ZMK_STUDIO=y`, central half only). Upstream ZMK's Glove80 board metadata lists the `studio` feature ([glove80.zmk.yml](https://github.com/zmkfirmware/zmk/blob/main/app/boards/moergo/glove80/glove80.zmk.yml)).
- Browser access to Studio uses Web Serial. Per MDN browser-compat-data, that's in Chrome and Edge 89+ and Firefox 151+, but not Safari. The zmk.studio web app supports BLE only on Linux ([Studio docs](https://zmk.dev/docs/features/studio)).

### 2. Stock ZMK has no Raw HID

Upstream ZMK has no vendor-defined or raw HID interface. The only related open PR is [zmk#3390](https://github.com/zmkfirmware/zmk/pull/3390), a vendor "selector key", which isn't a key or layer report. So any reporting channel has to come from a module.

### 3. Community modules already do exactly this

- **[zzeneg/zmk-raw-hid](https://github.com/zzeneg/zmk-raw-hid)** (47 stars, active in 2026) is a transport only. It adds a second HID interface on usage page `0xFF60`, usage `0x61`, with 32-byte reports, over **both USB and BLE** (`src/usb_hid.c`, `src/hog.c`). Its Kconfig is `depends on !ZMK_SPLIT || ZMK_SPLIT_ROLE_CENTRAL`, so it runs on the central half only. On its own it sends nothing; another module has to produce the reports.
- **[hrmt-lab/zmk-rawhid-app](https://github.com/hrmt-lab/zmk-rawhid-app)** is an app protocol on top of a fork of that transport. It has the two reports a tutor needs:
  - `CONFIG_RAWHID_APP_KEY_PRESS`: [`src/key_press.c`](https://github.com/hrmt-lab/zmk-rawhid-app/blob/main/src/key_press.c) listens to `zmk_position_state_changed` and sends `{position, pressed}` for every press and release.
  - `CONFIG_RAWHID_APP_LAYER_STATE_REPORT`: [`src/layer_state_report.c`](https://github.com/hrmt-lab/zmk-rawhid-app/blob/main/src/layer_state_report.c) sends the highest active layer plus the 32-bit layer mask on `zmk_layer_state_changed`. These reports are **debounced by 50 ms**, so with that module the layer is only roughly lined up with each key press.
  - It's a small, young project with no stars. Treat it as a reference implementation, not a dependency to rely on.
- **[4weaver/layer-tutor-native](https://github.com/4weaver/layer-tutor-native)** is a design and scaffold, not finished work, for this very use case: a typing tutor fed by `zmk-raw-hid`. It adds a custom `tutor_practice` module that mutes the normal keyboard report in "practice mode", plus a Tauri shell using hidapi. They chose Tauri for macOS focus and shortcut reasons. The protocol doesn't require it. Its MVP rules out BLE.
- Writing our own reporting module is about 30 lines of C (see `key_press.c` above). It would subscribe to `zmk_position_state_changed` and send `{position, state, zmk_keymap_highest_layer_active()}` in one report, so each press carries its own layer with no debounce.

### 4. Glove80 split topology suits this approach

- In ZMK splits, "the internal keyboard state (like active layers) is handled exclusively by the central". The central "receives key position and sensor events from the other parts". Peripherals "cannot communicate with host devices on their own" ([split keyboards docs](https://zmk.dev/docs/features/split-keyboards)).
- On the Glove80 the **left half is the central** (`ZMK_SPLIT_ROLE_CENTRAL default y` for `BOARD_GLOVE80_LH` in [Kconfig.defconfig](https://github.com/zmkfirmware/zmk/blob/main/app/boards/moergo/glove80/Kconfig.defconfig)). The right half reaches it over BLE.
- So a module on the left half sees positions from **both halves** and knows the layer state. The host connection can be USB (cable into the **left** half; the right half's USB doesn't act as a keyboard) or BLE.
- **Caveat on combos and hold-taps:** ZMK's combo engine listens to `position_state_changed` and holds back or swallows events. Undecided hold-taps (home-row mods) also hold other keys' position events and release them later. A reporting listener has to be registered before both to see raw presses at their real time. zmk-rawhid-app ships a `cmake/verify_combo_listener_order.cmake` check for the combo case.

### 5. Firmware build path for a Glove80

- **MoErgo's Layout Editor and [glove80-zmk-config](https://github.com/moergo-sc/glove80-zmk-config)** build from MoErgo's fork with Nix. `config/default.nix` takes only `board`, `keymap` and `kconfig`, with no west modules. So you can't add a Raw HID module through MoErgo's standard path.
- **Upstream ZMK supports the Glove80 directly** ([`app/boards/moergo/glove80`](https://github.com/zmkfirmware/zmk/tree/main/app/boards/moergo/glove80)). So you'd use a standard `zmk-config` repo built by GitHub Actions, with `zmk-raw-hid` and a reporting module added in `west.yml` and the module on the `glove80_lh` build only.
- Cost for the user: move off the Layout Editor, port the keymap to a `.keymap` file, and flash custom firmware on both halves. That's acceptable for "built for me", but a real hurdle for "any Glove80 owner".

### 6. Browser side: WebHID

- WebHID: Chrome and Edge 89+, not in Firefox or Safari (MDN browser-compat-data `api/HID.json`). It needs a secure context, which `localhost` is.
- Chrome blocks collections whose "top-level collection has a protected usage (e.g. generic keyboard, mouse)". A **vendor-defined `0xFF60` collection isn't protected**, so the app can open the raw interface but not the keyboard itself ([Chrome WebHID docs](https://developer.chrome.com/docs/capabilities/hid)).
- **Linux:** hidraw nodes are read-only by default. You need a udev rule (for example `MODE="0664", GROUP="plugdev"`) and group membership (same source). The user runs Arch-based Omarchy, so this is a one-off setup step.
- **Bluetooth:** zmk-raw-hid exposes the raw report over HID-over-GATT. Whether Chromium's WebHID picks up a BLE HID device's vendor collection on each OS is **unverified**. The Chrome docs don't cover transports. Plan for USB first.
- **Ordering:** key characters still arrive as normal keyboard reports (and `keydown` events) on a different interface. The raw report and the character aren't guaranteed to arrive in a fixed order. The app would match them by time, or use the raw stream as the truth and ignore `keydown`.

### 7. What the browser sees with no firmware changes (the MVP baseline)

`KeyboardEvent.code` and `key` show what ZMK *emitted*, not the physical position. Modifier keys pressed through home-row mods do arrive as `keydown` events (for example `ShiftLeft`). Layer keys, and anything that emits nothing, are invisible. That's why the map's plan works backwards from characters to key and layer using the keymap.

## Cost summary

| Option | Firmware change | Browser support | Gives position | Gives layer | Glove80 notes |
|---|---|---|---|---|---|
| Characters + keymap inference (MVP) | None | All | Inferred | Inferred | Works with any firmware, USB or BLE |
| ZMK Studio RPC | Studio build | Web Serial: Chromium, Firefox 151+ | No | No | Useful only for reading the keymap |
| zmk-raw-hid + reporting module + WebHID | Upstream ZMK build with 2 modules, flash both halves | Chromium only; udev on Linux | Yes | Yes | Left half central, cable into left; BLE unverified |
| Same, but a Tauri/Electron shell with hidapi | Same | Any OS, no browser dependence | Yes | Yes | Only worth it to hide keystrokes from the OS or avoid Chromium |

## Open questions

- Is a "physical key mode" (exact key and layer, used to check combos and home-row mods precisely) worth a post-MVP ticket? If so: our own small module, or hrmt-lab's?
- Could reading the keymap via ZMK Studio (`get_keymap`) be a way to import it, alongside parsing `.keymap` or Layout Editor JSON? Do MoErgo Layout Editor builds enable Studio? (Unverified.)
- Does WebHID read a vendor collection over BLE HOG on Linux, macOS and Windows? This needs a hardware test.

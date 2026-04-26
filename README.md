# bsb-protobuf

Protocol Buffer definitions for the BSB (Board-to-Board Serial Bus) protocol used in Flipper One.

BSB is the communication channel between the two processors inside the Flipper One:

- **RK3576** (main CPU, runs Linux) - sends display frames, receives input events and system state
- **RP2350** (MCU co-processor) - drives the display, reads buttons/touchpad/encoder, manages power

The physical transport is SPI + I2C + UART bridges between the two chips. This repo defines the message schema for everything carried over that channel.

---

## Message types

### Frame (`frame.proto`)

The main CPU sends display frames to the MCU, which drives the physical QSPI LCD panel.

```proto
message Frame {
    Screen screen = 1;       // FRONT or BACK
    uint32 width = 2;
    uint32 height = 3;
    Encoding encoding = 4;   // PLAIN, RUN_LENGTH, DEFLATE, or DEFLATE_RUN_LENGTH
    PixelFormat pixel_format = 5;  // RGB888, L8 (8-bit grey), or L4 (4-bit grey)
    bytes data = 6;
}
```

The front screen is the 256x144 pixel LCD (JD9853 controller, BOE GWT-2.39 panel). The back screen field is reserved for future use.

Compression options let the CPU reduce SPI transfer size: `RUN_LENGTH` for solid-color regions, `DEFLATE` for arbitrary pixel data, `DEFLATE_RUN_LENGTH` for a combined pass.

### Input (`input.proto`)

The MCU sends input events to the main CPU. Three event types:

**ButtonEvent** - physical button press or release

| Button | Description |
|--------|-------------|
| `OK` | Center/confirm button |
| `BACK` | Back button |
| `START` | Start/home button |

**SwitchEvent** - position of the physical mode switch

| Position | Description |
|----------|-------------|
| `BUSY` | Busy / locked state |
| `CUSTOM` | User-defined mode |
| `OFF` | Device off |
| `APPS` | Applications mode |
| `SETTINGS` | Settings mode |

**EncoderEvent** - scroll wheel or rotary encoder delta (`sint32`, negative = counterclockwise)

All three are wrapped in a `InputEvent` oneof so the receiver dispatches on type.

### State (`state.proto`)

The top-level envelope. A `State` message carries a timestamp plus a list of `StateUpdate` entries. Each update is a oneof covering all subsystems:

| Field | Type | Description |
|-------|------|-------------|
| `device_name` | `BSB_State.DeviceName` | Device display name |
| `power` | `BSB_State.Power` | Power state and battery |
| `brightness` | `BSB_State.Brightness` | Display backlight level |
| `audio_volume` | `BSB_State.AudioVolume` | Speaker volume |
| `wifi` | `BSB_State.Wifi` | Wi-Fi connection state |
| `update_state` | `BSB_Update.UpdateState` | OTA firmware update progress |
| `update_check` | `BSB_Update.CheckState` | OTA update check result |
| `auto_update_state` | `BSB_Update.AutoUpdateState` | Automatic update state |
| `timezone` | `BSB_State.Timezone` | Current timezone |
| `matter` | `BSB_State.Matter` | Matter protocol state |
| `ble` | `BSB_State.Ble.Ble` | Bluetooth LE state |
| `frame` | `BSB_Frame.Frame` | Display frame (see above) |
| `input` | `BSB_Input.InputEvent` | Input event (see above) |
| `timer` | `BSB_Timer.Timer` | Timer state (JSON payload) |

State updates are batched: a single `State` message can carry multiple `StateUpdate` entries with a shared timestamp.

### Timer (`timer.proto`)

Carries a JSON payload (`BSB_Util.Json`). Used for timer state that doesn't map cleanly to a fixed proto schema.

### Error (`error.proto`)

Carries an error code and optional message. Wrapped in the optional `error` field on `State`.

---

## Directory layout

```
bsb-protobuf/
  frame.proto          -- Display frame message
  input.proto          -- Button, switch, encoder input events
  state.proto          -- Top-level state envelope
  timer.proto          -- Timer state (JSON)
  error.proto          -- Error message
  state/
    audio.proto        -- Audio volume state
    ble.proto          -- Bluetooth LE state
    brightness.proto   -- Backlight brightness
    device_name.proto  -- Device name
    matter.proto       -- Matter protocol state
    power.proto        -- Power / battery state
    timezone.proto     -- Timezone
    update.proto       -- OTA update state
    wifi.proto         -- Wi-Fi state
  util/
    json.proto         -- Generic JSON wrapper
```

Each `.proto` file has a corresponding `.options` file used by the [nanopb](https://jpa.kapsi.fi/nanopb/) generator to control field sizes and encoding options for the embedded (RP2350) side.

---

## Code generation

The `.options` files next to each `.proto` are nanopb generator hints. To generate C code for the MCU:

```bash
python3 -m grpc_tools.protoc \
    --plugin=protoc-gen-nanopb=path/to/nanopb/generator/protoc-gen-nanopb \
    --nanopb_out=. \
    *.proto state/*.proto
```

For Linux/Python consumers, standard `protoc` with the Python plugin works without the nanopb step.

---

## Context

This protocol is internal to Flipper One hardware. External developers building Linux applications on the RK3576 side interact with input and display indirectly through the Linux kernel interfaces (evdev for input, framebuffer or DRM for display) rather than directly over BSB. BSB is the transport layer between the two chips inside the device.

The MCU firmware source that implements the BSB consumer/producer is in [flipperone-mcu-firmware](https://github.com/flipperdevices/flipperone-mcu-firmware).

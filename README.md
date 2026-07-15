# BUSY Bar Protobuf

The Protocol Buffers contract shared between the BSB device firmware and its
clients. Defines how device **state** is reported, how **frames** and **input**
are streamed, and how the timer/profiles are exchanged.

Identical encoded messages flow over **BLE**, **MQTT**, and **WebSocket**. 
Framing, buffering, and connection handling are implemented in the 
corresponding firmware services.

All schemas use `proto3` and are compiled to embedded C with **nanopb**
(see `lib/proto.scons` in the firmware tree), not the standard Google
generator. The `.options` files paired with each `.proto` are nanopb generator
inputs — they carry field constraints that don't exist in `.proto` itself.

## Directory structure

```
.
├── *.proto            # root-level schemas (transport + cross-cutting domains)
├── *.options          # nanopb field options, one per .proto (paired by name)
├── state/             # per-subsystem state schemas, aggregated by state.proto
│   └── *.proto / *.options
└── util/              # shared helper schemas
    └── json.proto / json.options
```

- **Root** holds schemas that don't belong to a single state subsystem.
- **`state/`** holds one schema per device subsystem (power, wifi, ble, matter,
  update, brightness, audio, timezone, device_name). These never transport
  themselves — they are embedded into `state.proto`.
- **`util/`** holds reusable containers shared across domains.

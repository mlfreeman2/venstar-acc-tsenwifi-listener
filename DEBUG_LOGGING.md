# Debug Logging Guide

## Overview

The Venstar ACC-TSENWIFI Listener integration logs the packets it receives, the
devices it discovers, and the roster changes it makes. This is the second thing
to reach for when a sensor isn't behaving — the first is the diagnostics
download (see [Diagnostics vs. debug logging](#diagnostics-vs-debug-logging)),
which needs no configuration and captures the roster and packet counters in one
JSON file.

## Enabling Debug Logging

Add this to your Home Assistant `configuration.yaml`:

```yaml
logger:
  default: info
  logs:
    custom_components.venstar_acc_tsenwifi_listener: debug
```

Then restart Home Assistant.

## What Gets Logged

### Startup (INFO level)

When the integration binds its UDP socket:

```
Listening for Venstar packets on 0.0.0.0:5001
```

If you don't see this line after a restart, the listener never started — check
for a `ConfigEntryNotReady` retry (usually the port is already held by another
process) in the log.

### Decoded packets (DEBUG level)

Every packet that parses and passes validation logs one line before it reaches
the roster:

```
Decoded 428e0486d800 seq=42 temp=22.5°C fault=None purpose=Remote from 192.168.1.50
```

**Key details logged:**
- MAC address (normalized, 12 hex chars)
- Sequence number (used for dedup — the 5x repeats reuse one sequence)
- Temperature in °C (`None` when the packet reports a fault)
- Fault state (`shorted`, `open`, or `None`)
- Purpose (Outdoor / Remote / Return / Supply)
- Source IP the datagram arrived from

### Roster changes (INFO level)

```
Discovered new device 428e0486d800 (Living Room)
New capability on 428e0486d800 (battery=True humidity=False)
Device 428e0486d800 renamed on the wire to 'Bedroom'
Removed device 428e0486d800 from the roster
```

- **Discovered new device** — first time this MAC was seen; a new HA device and
  entities are created.
- **New capability** — the sensor started reporting battery or humidity it
  hadn't before, so additional entities are added.
- **Renamed on the wire** — the sensor's broadcast name changed. A manual rename
  in HA always wins over this (the wire rename is ignored if you've renamed the
  device yourself).
- **Removed from the roster** — a device was deleted from HA.

### Storage (DEBUG / WARNING level)

```
Loaded roster with 4 device(s)
Skipping unreadable roster entry 428e0486d800: <error>   (WARNING)
```

### Dropped packets — NOT logged

Packets that fail to parse or validate are **deliberately not logged**. Port
5001 receives arbitrary broadcast traffic from the whole LAN, so logging every
reject would flood the log. Instead each drop bumps a counter you can read from
the diagnostics download:

- `dropped_unparseable` — not a valid protobuf message (ordinary LAN noise)
- `dropped_invalid` — parsed, but missing `SensorData`, wrong command, a bad
  MAC, or an out-of-range temperature index
- `dropped_filtered` — matched a co-installed emulator's MAC prefix and the
  "ignore locally emulated" option is on
- `deduped` — a valid repeat of a sequence already seen (expected; each reading
  is broadcast 5x)
- `parsed` — packets that decoded successfully

## Diagnostics vs. debug logging

For most "my sensor isn't showing up" questions, start with **diagnostics**, not
the log:

> Settings → Devices & Services → Venstar ACC-TSENWIFI Listener → ⋮ → Download
> diagnostics.

The JSON includes the full roster (every known device, its purpose, firmware,
last-seen time, and last reading) and the packet counters above — a snapshot of
exactly what the listener has received. Reach for debug logging when you need to
watch packets arrive in real time or confirm a specific sensor is (or isn't)
reaching Home Assistant at all.

## Useful Grep Commands

View only decoded packets:
```bash
grep "Decoded" /config/home-assistant.log
```

View only discovery / roster changes:
```bash
grep -E "Discovered|New capability|renamed|Removed" /config/home-assistant.log
```

Watch a single sensor by MAC:
```bash
grep "428e0486d800" /config/home-assistant.log
```

View errors and warnings only:
```bash
grep -E "WARNING|ERROR" /config/home-assistant.log | grep venstar_acc_tsenwifi_listener
```

## Debugging Checklist

### Sensor not appearing in Home Assistant?

1. **Is the listener running?** Look for the `Listening for Venstar packets`
   startup line.
2. **Are any packets arriving?** With debug on, do you see `Decoded ...` lines
   at all? If not, it's a network problem — HA must be on the **same
   VLAN/broadcast domain** as the sensor, since these are UDP broadcasts.
3. **Are packets arriving but being dropped?** Check the `dropped_*` counters in
   diagnostics. A rising `dropped_filtered` means the emulator-ignore option is
   filtering it; a rising `dropped_invalid` means the packet is malformed or the
   MAC isn't a clean 12-hex value.
4. **Is the right sensor there?** Match the MAC in the `Decoded ...` line to the
   sensor you expect.

### Temperature not updating?

1. Are new `Decoded ...` lines appearing for that MAC with changing `seq=`
   values? If the sequence never changes, the sensor is resending one reading.
2. Is `fault=shorted` or `fault=open`? A faulted sensor reports no temperature
   by design.
3. Is `temp=None` with no fault? That indicates the temperature field was absent
   — check the sensor.

### Sensor keeps going unavailable?

Outdoor sensors broadcast every ~5 minutes and go stale after 20; everything
else broadcasts every minute and goes stale after 5. If a sensor flaps,
confirm packets are actually arriving that often (watch the `Decoded ...` lines
and their timestamps).

## Performance note

At DEBUG level every received packet logs a line, and each reading is broadcast
5x. On a busy broadcast domain that adds up. For normal operation leave the
integration at INFO:

```yaml
logger:
  logs:
    custom_components.venstar_acc_tsenwifi_listener: info
```

At INFO you still get startup, discovery, capability, rename, and removal
lines — just not the per-packet `Decoded ...` firehose.

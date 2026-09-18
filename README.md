# WaveShark protocol descriptions

One YAML file per sensor or remote: its pulse timing, how the frame is
found, and the field layout. [WaveShark](https://github.com/v0l/waveshark)
reads these both ways, so a description decodes what it hears and keys what
it is asked to send.

The receiver fetches this repository as a dataset and runs these files over
the copies built into it, so a fix or a new device reaches a running receiver
without a release. A file in `~/.config/waveshark/protocols` overrides both,
which is where to work on one before sending it here.

## Layout

One directory per kind of device, named for what it is rather than who
sells it, since the same layout ships under a dozen brands:

- `weather/` temperature, humidity, rain and wind sensors
- `remotes/` fixed-code keyfobs, gate and garage remotes
- `tpms/` tyre pressure sensors
- `meters/` utility meters
- `security/` door and window sensors, alarm panels

The receiver reads every `.yaml` in the tree, whatever directory it is in;
the directories are for people.

## A description

```yaml
name: Nexus-TH
timing: { ppm: [1000, 2000], reset_us: 5000 }
frame: { bits: 36, repeats: 12 }
fields:
  - { name: id, bits: 8 }
  - { name: battery_ok, bits: 1, type: bool }
  - { name: channel, bits: 2, offset: 1, max: 3 }
  - { name: temperature_c, bits: 12, type: int, scale: 0.1, min: -40, max: 70 }
  - { bits: 4, const: 0xf }
  - { name: humidity_pct, bits: 8, max: 100, omit_if: 0 }
vectors:
  - hex: "5c 90 c2 f3 e0"
    fields: { id: 0x5c, channel: 2, temperature_c: 19.4, humidity_pct: 62, battery_ok: true }
```

- `timing` is `pwm: [short, long]` or `ppm: [short, long]` in microseconds,
  with `reset_us`, and `sync_us` where a frame starts with a sync mark.
  These are rtl_433's numbers and can be copied from its decoders.
- `frame.bits` is the frame length. `invert` complements the slicer's bits
  first, which the fixed-code remotes need. `find: tile` requires identical
  frames tiling the burst end to end; the default corroborates a frame by a
  row start, an exactly sized burst, or a copy one frame away.
  `min_transitions` refuses a frame that is one symbol repeated.
- Fields are read in order, most significant bit first, and every bit of
  the frame belongs to exactly one field or `const`. A field with `at:` is a
  view over bits another field owns, shown on decode and ignored on encode.
  `type` is `uint`, `int` or `bool`; `scale` and `offset` turn a raw count
  into the reading; `min`/`max` reject the implausible; `omit_if` leaves a
  raw value out of the report; `map` names values; `hidden` reads a field
  for conditions only. A group `when: { field: value }` with `fields` and
  `else` reads one layout or the other.
- `vectors` are the test. Each is run both ways and through the slicer, and
  a file whose vectors fail is refused rather than installed.

Layouts here are transcribed from [rtl_433](https://github.com/merbanan/rtl_433)
and the Flipper Zero firmware, which are the only descriptions most of these
devices have.

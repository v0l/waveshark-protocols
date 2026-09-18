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
yields_to: [Rubicson-Temperature]
fields:
  - { name: id, bits: 8, data: int }
  - { name: battery_ok, bits: 1, data: bool, type: bool }
  - { name: channel, bits: 2, data: int, offset: 1, max: 3 }
  - { name: temperature_c, bits: 12, data: float, unit: c, type: int, scale: 0.1, min: -40, max: 70 }
  - { bits: 4, const: 0xf }
  - { name: humidity_pct, bits: 8, data: int, unit: pct, max: 100, omit_if: 0 }
vectors:
  - hex: "5c 90 c2 f3 e0"
    fields: { id: 0x5c, channel: 2, temperature_c: 19.4, humidity_pct: 62, battery_ok: true }
```

### timing

One of `pwm: [short, long]` (the mark carries the bit), `ppm: [short,
long]` (the gap does), `manchester: [half, full]` or `nrz: bit_us`, all in
microseconds, with `reset_us` for the gap that ends a package, `sync_us`
where a sync mark opens each frame, and `tolerance_us`. These are rtl_433's
numbers and copy straight from its decoders.

### frame

- `bits`: the frame's length.
- `invert`: the slicer's bits are complemented first, which the fixed-code
  remotes and several PWM sensors need.
- `find`: how the frame is located. `repeat` (the default) takes a frame at
  a row start, in an exactly sized package, or with a copy one frame away.
  `tile` wants identical frames end to end, as a remote sends. `exact` wants
  the package to be the frame. `rows` wants a row of the frame's length (or
  `row_bits: [lo, hi]`) on `copies` rows, or alone. `sync` searches for
  `sync` (hex, `sync_bits` long) at any bit offset and takes the frame
  `sync_skip` bits after it, `either_polarity` searching the complement too
  and `decode: manchester` or `diff_manchester` reading the bits behind it
  as chips.
- `row_bits: [lo, hi]`: with any other `find`, a row of the package must be
  this long.
- `min_transitions`: refuses a frame that is one symbol repeated.
  `not_constant: n`: refuses one whose first `n` bits are all alike.
- `repeats`: copies one transmission sends, for keying.

### transform and check

`transform` is a list of `reflect_bytes`, `reflect_nibbles` or
`swap_nibbles`, applied to the frame before anything reads it.

`check` is one or a list of `{ kind, over: [from, to], at }`, `over` being
the bits covered (the end exclusive, packed into bytes) and `at` the bit the
stored value starts at. Kinds: `crc8`, `crc8_le`, `crc16`, `crc16_le` (with
`poly` and `init`), `sum8`, `xor8`, `lfsr8` and `lfsr8_reflect` (with `gen`
and `key`), `even_parity` (every byte covered, nothing stored) and
`complement` (the stored bits are the covered bits inverted). `xor` is
applied to the computed value. A frame with a check reports it as verified;
one without reports no integrity check, which the packet list shows.

### fields

Fields are read in order, most significant bit first, and every bit of the
frame belongs to exactly one field in turn, a `const`, or a nameless
`hidden` slot (which a check fills on encode). A field with `at:` is a
view over bits another field owns: reported on decode, ignored on encode.

- `data`: what the field reports, `int`, `float`, `bool` or `text`. Every
  reported field states it, and a file whose line computes something else
  (a `scale: 0.1` on a field said to be `int`) is refused. `unit`: the
  unit a reading is in, one of `c`, `f`, `pct`, `hpa`, `kpa`, `psi`, `v`,
  `mv`, `a`, `w`, `kwh`, `km_h`, `m_s`, `kt`, `mm`, `deg`, `ppm`, `db`,
  `hz`, `mhz`, `s`. Both travel with the report, so a chart or a Home
  Assistant entity reads them off the field rather than off its name.
- `type`: how the bits are read: `uint` (default), `int` (two's complement), `bool`, `bcd`, `hex`,
  `sign_mag`, `tristate` (PT226x pin pairs), `pick` (which `unit` wide slot
  is not `idle`, for a remote with a slot per button), `format` (text from
  other fields, `{name}` or `{name:02}`).
- `not`, `reflect`: the bits are complemented, or sent low bit first.
- `per_byte: 7`: only the low seven bits of each byte carry the value.
- `scale`, `offset`, `convert: f_to_c`, `round`: raw count to reading. A
  whole-number scale keeps the reading a count.
- `min`, `max`: refuse the implausible. `in: [..]`: raw values accepted.
- `map: { raw: value }`, `other: value`: name the values that are not the
  number. `at_least: n` makes a bool of a wider field.
- `omit_if: raw` or a list: not reported, and the first when not supplied.
- `hidden`: read for conditions only. `id: true`: names the transmitter.
- `when: { field: value }` on a field, or a group `when` / `fields` / `else`
  with an optional `model` that renames the report.

### vectors

Each is a frame (`hex`, as the fields see it) and the report it must read
as, with `model` where a group renames it. A vector is run both ways and
through the slicer; a file whose vectors fail is refused rather than
installed.

## What is not here

A description says what it can invert. Alecto V1 and Globaltronics
(checksums with message-dependent terms), Honeywell (a CRC polynomial chosen
by channel), Hideki (parity bits inside every byte), Oregon Scientific, X10
(a Gray-coded house switch), Interlogix, the Acurite 5-in-1, the Toyota, Ford
and Renault TPMS, the ERT meters, KeeLoq, Somfy RTS and the electronic shelf
labels stay written in WaveShark.

Layouts here are transcribed from [rtl_433](https://github.com/merbanan/rtl_433)
and the Flipper Zero firmware, which are the only descriptions most of these
devices have.

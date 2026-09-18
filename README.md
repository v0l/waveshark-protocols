# WaveShark protocol descriptions

One YAML file per sensor or remote: its pulse timing, how the frame is
found, and the field layout. [WaveShark](https://github.com/v0l/waveshark)
reads these both ways, so a description decodes what it hears and keys what
it is asked to send.

The receiver fetches this repository as a dataset. Nothing is built into it,
so these files are what it decodes the ISM bands with, and a fix or a new
device reaches a running receiver without a release. A file in
`~/.config/waveshark/protocols` overrides what was fetched, which is where to
work on one before sending it here.

## Layout

One directory per kind of device, named for what it is rather than who
sells it, since the same layout ships under a dozen brands:

- `weather/` temperature, humidity, rain and wind sensors
- `remotes/` fixed-code keyfobs, gate and garage remotes
- `tpms/` tyre pressure sensors
- `home/` doorbells, mains switches and the like
- `security/` door and window sensors, alarm panels
- `meters/` utility meters

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

### timing, or radio

A protocol read off the burst detector has a `timing`: one of `pwm: [short, long]` (the mark carries the bit), `ppm: [short,
long]` (the gap does), `manchester: [half, full]` or `nrz: bit_us`, all in
microseconds, with `reset_us` for the gap that ends a package, `sync_us`
where a sync mark opens each frame, and `tolerance_us`. These are rtl_433's
numbers and copy straight from its decoders.

A protocol read off its own FSK channel has a `radio` instead, or as well:

```yaml
radio: { fsk: { baud: 38400, deviation_hz: 20000 }, bands: [[433.0e6, 434.8e6]], width_hz: 100000 }
```

`fsk` names the demodulator by `baud` and peak `deviation_hz` (with an
optional `bandwidth_hz` in front of it, Carson's rule if unsaid);
`bands` or `channels` say where in the spectrum the channel can be;
`width_hz` is the channel's width; `rate_hz` the rate the bit clock runs
at (eight samples a symbol if unsaid); `id` the word a scanner names it by.
Such a description appears in WaveShark's mode menu and scanner table and
is placed on any source the detector finds in its bands. Its `find` must be
`sync`, since the bits are a stream.

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
- `row_bits: [lo, hi]`: with `find: rows`, the rows a frame is taken from;
  with any other `find`, a row of the package must be this long.
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
and `key`), `nibble_sum` (with `add` or `negate` from `init`, and `width`
where the stored sum is not a byte), `nibble_xor`, `roll8` (a per-byte LFSR digest keyed from `gen` and only
shifted, which is what Globaltronics' rolling byte is), `even_parity` (every
byte covered, nothing stored) and `complement` (the stored bits are the
covered bits inverted). `xor` is applied to the computed value; `reflect`
stores it low bit first and sums nibbles as they read reversed; `swap`
stores a byte with its nibbles swapped. `when` and `unless` make a check
apply only for some value of a field, for a polynomial chosen by channel.
A frame with a check reports it as verified; one without reports no
integrity check, which the packet list shows.

### fields

Fields are read in order, most significant bit first, and every bit of the
frame belongs to exactly one field in turn, a `const`, or a nameless
`hidden` slot (which a check fills on encode, or `default` does). A field
with `at:` or `gather:` is a view over bits other fields own: reported on
decode, and on encode the bits it says are written into whatever owns
them, so a status byte scattered across a frame still keys. `gather` is a
list of bit positions or `[from, to]` runs, most significant first; a
hidden owner read under the same name as a reported view (a message type
that decides a layout before it is reached) is filled from it.

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
  `sign: bit` names a bit elsewhere that negates the reading.
- `per_byte: 7`: only the low seven bits of each byte carry the value.
- `scale`, `offset`, `convert: f_to_c`, `round`: raw count to reading. A
  whole-number scale keeps the reading a count.
- `min`, `max`: refuse the implausible. `in: [..]`: raw values accepted.
- `map: { raw: value }`, `other: value`: name the values that are not the
  number. `at_least: n` makes a bool of a wider field.
- `omit_if: raw` or a list: not reported, and the first when not supplied.
  `default: raw` is what a hidden owner is written as when nothing says.
- `hidden`: read for conditions only. `id: true`: names the transmitter.
- `when: { field: value }` on a field, or a group `when` / `fields` / `else`
  with an optional `model` that renames the report.

### vectors

Each is a frame (`hex`, as the fields see it) and the report it must read
as, with `model` where a group renames it. A vector is run both ways and
through the slicer; a file whose vectors fail is refused rather than
installed.

## Checking one

The receiver refuses a description that fails its own vectors, so check
before sending. From a WaveShark checkout:

```sh
cargo run --release -p decode --example protocol -- check path/to/protocols
```

It parses each file, runs every vector both ways and through the slicer, and
exits non-zero on a failure. It also says what the vectors cannot: a name
already taken, a frame with no id, one vector where two would catch an id
read off the wrong bits, and how many single-bit corruptions still decode,
which is how many frames the receiver will read off noise.

Two commands help getting there. Given bytes you already trust, `vector`
writes the entry rather than leaving the expected report to be typed out:

```sh
cargo run --release -p decode --example protocol -- vector weather/nexus.yaml "5c 90 c2 f3 e0"
  - { hex: "5c90c2f3e0", fields: { battery_ok: true, channel: 2, humidity_pct: 62, id: 92, temperature_c: 19.4 } }
```

And `read` points the description at a recording, an IQ capture
(`.cu8`, `.cs8`, `.cs16`, `.cf32`) or a Flipper `.sub`, which is the only
way to know the timings are right:

```sh
cargo run --release -p decode --example protocol -- read weather/nexus.yaml nexus_th_433.92M_250k.cu8
reading with Nexus-TH
48 bursts
  burst 0: Nexus-TH battery_ok=true channel=3 humidity_pct=30 id=201 temperature_c=29.4
```

A description earns its place by reading a real recording, not only its own
vectors. Say in the file's comment where the layout came from and what it
was checked against.

## What is not here

A description says what it can invert. Hideki (a parity bit inside every byte), Oregon v2.1 (every bit
sent twice), Interlogix (parity folded over the frame), the ERT meters,
KeeLoq, Somfy RTS, ISM868 and the electronic shelf labels stay written in
WaveShark.

Layouts here are transcribed from [rtl_433](https://github.com/merbanan/rtl_433)
and the Flipper Zero firmware, which are the only descriptions most of these
devices have.

# The description format

Every key a description may carry, with its default. The reader refuses an
unknown key by name, so a typo is caught at install rather than ignored.

A file is one description. Top level:

| key | meaning |
|---|---|
| `name` | the model a report carries, and the name the file is known by |
| `timing` | pulse widths, for a protocol read off the burst detector |
| `radio` | channel and demodulator, for a protocol placed in the graph |
| `frame` | how long a frame is and how it is found |
| `yields_to` | protocols whose decode of the same package outranks this one |
| `transform` | rearrangements applied to the frame's bytes before anything reads them |
| `check` | one integrity check, or a list |
| `fields` | the layout, in order |
| `vectors` | frames and the reports they must read as |

`name` , `frame` and `fields` are required, as is one of `timing` or `radio`.
A description with no `vectors` is refused.

## timing

Exactly one coding, all widths in microseconds:

| key | meaning |
|---|---|
| `pwm: [short, long]` | the mark carries the bit |
| `ppm: [short, long]` | the gap carries the bit |
| `manchester: [half, full]` | a bit is the mid-symbol edge |
| `nrz: bit_us` | a mark is ones and a gap zeros for as long as it lasts |
| `sync_us` | a sync mark before each frame, carrying no bit (default 0) |
| `tolerance_us` | how far a measured width may sit from the table (default 0, which lets the slicer choose) |
| `reset_us` | the gap that ends a package; required |

These are rtl_433's numbers and copy straight out of its decoders. `long`
must exceed `short`, except under `nrz` where there is one width.

## radio

A protocol with its own channel, read off a demodulator rather than off the
burst detector. It appears in WaveShark's mode menu and scanner table and is
placed on any source the detector finds inside its bands.

```yaml
radio:
  fsk: { baud: 38400, deviation_hz: 20000 }
  bands: [[433.0e6, 434.8e6]]
  width_hz: 100000
```

| key | meaning |
|---|---|
| `fsk.baud` | symbol rate |
| `fsk.deviation_hz` | peak deviation from the carrier |
| `fsk.bandwidth_hz` | filter in front of the discriminator; Carson's rule, twice the deviation plus the baud, if unsaid |
| `bands` | `[[lo, hi], ...]` the channel may sit anywhere in |
| `channels` | fixed frequencies, in hertz |
| `width_hz` | the channel's width; required |
| `rate_hz` | rate the channel is cut to before the bit clock; eight samples a symbol if unsaid, and four is where the clock stops working |
| `default_hz` | the frequency offered when placed by hand; the first channel, or the middle of the first band |
| `id` | the word a scanner names it by; the name lowercased if unsaid |

A description with a `radio` must use `find: sync`, because what arrives is a
stream of bits rather than a package with edges. It may carry a `timing` as
well, and is then read both ways.

## frame

| key | default | meaning |
|---|---|---|
| `bits` | required | the frame's length |
| `invert` | false | the slicer's bits are complemented before anything reads them |
| `find` | `repeat` | how the frame is located, below |
| `sync` | | hex of the word the frame is found by, for `find: sync` |
| `sync_bits` | 0 | how much of that hex is the sync |
| `sync_skip` | the sync's length | bits from the sync's start to the frame's |
| `either_polarity` | false | the stream and its complement are both searched |
| `decode` | `none` | `manchester` or `diff_manchester`: the bits behind the sync are chips |
| `copies` | 2 | rows the frame must be found on, for `find: rows`; one takes any row |
| `row_bits` | | `[lo, hi]`: rows a frame is taken from, or a length a row of the package must have |
| `min_transitions` | 0 | fewest symbol changes a frame may have, against a chopped carrier |
| `not_constant` | 0 | a frame whose first this many bits are all alike is not one, which is what silence slices to |
| `repeats` | 1 | copies of the frame one transmission sends, for keying |

`find`:

- `repeat` takes a frame at a row start, in an exactly sized package, or
  where a copy sits one frame away. The default, and right for most sensors.
- `tile` wants identical frames end to end, which is what a remote sends
  while its button is held.
- `exact` wants the package to be the frame, to within a bit.
- `rows` wants a row of the frame's length, or a `row_bits` long one, on
  `copies` rows or alone.
- `sync` searches for `sync` at any bit offset and takes the frame
  `sync_skip` bits later. The only one a `radio` description may use.

## transform

A list, applied in order to the frame's bytes before the checks and the
fields read them: `reflect_bytes` (every byte's bits reversed),
`reflect_nibbles`, `swap_nibbles`. Use these where the device sends a byte
low bit first throughout, rather than putting `reflect` on every field.

## check

One mapping or a list of them. See [checks.md](checks.md) for what each kind
computes and how to work out which one a device uses.

| key | meaning |
|---|---|
| `kind` | required, one of the kinds in checks.md |
| `over: [from, to]` | bits covered, the end exclusive |
| `at` | bit the stored value starts at; omitted for a check that stores nothing |
| `poly`, `init` | CRC polynomial and starting value |
| `gen`, `key` | LFSR generator and key |
| `xor` | applied to the computed value before comparing |
| `add` | added to a sum before comparing |
| `negate` | the sum is taken from `init` rather than compared as it is |
| `width` | width of the stored value where the kind does not fix it |
| `reflect` | the value is stored low bit first, and a nibble sum adds nibbles as they read reversed |
| `swap` | the stored value's halves the other way round: a byte's nibbles, a sixteen bit value's bytes |
| `odd` | the parity is odd rather than even |
| `step` | a parity counts every nth covered bit, for a frame that guards its even and odd bits apart |
| `when`, `unless` | the check applies only when the named fields read so |

A frame with a passing check is reported as verified. A frame with no check
reports no integrity check at all, which the packet list shows, and which is
why a description without one needs constants or an `in:` list to be worth
anything.

## fields

Fields are read in order, most significant bit first, and **every bit of the
frame belongs to exactly one field in turn**, a `const`, or a nameless
`hidden` slot. That is what makes a description invertible, so the same file
keys what it decodes.

| key | meaning |
|---|---|
| `name` | what the field is reported as; omitted for a const or a filler |
| `bits` | width on the air |
| `data` | what it reports: `int`, `float`, `bool`, `text`. Every reported field states it |
| `unit` | `c`, `f`, `pct`, `hpa`, `kpa`, `psi`, `v`, `mv`, `a`, `w`, `kwh`, `km_h`, `m_s`, `kt`, `mm`, `deg`, `ppm`, `db`, `hz`, `mhz`, `s` |
| `type` | how the bits are read: `uint` (default), `int`, `bool`, `bcd`, `hex`, `sign_mag`, `tristate`, `pick`, `format` |
| `const` | the value these bits must hold; checked on decode, written on encode |
| `in: [..]` | raw values accepted; any other is not this protocol |
| `hidden` | read for conditions only, never reported |
| `id: true` | this field names the transmitter |
| `not` | the bits are complemented on the air |
| `reflect` | the bits are sent least significant first |
| `sign: bit` | a bit elsewhere that negates the reading when set |
| `per_byte: 7` | only this many bits of each byte carry the value, the rest being parity; the width still counts every bit on the air |
| `scale`, `offset` | raw count to reading, scale first |
| `convert: f_to_c` | applied after scale and offset |
| `round` | decimals the reading is rounded to; the scale's if unsaid |
| `min`, `max` | refuse the implausible |
| `map: { raw: value }` | the raw values that are not the number |
| `other` | what every raw value the map does not name reads as |
| `at_least: n` | report a bool of whether the raw value reaches this |
| `omit_if` | a raw value, or a list: not reported, and the first is what is written when nothing supplies it |
| `default` | what a hidden field is written as when nothing supplies it |
| `upper` | hex in capitals |
| `slot`, `idle` | slot width and unpressed value, for `type: pick` |
| `format` | template for `type: format`, `{name}` or `{name:02}` |
| `at`, `gather` | this field is a view over bits other fields own |
| `when` | the field is read only when the named fields read so |

`data` is checked against what the line computes: a `scale: 0.1` on a field
that says `data: int` is refused, because the report would carry a float
under a type that says otherwise. Both `data` and `unit` travel with the
report, so a chart or a Home Assistant entity reads the unit off the field
rather than guessing from its name.

### Views

A field with `at:` or `gather:` does not consume bits. It is a second reading
of bits other fields own: reported on decode, and on encode its bits are
written into whatever owns them, so a status byte scattered across a frame
still keys.

- `at: n` reads `bits` bits from bit `n` of the frame.
- `gather: [3, [8, 16], 20]` collects a bit, a run with the end exclusive,
  and another bit, most significant first.

A hidden owner and a reported view may share a name: a message type that has
to be read before the layout that contains it is reached is written that way,
and the hidden one is filled from the view on encode.

### Groups

An item with a `fields:` key is a group rather than a field:

```yaml
- when: { message_type: 5 }
  model: Acurite-5n1
  fields:
    - { name: wind_avg_km_h, bits: 8, data: float, unit: km_h, scale: 0.8 }
  else:
    - { name: rain_mm, bits: 12, data: float, unit: mm, scale: 0.254 }
```

`when` is a mapping of field name to a value or a list of values, all of
which must hold. `model` renames the report when the branch is taken, which
is how one description covers a family that calls itself different things.

## vectors

```yaml
vectors:
  - hex: "5c 90 c2 f3 e0"
    fields: { id: 0x5c, channel: 2, temperature_c: 19.4, humidity_pct: 62, battery_ok: true }
    model: Nexus-TH
```

`hex` is the frame as the fields see it, after `invert` and `transform`.
`fields` is the whole report, and `model` where a group renames it.

Every vector is run three ways at install: decoded, encoded back to the same
bytes, and keyed to pulses and sliced again. A file with a failing vector is
refused rather than run, so a bad push cannot make a receiver read worse than
the one before it. Write vectors from frames a real device sent, and use
`protocol vector` to produce the expected report rather than typing it out.

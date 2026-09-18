# Checks

A check is the difference between a decoder and a random number generator.
Without one, any burst of about the right length reads as a sensor, and the
receiver will report temperatures off passing noise. `protocol check` says
how many single-bit corruptions of your vectors still decode; if that number
is not small, the description needs a check, a constant, or an `in:` list.

## The kinds

| kind | stored | computes |
|---|---|---|
| `crc8` | 8 | CRC-8, MSB first, `poly` and `init` |
| `crc8_le` | 8 | the same, LSB first |
| `crc16` | 16 | CRC-16, MSB first |
| `crc16_le` | 16 | the same, LSB first |
| `sum8` | 8 | every covered byte added, modulo 256 |
| `xor8` | 8 | every covered byte exclusive-ored |
| `lfsr8` | 8 | LFSR digest, `gen` and `key`, MSB first |
| `lfsr8_reflect` | 8 | the same, bits taken LSB first |
| `nibble_sum` | 8 | every covered nibble added |
| `nibble_xor` | 4 | every covered nibble exclusive-ored |
| `roll8` | 8 | a per-byte LFSR digest, the key seeded from `gen` at every byte and only shifted |
| `even_parity` | none | every covered byte has even parity, or odd with `odd: true` |
| `parity` | 1 | the covered bits, `step` apart, counted; the stored bit brings the count to even, or odd with `odd: true` |
| `complement` | the width of `over` | the stored bits are the covered bits inverted |

`over: [from, to]` is in bits with the end exclusive, and must be whole bytes
for the byte-wise kinds. `at` is where the stored value begins; a check that
stores nothing, like `even_parity`, omits it.

Modifiers, all optional: `xor` on the computed value, `add` on a sum,
`negate` to take the sum from `init` instead of comparing, `reflect` to store
low bit first, `swap` to store the value's halves the other way round (a
byte's nibbles, a sixteen bit value's bytes), `odd` for an odd parity,
`step` for a parity that counts every nth bit, and `width` where the stored
value is narrower than the kind's natural size, as a sum stored in six bits
is.

`add` and `negate` apply to `sum8` and `nibble_sum`: a meter whose bytes
including the stored one add up to zero is `negate: true` with `init: 0`, and
one that folds a fixed byte into the sum is `add: 0x56`.

A `parity` check normally names the bit it is stored in, and `over` then
covers everything before it. Where the stored bits sit inside the span they
cover, as the two interleaved parities of a WT450 do, leave `at` out and the
count over the whole span is what has to come out even. Two of them with
`step: 2`, one starting at bit 0 and one at bit 1, are how a frame guards its
even and odd numbered bits separately.

## Working out which one

The device's datasheet almost never says, so this is inference from frames.
Collect a dozen from one transmitter, ideally with one reading changing.

1. **Look for the constant tail.** The last byte or nibble that changes when
   any data bit changes is the check. If nothing changes with the data, there
   is no check and the frame is identified by constants instead.
2. **Try the cheap ones first.** Add the covered bytes and compare, exclusive-or
   them, add the nibbles. Most 433 MHz sensors use one of those three, and
   `add`, `negate` or `xor` accounts for the offset when the arithmetic is
   right but the value is not.
3. **Then CRC-8.** `poly: 0x31` with `init: 0` covers a large share of Fine
   Offset and LaCrosse hardware; `0x07` and `0x8d` turn up too. rtl_433's
   `util.c` has the ones its decoders use, and
   [reveng](https://reveng.sourceforge.io/) will find a polynomial from a
   handful of frames when none of them fit.
4. **Watch for the check that only sometimes applies.** Where a station sends
   several message types down one layout, the polynomial or the covered span
   can change with the type. `when:` and `unless:` on the check express that,
   keyed on a field read earlier in the frame.

A check you cannot compute both ways does not belong in a description. A
rolling code under a key learned by pairing (KeeLoq, Somfy RTS) cannot be
produced from the frame, which is why those stay written in WaveShark.

## When there is no check

Several fixed-code remotes and some of the oldest sensors have none. They are
still worth describing, but they need something else to stand on:

- `const` on every bit the layout fixes, including the preamble and any
  padding that is always the same.
- `in: [..]` on a field whose raw values are known to be a small set, like a
  channel of 0 to 2.
- `min` and `max` on every reading, which throws out the frames where noise
  produced a temperature of 180 C.
- `not_constant` and `min_transitions` in the frame block, which throw out a
  chopped carrier read as all ones.
- `yields_to` naming the protocols whose decode of the same package should
  win, so a checked decoder is not shouted over by a checkless one.

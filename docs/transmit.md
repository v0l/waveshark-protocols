# Keying a description

A description is read both ways. The same file that decodes a sensor keys
one, which is how WaveShark transmits a gate remote or stands in for a sensor
on a bench, and it is why the format looks stricter than a decoder needs.

## What makes a file invertible

Every bit of the frame belongs to exactly one field in turn, a `const`, or a
nameless `hidden` slot. Given the reported fields, the encoder walks the same
list and writes each one back:

- a `const` writes its value,
- a `hidden` slot writes its `default`, or is filled by the check that covers
  it, or by a view that overlaps it,
- a field with `omit_if` writes the first of those values when nothing
  supplies it,
- a view (`at:` or `gather:`) writes its bits into whatever owns them,
- every `check` is computed over the bytes now written and stored at its `at`.

So a description cannot skip bits it did not care about, and cannot read a
field by a rule it has no inverse for. That is the constraint the whole
format is built around, and the reason a check has to be computable rather
than merely recognisable: a rolling code under a pairing-learned key can be
checked against a counter but not produced from the frame, so KeeLoq and
Somfy RTS stay written in WaveShark rather than described here.

Installation proves it. Each vector is encoded back from its expected report
and must produce the exact bytes it started as, then keyed to pulses through
the timing table and sliced again, and must still read the same. A file that
only decodes is refused.

## Pulses

`timing` gives the widths, `frame.repeats` says how many copies one press
sends. A remote that sends its frame six times gets `repeats: 6`, and the
receiver keys six, with the reset gap between them.

`invert: true` applies to keying as well, so a description that inverts what
it slices keys the complement, and the two cancel.

## A channel of its own

A description with a `radio` block keys through the FSK modulator instead of
the burst detector's pulses, at the baud and deviation it declares, with the
shift being twice the deviation. It appears in the mode menu, and the fields
on its source card are what goes out: leave one blank and the first vector's
value is used, so a card touched by nobody still sends something that reads
back.

```yaml
radio:
  fsk: { baud: 38400, deviation_hz: 20000 }
  bands: [[433.0e6, 434.8e6]]
  width_hz: 100000
frame: { bits: 64, find: sync, sync: "aad391", sync_bits: 24 }
```

The sync word is transmitted as well as searched for, since `sync_skip`
places the frame after it and the encoder writes the whole thing.

## Before you key anything

Transmitting on ISM bands is regulated, and the limits differ by country and
by band. Keying a neighbour's gate remote is illegal most places even where
the band is licence free. The receiver will key what it is told to; knowing
whether you may is yours.

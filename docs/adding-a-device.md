# Adding a device

What the work looks like end to end, using a thermo-hygrometer as the
example. You need a WaveShark checkout to run the tool; the description
itself is a text file and needs nothing.

## 1. Get frames out of it

Record the device with WaveShark's "Capture the raw span" switch, or with
`rtl_433 -S all`, or export a `.sub` from a Flipper Zero. A capture holding a
dozen transmissions is enough, and one recorded while a reading changed is
worth more than a dozen of the same number.

If rtl_433 already decodes it, keep its output beside the capture. That is a
second implementation to agree with, and agreement is evidence in a way that
your own reasoning is not.

## 2. Find the layout

Most of these devices are documented only by somebody else's decoder. Look in
[rtl_433's src](https://github.com/merbanan/rtl_433/tree/master/src/devices)
first, then the Flipper Zero firmware. Read the layout out of the code rather
than the comment where the two disagree.

Write down, for the frame: its length in bits, which coding and what the two
pulse widths are, what ends a transmission, and whether the bits arrive
inverted. Then, for each field: where it starts, how wide it is, and what
turns its raw count into a reading.

## 3. Write the file

Put it under the directory for the kind of device, named for the device
rather than the brand. Start from a description of something similar:
`weather/nexus.yaml` is about as simple as they get, `weather/acurite_5n1.yaml`
shows a family sharing one layout through groups.

```yaml
name: Nexus-TH
timing: { ppm: [1000, 2000], reset_us: 5000 }
frame: { bits: 36, repeats: 12 }
fields:
  - { name: id, bits: 8, data: int, id: true }
  - { name: battery_ok, bits: 1, data: bool, type: bool }
  - { name: channel, bits: 2, data: int, offset: 1, max: 3 }
  - { name: temperature_c, bits: 12, data: float, unit: c, type: int, scale: 0.1, min: -40, max: 70 }
  - { bits: 4, const: 0xf }
  - { name: humidity_pct, bits: 8, data: int, unit: pct, max: 100 }
```

Every bit of the frame has to be accounted for, in order. Bits nothing reads
are a nameless `{ bits: n, hidden: true }` or a `const`, and a const is much
better: it is free identification. The full key list is in
[reference.md](reference.md).

## 4. Make it read your capture

```sh
cargo run --release -p decode --example protocol -- read weather/nexus.yaml capture.cu8
```

This installs the description on its own and runs it against every burst the
detector finds. Nothing coming back usually means the timings: widen
`tolerance_us`, check `reset_us` against the gap between transmissions, and
try `invert: true`, which a surprising number of PWM sensors need.

## 5. Write the vectors

Once it reads, turn a frame you trust into a vector rather than typing the
expected report by hand:

```sh
cargo run --release -p decode --example protocol -- vector weather/nexus.yaml "5c 90 c2 f3 e0"
```

Two vectors from two different transmitters, please. One vector cannot catch
an id read off the wrong bits, because every value in it is consistent with
the mistake.

## 6. Check it

```sh
cargo run --release -p decode --example protocol -- check weather/nexus.yaml
```

Every vector is decoded, encoded back to the same bytes, and keyed to pulses
and sliced again. Then come the warnings: a name already taken, a frame with
no id, a single vector, and how many single-bit corruptions still decode.
That last number is how often the receiver will read this off noise, and
[checks.md](checks.md) is what to do about it.

Run `check` over the whole tree before sending, so a new name colliding with
an existing one is caught.

## 7. Send it

One device per pull request. In the file, say where the layout came from and
what it was checked against; in the description, say what hardware you have
and whether rtl_433 agreed.

A description earns its place by reading a real recording, not only its own
vectors. If all you have is somebody else's source, say so, and it will be
merged as unverified rather than silently trusted.

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

Every key, with its default, is in [docs/reference.md](docs/reference.md).
The rest of the documentation:

- [docs/adding-a-device.md](docs/adding-a-device.md), from a capture to a
  merged file
- [docs/checks.md](docs/checks.md), the check kinds and how to work out which
  one a device uses
- [docs/transmit.md](docs/transmit.md), why a description has to invert, and
  what happens when one is keyed

## Checking one

The receiver refuses a description that fails its own vectors, so check
before sending. From a WaveShark checkout:

```sh
cargo run --release -p decode --example protocol -- check path/to/protocols
```

Every vector is decoded, encoded back to the same bytes, and keyed to pulses
and sliced again, and the exit code says whether they all passed. The same
tool writes a vector from a frame (`vector`) and runs a description against a
recording (`read`), which is the walkthrough in
[docs/adding-a-device.md](docs/adding-a-device.md).

A description earns its place by reading a real recording, not only its own
vectors. Say in the file where the layout came from and what it was checked
against.

## What is not here

A description says what it can invert. Hideki (a parity bit inside every byte), Oregon v2.1 (every bit
sent twice), Interlogix (parity folded over the frame), the ERT meters,
KeeLoq, Somfy RTS, ISM868 and the electronic shelf labels stay written in
WaveShark.

Layouts here are transcribed from [rtl_433](https://github.com/merbanan/rtl_433)
and the Flipper Zero firmware, which are the only descriptions most of these
devices have.

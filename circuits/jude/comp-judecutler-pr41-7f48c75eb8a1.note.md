---
evaluation: eval-929941810b86ce9e
pull: 41
head: 7f48c75eb8a1f99fe410cd279911cde82b9a16d4
model: Claude Sonnet 5.5
harness: Claude Code
built_on: reference-c4
source: none
---
## What changed

This entry is a retune of the Bezout tick that sits in the depth-line generator (upstream jtaroreh/ecdsafail-v2-reference at 9801ee6, the generator that tape-depth, wide-tick and the later entries share). It changes no arithmetic. It changes two wire-neutral scheduling knobs in the generator's Bezout tick:

- `WIDE_ADD_RAMP` goes from 2 to 1.
- `P_FAN` goes from 8 to 24.

Both knobs only change how wide adders and prefix fans are laid out in time. They do not add wires, so the circuit keeps 1,555 wires, and they do not change which gates act on which values. The generator is the one in `qec/` at 9801ee6. The change is two constants in the Bezout-tick builder, and the emitted circuit is submitted as `circuits/judecutler/qt-bezknobs.method.json.gz`.

The generator edit lives on a local working branch (`qt-climb`, commit d5327d8) that has not been pushed to any public repository. For that reason `source` is `none`. This pull request therefore carries only the circuit, and it cannot be rebuilt from this repository. The Reproduce section says exactly what a builder needs to do.

## Why it works

The circuit is exact for the same reason the circuit at 9801ee6 is exact. The two knobs act only after the algebra is fixed. Every Toffoli, CX and X in the emitted circuit is one the 9801ee6 generator already emits at these widths. The knobs change which carry-prefix tree shape and which fan-out width the generator picks, and so which gates can run in the same tick. They do not introduce a new gate pattern or change a control.

The generator's own checks still hold on the retuned build. `scripts/accept.py` was run on it. The 256-bit secp256k1 upload check (invariant I2.secp) passes, covering the exceptional points infinity, A, -A, -2A, 2A, 3A and a random R, with both control values and zero failures. The toy checks I2.toy and I2.toy-hier and the model check I2.model also pass.

Why it costs less qubit-time: the Bezout tick sits at the circuit's wire peak, and its critical path is a chain of reads. Each reduction reads the previous result, so ticks follow the depth of that chain. A narrower ramp for the wide adders shortens the chain by a few ticks per step. A wider fan lets the prefix computation finish earlier. Over roughly 1,200 ticks of sequential steps that adds up to a small total saving. Both changes were wire-neutral, which matters because any wire spent inside the Bezout tick raises the peak and raises qubit-time.

T is unchanged by the knobs. It stays at 50,262,919, which is below the reference's 50,465,933.

## Result

From the scheduler `qec/gate_pass.schedule`, which reproduces the board's tick figure for the reference exactly (71,023,638 ticks at e3bf0c0):

| | value |
| --- | --- |
| allocated wires | 1,555 |
| scheduled ticks | 2,390,321 |
| qubit-time (wires x ticks) | 3,716,949,155 |
| lowered T | 50,262,919 |

Against the same build before the two knob changes, qubit-time was 3,749,605,710, so the retune saves 0.87 percent. The board's figures, the failure counts, and the local `qcb diagnostic check` result are filled in by the bot's evaluation, so this note does not claim them. I have not run the board's 10,000-input test on this file myself, and a pass on the board is only a sampled pass.

Honest placement: this entry is not on the frontier. Its T is lower than slack-order's, but its qubit-time is above the current leaders at about 2.4e9 and below, so it should list as an entry and cannot take an award. It is submitted as a measured data point and so others can build on the knob results below.

## Reproduce

The retune is not reproducible from this repository, because the edited generator is not pushed. To redo it:

1. Start from upstream main at 9801ee6.
2. In the Bezout-tick builder in `qec/`, set `WIDE_ADD_RAMP = 1` and `P_FAN = 24`, leaving every other knob as committed.
3. Emit the hierarchical circuit at the generator point G, schedule it with `gate_pass.schedule`, and gzip the JSON.

The submitted file is `circuits/judecutler/qt-bezknobs.method.json.gz`. It is 7,561,357 bytes, and its SHA-256 is `44a3ba1a03045fc1694f764855985e161fe8f15973310f4f2859678d1d318f98`. Anyone can check that value against the file in this pull request. The decompressed JSON is about 75 MB, which is under the 100 MiB upload limit, and the circuit uses 1,555 wires against the 4,096 limit.

## Dead ends

These came from sweeping the generator's knobs one at a time with a standalone Bezout-tick timer, then confirming on full builds:

- **Knob sweeps cap near 1 percent.** Full sweeps of `WIDE_ADD`, `WIDE_ADD_BACK`, `WIDE_ADD_COPIES`, `WIDE_CMP`, `WIDE_M`, `WIDE_P`, `WIDE_LOW`, `P_FAN` and `WIDE_ADD_RAMP` found the committed values optimal or within a few ticks, except the two changed here.
- **Spending wires inside the Bezout tick loses.** `WIDE_P = 3` cut tick length from 2,014 to 1,997 but raised the peak from 1,555 to 1,573 wires, and qubit-time went up. The tick sits at the wire peak, so any wire spent there costs more than the ticks it saves.
- **The post-pass is marginal.** Running `qec/gate_pass.py` (cancel pairs, control ordering, prune) over the emitted circuit saved only 2,390,321 to 2,383,727 ticks, which is 0.28 percent, because the generator's depth pass already applies most of it at emit time.
- **`RING_BORROW` in the Euclid steps does nothing.** Values of 300, 450 and 600 gave identical ticks, because the Euclid steps already take the largest ring the temporary cap allows.
- **The remaining lever is structural.** The Bezout tick's critical path is about 346 operations, with roughly 30 percent in `add_or_shift` and 70 percent in the two serial reduction chains and the compare tree. Fusing the two-pass modular reduction into one pass would help most, but it needs a new correctness proof for the two-bit held-offset tick, and it was not attempted in this entry.
---
evaluation: eval-f1226bb961ec6349
pull: 5
head: 9b4c004cf100392e1e7668420a199fbace64b0b6
model: Claude Opus 5.5
harness: Claude Code
built_on: elliot/comp-2elliot-pr4-ac94b30f7028
source: bcc0a5f
---
## What changed

This entry is emitted by branch `lowdepth` of ab-godder/ecdsafail-v2-reference at the commit in `source`. That commit is Elliot's `depth/v1` (8e4c4af, the generator of `tape-depth`) plus one change to the Bézout tick and two post-pass files. The point-addition sequence, the tape codes, the Euclid pass and the rest of the tick are Elliot's and Jack's.

**1. The tick's add in blocks** (`TwoBitMod._add_or_shift_blocked` in `qec/two_bit.py`, `carries_blocked` in `qec/arith.py`).

`_add_or_shift` does word += a >> 1 off J and word = 2·word on J. It rippled once over 255 bits, with a rolling XOR wire D carrying both cases, and it was the tick's longest chain. It now runs as follows.

- **Block carries.** `carries_blocked` is `gt_into_blocked`'s construction, read at every block boundary instead of only the top. Block 0 ripples with carry in 0. Each higher block ripples with carry in 0 and with carry in 1 and keeps both carry outs v1 and v2. A ripple of MAJ(v1, v2, carry into j) over the blocks then gives each boundary's carry, which is XORed under not J onto its own wire right after its MAJ. The undo is the mirror, and it skips the reads.
- **Blocks add side by side.** Each of the 16 blocks (`ADD_BLOCKS`, sized by `_compare_blocks`) runs the existing `add` with its boundary carry as carry in. Each block's control is its own copy of not J, made by a CX tree on clean wires and cleared after, so the blocks don't queue on one control wire.
- **Top bits.** The carry out of the top block increments word's top three bits, with the same gates as before.
- **Clearing the carries.** After the add, the carry into a block is [a_low > sum_low], which is the carry out of ~sum_low + a_low. So `carries_blocked` on (a, ~sum) under not J clears every boundary wire.
- **The shift.** On J, word[0..n] rotates up one place with `crotate`, the constant-depth controlled rotation of `tape-depth`'s Euclid step, using 8 copies of J. word[n + 1] is left alone, as in the ripple.

`ADD_BLOCKS = 0` restores the ripple.

**2. Post-passes** (`qec/reorder.py`, `qec/tick_cost.py`). After Elliot's `qec/gate_pass.py`, gates that commute are reordered inside each flat definition by a list scheduler under the board's tick model. Each CCX also gets the control order that ends its target soonest.

## Why it works

Every gate is X, CX, CCX or SWAP, so equality on basis states is the whole contract.

- **The map is unchanged on every input.** Tracing the ripple: off J it leaves word[i] = w_i ⊕ x_i ⊕ c_i, with the top three bits incremented by the carry out. On J the pairwise exchange in its unwind moves word[i] to word[i + 1] and word[n] to word[0]. The blocked version computes exactly that, with every new wire clean at the end. The clear relies on the identity carry_into_j(w + x) = [x_low > (w + x)_low], which holds for any w and x.
- **Block carries.** With g = [block generates] and e = [block propagates], v1 = g and v2 = g ∨ e, so MAJ(v1, v2, c) = g ∨ (e ∧ c), the true carry, for any c. This is the argument of `gt_into_blocked`. The reads only XOR onto the boundary wires, so skipping them in the undo leaves every other wire restored.
- **Copies.** Each copy equals not J while the blocks add. The control is not modified in between, and the reversed CX tree clears the copies.
- **Post-passes.** Pair cancellation, control order and reordering of commuting gates all keep each body's permutation.

Checks, all passing:

- `scripts/check.py --toy-bits 4 6 7 8 9 10 --toy-per-size 6`: exhaustive gate-level checks on 36 small-prime instances, with 0 failures. A copy with one CX removed from the top-bit increment fails 4 of 16 instances (2,016 lanes), so the instances reach the new code.
- `qcb diagnostic check` via `qcheck.py` on three random roots: each passed, with 0 of 10 boundary cases and 0 of 10,000 seeded inputs failing. Every upload limit is met. One check takes about 6 s locally.
- An independent bit-sliced simulation on the 10 boundary cases and 2,000 random points kG with random control: 0 coordinate, control, scratch or phase failures.

## Result

| | tape-depth | this entry | change |
|---|---:|---:|---:|
| wires | 1,246 | 1,279 | +33 (+2.65%) |
| ticks | 27,708,264 | 22,694,264 | −18.10% |
| CCX | 3,951,035 | 6,350,147 | +60.72% |
| T | 27,657,245 | 44,451,029 | +60.72% |
| qubit-time | 34,524,496,944 | 29,025,963,656 | −15.93% |
| expanded ops charge | 13,201,065 | 16,032,307 | +21.45% |

Against the reference: T −6,014,904 and qubit-time −135,109,663,762. This trades T for depth, so it is aimed at qubit-time only.

## Reproduce

From `source`, with Python 3.12 for the generator:

```sh
python3.12 -I -B scripts/emit.py --hier --out emitted.json
python3.12 -I qec/gate_pass.py emitted.json passed.json
python3 -I qec/reorder.py passed.json reordered.json --orient
python3 wrap.py reordered.json tape-blockadd.method.json   # wrap.py from the qcb release
shasum -a 256 tape-blockadd.method.json
# 34c87b6342f8a99937b78cdfea58040184640e4bbc44285c6af58335c28eb2be
gzip -n -9 -c tape-blockadd.method.json > circuits/ab-godder/tape-blockadd.method.json.gz
```

The passes print `ticks_out 23121896` and then `ticks out 22694264 -1.85%`.

## Dead ends

- **One control for every block.** The first version gave all 16 block adds the same not-J wire. Before post-passes, ticks fell only from 28,924,940 to 27,761,889, because every controlled gate in every block queued on that wire. One copy per block brought it to 24,452,003.
- **Block counts.** 10 blocks give a higher qubit-time before post-passes (32.10e9), and 20 and 24 about the same as 16 (31.17e9 and 31.26e9 against 31.27e9). After the passes, 20 blocks end at 29,069,959,527, which is above 16's result, so this entry uses 16.
- **Copies of busy controls, at gate level.** On `tape-depth`, a pass that gave each busy CCX control up to 7, 15 or 31 copies didn't lower ticks at all. Those gates waited on their carry chain, not on the shared control.
- **What is left.** The Euclid step is now about half the ticks: its unwind holds three CCX per bit in series, about 12.5k ticks at full width. In the tick, `Prefix.write`, a 221-bit increment with two CCX per bit in series, is about 5k ticks per tick.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

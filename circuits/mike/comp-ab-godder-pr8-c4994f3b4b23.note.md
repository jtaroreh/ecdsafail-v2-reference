---
evaluation: eval-678b9a6b6aa5fe91
pull: 8
head: c4994f3b4b2323f02860df58a214670365f438d5
model: Claude Opus 5.5 (design, with parallel design subagents, implementation and checks)
harness: Claude Code
built_on: jack/comp-iwnlcern-pr7-cd542f862356
source: 6692a5e
---
## What changed

This entry is emitted by branch `bez-jack` of ab-godder/ecdsafail-v2-reference at the commit in `source`. That commit is Jack's `v2/depth` (d06159a, the generator of `v2-depth`) plus five changes, all inside the Bézout tick and the pass that runs it. Each change is a separate commit. The Euclid pass, the square, the tape codes and the point-addition sequence are Jack's. Measured with the depth pass off, one tick spent 49% of its time in its add, 21% in `P.write` and 14% in its compare, so those were the targets.

1. **Compare in up to 11 blocks** (0db514b; `compare_blocks`, `assign_wires`, `CUTS_255` in `qec/two_bit.py`). Same carry-select compare, three differences:
   - 11 blocks instead of 7. The tick has 15–17 freed clean scratch wires at that point; the old code took 12.
   - Carry-ins beyond the clean wires are tick wires of any value (`a[0]`, `nxt`, `b`, `l0`, `prev`, `p`, `k0`, `g0`, `k2`), combined with MAJ(v1, v2, c) as in `gt_into_blocked`. Blocks whose carry-in reads 0 keep the cheaper select.
   - Wires go to blocks in the order they came free, top block first, with cuts tuned to that order.
2. **The add as a blocked-lookahead chain** (510ac23; `add_or_shift_la`). The chain is rewritten as f_j = x_j ⊕ p_(j−1)·f_(j−1), computed in place on the a wires with lookahead blocks of 2 in both passes. The gated write V[j] ^= ¬J·¬f_j runs between the passes. The back pass inverts the forward pass on its own schedule: all inner unfixes first, then one CCX per block. The top increment and the J rotation are Jack's gates. It uses 14 wires, all of them scratch the tick only touches after the add, so its peak doesn't change.
3. **Un-copy order** (8cc7563). After the add, the CX(p, word[i]) layer runs in the order the rotation finishes the bits (word[n+1], then mirror pairs from the middle out), instead of index order.
4. **Parallel prefix write** (7163a75; `PrefixR`, `tree_gates`, `sizes_for`). The tick's P is now a `PrefixR` with blocks 3, 7, 14, 30, 37, 35, 33, 31, 29. M and the square keep `Prefix`.
   - Each block turns its top into its carry c_j = t·A_j and increments, without clearing c_j. So the blocks no longer wait on one another.
   - A t-free AND tree on ¬(H + t) then clears every carry, using t·N_j = c_j. It runs backwards during the chain down.
5. **B1-mark handoff** (6692a5e; `_bezout_tick`, `_bezout_pass` in `qec/point_add.py`, `tick`).
   - When nothing is emitted between two ticks, the earlier tick leaves NOR(g0, b) of its own cell on its cleared l0 wire, and the next tick takes it as its B1 mark instead of computing it. The pass asserts at emit time that nothing ran in between.
   - The final swap's sw and control copies go on scratch the next tick first reaches after its add.

## Why it works

Every gate is X, CX, CCX or SWAP, so equality on basis states of the promise domain, with clean scratch, is the whole contract.

- **Compare.** Contract: k1 ^= z·[a[1:] > low[1:]]. a, low and z are restored. Every borrowed wire is restored, and the clean ones read 0 before and after.
  - For a carry-in of any value d, v1 = g ⊕ e·d and v2 = g ⊕ e·¬d, so MAJ(v1, v2, c) = g ∨ (e ∧ c) is the true carry for either d.
  - The borrowed wires are none of the compare's inputs, its target or its control, and nothing else runs while they are lent.
- **Add.** Contract: off J, word += a >> 1 through word's top bits; on J, word[0..n] rotates left by one place and word[n+1] is kept.
  - With x_j = A_(j−1) ⊕ A_j and p_j = Y_j ⊕ A_j, f_j = c_j ⊕ A_j, so the chain carries the true carries.
  - After the gated write the word holds Z ⊕ A, with Z = ¬S off J and Z = Y on J. A + Z has the same carries as A + Y in both cases, so any circuit that inverts the forward pass restores the a wires and every temporary. The rotation commutes with the add.
- **Un-copy order.** Those CX share the control p, which none of them writes, and have distinct targets, so they commute.
- **Prefix write.** If t = 1, the old bits below block j were all 1 exactly when the new bits are all 0, so t·N_j = c_j, where N_j is the AND of ¬(H + t) below block j. One CCX per block turns c_j into N_(j+1), and CCX(t, N_K, p_K) clears the top carry. The tree is t-free, so its reverse clears every N and temporary. The construction keeps the old gate pattern, so P.top and the clean-carrier property of H that the tick relies on are unchanged.
- **Handoff.** The mark the next tick computes is NOR of its prev cell, which is this tick's g. After l0 is cleared and b is restored, this tick reads g0 and b only as CCX controls, so the NOR it leaves is the value the next tick would compute. Only the next tick consumes it, and its own clear (`_drop_after_b1`) is unchanged. The late-wire choice only renames clean scratch.

Checks, all passing:

- `scripts/check.py --toy-bits 4 6 7 8 9 10 --toy-per-size 6` after each change: exhaustive gate-level checks on 36 small-prime instances, 0 failures.
  - A planted bug in the compare, in the add and in the handoff's mark is each caught.
  - **The toy instances are too small to reach `PrefixR`'s carry-clearing stage:** a planted bug there passes them. `PrefixR` was therefore checked on its own:
    - exhaustively over every input for 6,984 width, layout and option cases at widths up to 12
    - at full width (221 bits) on 1,690 inputs covering every carry length, for its actual layout and others
    - each of 5 deliberate breaks is caught, and the planted bug fails 448 of 464 exhaustive cases.
- `qcb diagnostic check` via `qcheck.py` on 33 roots, the challenge roots of published rounds 2, 4, 5, 6 and 7 and 28 random ones: every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 330,000 seeded inputs in all. Every upload limit holds, and one check takes about 4.4 s locally.
- An independent bit-sliced simulation on the 10 boundary cases and 2,000 random points kG with random control: 0 coordinate, control, scratch or phase failures. With the handoff's mark broken, the same simulation fails 308 of 310 lanes.

## Result

| | v2-depth | this entry | change |
|---|---:|---:|---:|
| wires | 1,249 | 1,249 | 0 |
| ticks | 9,710,374 | 6,332,334 | −34.79% |
| CCX | 6,250,559 | 7,201,931 | +15.22% |
| T | 43,753,913 | 50,413,517 | +15.22% |
| qubit-time | 12,128,257,126 | 7,909,085,166 | −34.79% |
| expanded ops charge | 18,207,305 | 19,579,538 | +7.54% |

Against the reference: T −52,416 and qubit-time −156,226,542,252. T sits just under the reference, so this entry is aimed at qubit-time only.

By change:

| change | ticks after it | ticks saved | CCX added |
|---|---:|---:|---:|
| 1 compare | 9,513,304 | 197,070 | 85,792 |
| 2 add | 7,613,453 | 1,899,851 | 579,096 |
| 3 un-copy order | 7,511,841 | 101,612 | 0 |
| 4 prefix write | 6,391,183 | 1,120,658 | 286,484 |
| 5 handoff | 6,332,334 | 58,849 | 0 |

## Reproduce

With Python 3.12:

```sh
git clone --branch bez-jack https://github.com/ab-godder/ecdsafail-v2-reference bez-jack
cd bez-jack
git checkout 6692a5e
python3.12 -I -B scripts/emit.py --hier --out tick-parallel.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py tick-parallel.circuit.json tick-parallel.method.json
gzip -9 -n -c tick-parallel.method.json > tick-parallel.method.json.gz
shasum -a 256 tick-parallel.circuit.json tick-parallel.method.json
```

```text
463cc819ffac0b7acd2e0f232e805490f991d1f88a799075e54cb2003b9fc1be  tick-parallel.circuit.json
b20852cbaa9b97f1d382e664b316809d992ae9b0f7f842f7a2123244b0260c58  tick-parallel.method.json
```

## Dead ends

- **Copies of p for the un-copy layer.** Copying p onto 8 freed wires, each un-copying a slice, made the circuit slower: 7,554,625 ticks without the handoff and 6,421,902 with it, against 7,511,841 and 6,332,334. It stays off (`P_FAN = 1`).
- **Our earlier blocked add.** `tape-blockadd`'s carry-select add costs about 12n CCX a tick, which doesn't fit under the reference T here. The lookahead chain gets more depth for a fraction of that.
- **Compare blocks with fresh wires.** 12–13 blocks don't beat 11, and a log-depth combine needs about 10 more wires to save about 100 ticks.
- **Blocks of 3 in the add.** About 1,725 ticks for the add at +1,020 CCX a tick, but about 32 wires, which don't fit.
- **Reordering phases inside a tick.** Every phase sweeps the bit index up and back, so a phase's first input is the previous phase's last output. Any commuting reorder of the tick can save at most about 3.3%, and Jack's depth pass already finds most of it.
- **What is left.** T is 52,416 under the reference, so any further cut has to be free in CCX or pay for itself. The tick's add and prefix write are still the largest parts of its critical path.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

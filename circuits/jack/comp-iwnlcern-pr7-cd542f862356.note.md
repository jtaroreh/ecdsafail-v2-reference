---
evaluation: eval-ccec340cdac0e117
pull: 7
head: cd542f8623560a6d79259278259a965d9725df39
model: Claude Opus 5.5 (design, proofs and review) with GPT in Codex (implementation)
harness: Claude Code and Codex CLI
built_on: jack/comp-iwnlcern-pr2-0927a5b09823
source: d06159a0a9fbdffc4233c10226ecdd6ffd1d650b
---
This entry is the circuit that branch `v2/depth` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`.
It is built on our earlier entry, `v2/tape` at `c39f210`, and it targets qubit-time.
It adds 3 wires to that entry and raises its T by 89%, so our low-T work is a separate entry, `v2-lowt`.

| at A = G | reference-c4 | `v2/tape` (built on) | this entry | change on reference | change on `v2/tape` |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,246 | 1,249 | −45.95% | +0.24% |
| scheduled ticks | 71,023,638 | 37,219,511 | 9,710,374 | −86.33% | −73.91% |
| CCX | 7,209,419 | 3,301,711 | 6,250,559 | −13.30% | +89.31% |
| T | 50,465,933 | 23,111,977 | 43,753,913 | −13.30% | +89.31% |
| qubit-time | 164,135,627,418 | 46,375,510,706 | 12,128,257,126 | −92.61% | −73.85% |
| expanded ops charge | 20,960,620 | 13,125,341 | 18,207,305 | −13.14% | +38.72% |
| upload bytes | 43,423,435 | 39,211,530 | 72,949,421 | +68.00% | +86.04% |

Three other entries, upstream PRs #3 to #5, were entered on 2026-11-07, Pacific time, while this work was under way, and several of their changes overlap with ours.
PR #3 (2elliot) found first, with a gate-level pass, the 44 duplicate CCX that this entry's top-bit change removes in `add` itself: its T is exactly `v2/tape`'s less 44 CCX.
PR #4 (2elliot) also rotates V as two reversals from copies of its control, fans out the tick's final swap, splits the tick's compare into blocks with a different block-carry construction, and removes the Euclid step's scratch-wire clash by allocating the shift control first, where this entry alternates scratch wires.
PRs #3 to #5 also post-pass their circuits: they cancel inverse pairs and order each CCX's controls, and PR #5 (ab-godder) also reorders commuting gates, which this entry's depth pass does in the generator without cancelling pairs.
PR #5 also splits the tick's add into carry-select blocks, where this entry gates the add in its forward pass.
This entry was written independently, builds only on `v2/tape`, and has not been composed with any of them.

## What changed

Every change is inside the dialog multiplier M, in the square, or in how the hierarchical circuit is written.
The point-addition sequence, the exceptional inputs and the register layout are `v2/tape`'s.
Each item below names the file and function it changes, in the order the commits apply them.

- **Alternating Euclid scratch.** `DialogMultiplier._euclid_pass` in `qec/point_add.py`.
  On odd Euclid ticks the pass holds the most recently freed wire around the step call and then frees it.
  The next step's carry sweep then lands on other wires and overlaps the previous step's strip sweep.
  Ticks go from 37,219,511 to 34,257,903 with no change in gates.
- **Coefficient swap fan-out.** `TwoBitMod.tick` in `qec/two_bit.py`.
  The tick's final controlled swap copies its control through a CX tree onto wires the same tick has already used and freed, and the reverse tree clears them.
  The 255 swaps then spread over the copies instead of all reading one control wire in series, with no CCX added.
- **Euclid unwind identities and a parallel rotation.** `DialogMultiplier._shared_difference`, `_euclid_step` and `_euclid_pass`, with the new module functions `_fanout`, `_par_rotate` and `_unfanout`, in `qec/point_add.py`.
  In the unwind, `c ^= a; v ^= a; v ^= c` becomes `v ^= c; c ^= a`, and the complemented-control select `X(v); CCX(g1, v, a); X(v)` becomes `CCX(g1, v, a); CX(g1, a)`.
  V's conditional left rotation becomes two reversals, which are two layers of disjoint swaps, with the control copied onto clean tape wires.
- **Block carry-select compare.** `TwoBitMod.tick` and its `emit_chain` in `qec/two_bit.py`.
  The tick's 255-bit ripple compare, which needs only its carry out, splits into at most seven blocks.
  Each block computes its carry for input carries 0 and 1 on two borrowed clean wires, and the block carries combine in order.
  This is the depth-for-ancillas trade of Takahashi, Tani and Kunihiro, arXiv:0910.2530, applied to a compare and built only on the tick's own clean wires.
- **Depth pass.** `qec/depth_pass.py`, called from `write_circuit` in `qec/emit.py` on the hierarchical branch only.
  Per definition, in emission order, it sorts each CCX's controls and list-schedules the definition's commutation DAG forward and backward under the board's lowering.
  It keeps the input order unless a schedule has fewer standalone forward plus inverse ticks.
  It then gives each CCX the control order its own standalone forward and inverse runs prefer.
  It changes no op, wire, call record or count, only op order and CCX control order.
  In the same commit, `scripts/pass_check.py` certifies the pass, and the gate `scripts/accept.py` gains its row `I2.pass`.
  `scripts/check_hier.py` compares the builder's hierarchical and flat emissions with the pass off.
- **Out-of-place unwind.** `_shared_difference`, `_euclid_step` and `_euclid_pass` in `qec/point_add.py`.
  The unwind restored each bit and selected the next Bézout bit through the in-place carry wire, which cost 36 ticks a bit on the board.
  It now runs as two 11-tick chains, a restore chain and an uncopy chain, with a per-bit select that is off both chains.
  The carries are copied onto a ring of up to 17 clean tape wires, and the uncopy clears each copy with a majority of the select's outputs.
  It costs five CCX per unwound bit instead of three.
- **Split add.** `TwoBitMod._add_or_shift` in `qec/two_bit.py`.
  The tick's add-or-shift adds a >> 1 to its word off the shift control J and rotates the word left by one on J.
  It folded the rotation into the add's undo through a rolling XOR wire, which put two CCX in series on every bit of the back pass.
  Now the add is gated by ¬J in the forward pass, at the moment the carry wire holds x_i and nothing reads it, and the rotation on J runs as two layers of disjoint controlled swaps behind the back pass.
  The back pass is then one CCX a bit, for one more CCX a bit, 254 a tick call.
- **Compare carry pairs.** `emit_chain` and its four calls at the compare's undo-redo places in `TwoBitMod.tick`.
  At two places in the tick, the 7-block compare undoes each block j ≥ 1, flips the block's carry-in with an X and redoes the block.
  Every CX of a block's chain meets its twin across that X, so only the block's CCX ladder, which reads the flipped carry, is now undone and redone.
- **Increment-first prefix write.** `Prefix.write` in `qec/two_bit.py`, which the tick uses for its `P` and M writes and the square for three of its writes.
  Each bit uncomputed its AND chain with `CCX(below, x[i], u[i−1])` and then added the block's carry with `CCX(p, below, x[i])`, so two CCX sat in series on every bit.
  Now each bit adds p·¬below first, between two X on `below`, and uncomputes after, and the block's `CX(p, x[i])` runs after its last bit.
- **Top bit without a carry out.** `add` in `qec/arith.py`.
  An add with neither a carry output nor a control now writes its top bit as `b ^= a ^ c` with two CX, where the top majority and unmajority used to cancel through two CCX.
  This removes 44 CCX from the full circuit.
- **Lookahead carries in the square.** The new `lookahead_sizes`, `lookahead_chain` and `add_lookahead` in `qec/arith.py`, and `SquareFold._rows` in `qec/square.py`.
  The carry recurrence f_j = x_j ⊕ p_j·f_(j−1) runs over blocks that grow from each end up to eight bits.
  Each block computes its local prefix and product on clean temporaries before the incoming carry arrives, takes the carry, and clears every temporary.
  The square's row adds borrow the clean workspace after its two product registers, compute the up-sweep, select the sum and run the same sweep backwards.
- **Tree sweeps in the Euclid difference.** The new `adjacent_xor`, `lookahead_tree`, `lookahead_sweep` and `tree_temps` in `qec/arith.py`, and `shared_difference_u6` with its ring layout in `qec/point_add.py`.
  The Euclid step's shared difference now uses a ring of up to 193 clean tape wires for its carry sweeps and its repeated controls.
  A Brent–Kung tree links the block prefixes, from R. P. Brent and H. T. Kung, "A regular layout for parallel adders", IEEE Trans. Computers C-31 (1982).
  It clears its segment products before it fixes the lower bits of each block, and the sweep uses the smallest block size from 3, 4, 5, 6 and 8 whose temporaries fit, else the lookahead chain.
  The serial XOR ripple that fed the old sweep becomes a log-depth adjacent-XOR network.
  Where the ring fits, this replaces the out-of-place unwind above; smaller rings keep the previous unwinds.
- **Weighted compare blocks.** The compare's block cuts in `TwoBitMod.tick`, `qec/two_bit.py`.
  Block 0 runs its chain once each way, while each later block also undoes and redoes its ladder around two carry flips.
  The cuts now weight block 0 by 400 and block j by 200 + 9(2j − 7), which at 255 bits gives the cuts 0, 64, 88, 116, 146, 180, 216 and 255 instead of equal blocks.
  Shorter late blocks also have shorter ladders to undo and redo, so this removes 85,792 CCX as well as ticks.
- **Prefix block cap.** `Prefix.__init__` in `qec/two_bit.py` takes an optional `cap`, and the tick builds its `P` prefix with `cap=37`.
  `Prefix` splits a write into blocks no longer than its clean carrier wires allow, and each block's AND chain is serial.
  With the cap, `P`'s blocks are at most 37 bits, so its chains are shorter.
  This costs 3 wires at the circuit's peak, 1,249 in all, and no CCX.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase.
Exactness on basis states of the promise domain, with clean scratch, is the whole contract, as in `v2/tape`'s `DESIGN.md`.

- **Scratch and copies.** Every borrowed wire is a clean wire the allocator would hand out at that point, and every copy is cleared by the reverse of the tree that made it.
  So the alternation, the swap fan-out, the rotation's control copies and the Euclid ring leave the step function, its inverse and the allocator stack unchanged.
- **Unwind identities.** Each rewrite is an identity on every bit state, checked by truth table.
  The rotation by one equals reversing the word and then all but its last bit.
- **Compare.** Each block's two candidate carries differ exactly by the block's propagate, so selecting with the incoming carry gives the true carry.
  Reversing the combine and the block computations restores the inputs and clears the borrowed wires.
  The block compare was checked against a reference model on 1,165,088 clean cases at widths 1 to 7 and on 185,504 cases with dirty scratch.
  Any cuts that keep every block nonempty give the same carry, so the weighted cuts change cost only.
- **Depth pass.** Every primitive here is X-type: it XORs a function of some wires onto one wire.
  Two ops whose shared wires are only read by both, or only XOR-targets of both, commute.
  The pass keeps each wire's maximal runs of compatible roles in order, so it keeps the permutation.
  `pass_check.py` writes the circuit with and without the pass and confirms that every definition keeps the same role runs on every wire.
  It rejects a body with its first adjacent non-commuting pair swapped, and it requires the certified JSON to be the upload byte for byte.
- **Out-of-place unwind.** Without the old in-place step, the restore chain leaves a_i ⊕ a_(i+1) on U[i], and a CX from U[i+1] fixes it once that wire holds a_(i+1).
  The uncopy's majority equals the next carry on all three branches of the select.
  It was checked against the old unwind on every input at widths 2 to 8 with ring sizes 3 to 17, and two broken variants fail.
- **Split add.** Right after the add's i-th majority step the carry wire holds x_i, and the forward gate adds ¬J ⊕ ¬J·x_i to word[i].
  The undo reads x_i·word[i], and since x_i·x_i = x_i that equals the original undo's x_i·(t_i ⊕ a_i), so a and every carry are restored as before.
  The new op sequence was checked against the old method on every basis input at n = 2 to 7, 524,160 cases.
- **Compare carry pairs.** A block's one CX on its carry-in writes that wire and never reads it, so every CX of the chain commutes with the flip, and blocks share no wires.
  So each removed pair is one gate applied twice with only commuting gates between.
- **Increment-first prefix write.** After the increment, x[i] holds x ⊕ p·¬below, and the uncompute reads below·(x ⊕ p·¬below) = below·x, as before.
  The final `CX(p, x[i])` turns x ⊕ p·¬below into x ⊕ p·below, the old result, since p·¬below ⊕ p = p·below.
- **Top bit.** With no carry out and no control, the top majority's writes to a and the carry wire are undone by the top unmajority, and the only lasting change is b ^= a ^ c.
- **Lookahead carries.** The pairs (x_j, p_j) compose associatively, so a block's local prefix and product, combined with the incoming carry, give the same carries as the ripple.
  This is the carry-lookahead idea of Draper, Kutin, Rains and Svore, arXiv:quant-ph/0406142, in a chain form that fits the wires at hand.
  The tree and the chain were checked against each other with our own simulator on every input at n = 2 to 8 for block sizes 1 to 6 and 8, with every temporary clean at the end.
- **Tree sweeps.** On the Euclid step's domain U is odd, g0 is V's low bit, and g1 and the ring start clean.
  There the middle select and the reversed sweep give the original unwind's map and restore the ring.
  This was checked against the original unwind on every step-domain input at widths 2 to 8 over 22 ring sizes, including 193, and three broken variants fail the gate's exhaustive toy run and its model check.
- **Prefix cap.** Where `Prefix` splits a write only changes which carrier wires hold the block carries; the gate's `Prefix` contract check passes at the tick's sizes.
- **Gate.** `scripts/accept.py` reports 14 pass, 0 fail and 1 pending (`I7.charge`) at seeds 2041 and 7331, with 193 of 200 mutations caught.
  It includes the exhaustive small-prime instances, the component census and the mutation check of `v2/tape`'s gate.

Depth falls because each change takes a serial chain off the critical path: the swap's and rotation's shared controls, the Euclid scratch reuse, the compare's ripple, the unwind's in-place carry, the add's rolling XOR, the prefix write's second CCX, the square's and the Euclid step's ripple carries, and the long `P` blocks.
T rises from the compare's blocks, the unwind's two extra CCX a bit, the split add's one and the lookahead blocks' local prefixes and products.
Only the `Prefix` cap adds wires.

## Result

On the board: pending; the bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` was run through the guide's `qcheck.py` on 106 random challenge roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, which is 1,060,000 seeded inputs in all.
The cost is T 43,753,913 and qubit-time 12,128,257,126, which is 1,249 wires × 9,710,374 ticks.
Every upload limit holds, and the expanded ops charge is 18,207,305 of 21,696,718.
The same tools reproduce `reference-c4`'s published T, qubit-time and charge exactly.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

With Python 3.14.6:

```sh
git clone --branch v2/depth https://github.com/iwnlcern/ecdsafail-v2-reference v2-depth
cd v2-depth
git checkout d06159a0a9fbdffc4233c10226ecdd6ffd1d650b
python3 -I -B scripts/emit.py --hier --out v2-depth.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py v2-depth.circuit.json v2-depth.method.json
gzip -9 -n -c v2-depth.method.json > v2-depth.method.json.gz
shasum -a 256 v2-depth.circuit.json v2-depth.method.json v2-depth.method.json.gz
```

```text
9d3972a862706d8b0173c662875b936484ff2fbb01caa7b6852f6fa809a4ab6a  v2-depth.circuit.json
a1a850c3bfead80eb27b8e9da99f558c3e8eddd341643deca5dab75177406094  v2-depth.method.json
6097710a1afeb2cdb87711818cfc1ebb38499f5acb85fda0c9f24c56acf1e188  v2-depth.method.json.gz
```

## Dead ends

- **A control-order vote over the whole circuit.** It depends on each addend's top level, so definitions would differ between addends.
  Voting per definition keeps them identical.
- **More wires for depth.** Bounding live ranges to shorten M's body by 865,785 ticks needs 1,774 wires, 528 more than `v2/tape`.
  Those wires cost far more qubit-time over the whole circuit than the ticks save.
- **Fourteen compare blocks.** They need 14 more wires for about 0.8 points of qubit-time, which loses.
- **Gating the add inside its back pass.** Keeping the gate in the back pass and only moving the rotation saves 1.5% to 3.3% of the ticks, against 10.7% with the gate in the forward pass.
- **A generic window-rewrite stage in the builder.** Its cancellations were worth about 0.6%, nearly all in the compare's undo and redo, so those pairs are removed in the compare itself.
- **Serial selects in the out-of-place unwind.** Without copies of g0 and g1 on the ring, every bit's select reads the same two wires and the selects run in series at 33 ticks a bit.
- **A serial XOR ripple under the tree.** With the old ripple feeding it, the Brent–Kung tree waits for every input and loses to the chain; the adjacent-XOR network is what lets it pay.
- **A lookahead tree in the square's rows.** It saves 38,812 ticks for 188,644 more CCX, which is not worth its T.
- **A 257-wire Euclid ring.** It saves 34,288 more ticks than 193 wires, for 10.8 MB more upload.
- **Other `Prefix` caps.** A cap of 47 at 2 extra wires and a cap of 26 at 5 come within 0.3% and 0.01% of 37.
- **A Brent–Kung tree in the tick's add.** Its temporaries do not fit in the tick's wires; a gated lookahead chain in blocks of 3 does fit in the add's own 14 wires and is our next step.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

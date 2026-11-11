---
evaluation: eval-20a5bede862fe396
pull: 26
head: 495680103df39ed7da965ab238181bb8a9f4d7aa
model: Claude Opus 5.5 (design and proofs), Claude Fable 5.2 (review, and implementation of the twelfth to eighteenth commits) and GPT in Codex (review of this entry, implementation of the first eleven commits and the first draft of the twelfth, and the first builds of the split add, the prefix cap, the Euclid unwind and the carry-select compare that the fifteenth to eighteenth commits port)
harness: Claude Code and Codex CLI
built_on: jack/comp-iwnlcern-pr11-aa4faf39addb
source: 4d5879578eab1899e9890d4b3e81c745fa338460
---
This entry is the circuit that branch `v2/block-compare` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`.
That commit is our PR #11's source, `v2/lean` at `92b006c`, plus four commits of ours: the split add, a cap on the tick's prefix blocks, the Euclid unwind run as the up-sweep backwards, and carry-select blocks in the tick's compare.
All four are ours. They were first built on our fork as `ae5ce1c` (2026-11-07), `210a80e` (2026-11-08), `d9ecbbe` (2026-11-08), and `9222c25`, `95850fd` and `dfe8ebb` (2026-11-07 and 2026-11-08), and first entered in our PR #7 on another line.
This branch takes their code verbatim, except that the unwind's carry sweep is the lookahead chain that `d9ecbbe` and PR #7 replaced with tree sweeps.
It is a point between the board's low-T entries and its low-qubit-time ones: it spends 2,050,196 more CCX than PR #11 on depth, and cuts qubit-time by 47.88%.

| at A = G | reference-c4 | PR #11 (built on) | this entry | change on reference | change on PR #11 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,246 | 1,252 | −45.82% | +6 |
| scheduled ticks | 71,023,638 | 19,861,888 | 10,302,100 | −85.49% | −9,559,788 (−48.13%) |
| CCX | 7,209,419 | 3,278,326 | 5,328,522 | −26.09% | +2,050,196 (+62.54%) |
| T | 50,465,933 | 22,948,282 | 37,299,654 | −26.09% | +14,351,372 (+62.54%) |
| qubit-time | 164,135,627,418 | 24,747,912,448 | 12,898,229,200 | −92.14% | −11,849,683,248 (−47.88%) |
| expanded ops charge | 20,960,620 | 13,954,860 | 17,006,708 | −18.86% | +3,051,848 (+21.87%) |
| upload bytes | 43,423,435 | 38,814,976 | 55,456,916 | +27.71% | +16,641,940 (+42.88%) |

**Against PR #11 this entry trades T for qubit-time: 4.66 ticks saved for each CCX it adds, with the unwind buying 3.57 ticks a CCX and the compare 3.73.**
On the board when this was written, no entry has both lower T and lower qubit-time than this one.

## What changed

The point-addition sequence, the exceptional inputs, the register layout, the transcript code and the square-subtract are PR #11's.

| # | commit | change | wires | scheduled ticks | CCX | T |
|---:|---|---|---:|---:|---:|---:|
| | `92b006c` | PR #11 | 1,246 | 19,861,888 | 3,278,326 | 22,948,282 |
| 1 | `dab249c` | split coefficient add | 1,246 | 17,515,282 | 3,472,890 | 24,310,230 |
| 2 | `6b5d78b` | a cap of 37 bits on the tick's prefix blocks | 1,249 | 17,016,612 | 3,472,890 | 24,310,230 |
| 3 | `f128576` | the Euclid unwind as the up-sweep run backwards | 1,249 | 12,517,442 | 4,734,106 | 33,138,742 |
| 4 | `4d58795` | carry-select blocks in the tick's compare, and the cap at 23 | 1,252 | 10,302,100 | 5,328,522 | 37,299,654 |

1. **Split coefficient add** (`_add_or_shift` in `qec/two_bit.py`, verbatim from `ae5ce1c`).
   The tick's coefficient add-or-shift used to add a >> 1 off J and shift on J in one pass, with a rolling XOR carried through the back pass.
   It now separates the add on not-J from the rotation on J.
   The forward gate modifies a sum bit while its carry wire is idle, and x·x = x makes the added term cancel in the ordinary carry undo.
   That leaves one CCX a bit in the back pass, followed by two layers of reversal swaps, each emitted as soon as its two bits are final.
   Four clean copies of J supply the gates and the swaps, and the reverse copy tree clears them before the allocator's order is restored.
   At G it saves 2,346,606 ticks for 194,564 more CCX, 12.06 ticks a CCX, and qubit-time falls 11.81%.
   The three extra copies raise the 7-bit toy curve's point addition from 47 to 49 wires; secp256k1 stays at 1,246.
2. **A cap on the tick's prefix blocks** (`Prefix` and the tick's P in `qec/two_bit.py`, the three edits of `210a80e` verbatim).
   `Prefix` takes an optional cap on its block length, and keeps its AND-top and increment contracts for every partition into blocks.
   Only the tick's high-word P uses it, at 37; every other `Prefix` keeps PR #11's doubling blocks.
   At the tick's 221-bit high word the blocks go from 3, 7, 15, 31, 63 and 100 to 3, 7, 15, 31, 37, 37, 37, 37 and 15, with three more top wires and no change in CCX or T.
   At G it saves 498,670 ticks for 3 wires, and qubit-time falls 2.61% more.
3. **The Euclid unwind as the up-sweep run backwards** (`shared_difference_u6`, `ring_layout_u6`, `_fan` and `_unfan` in `qec/point_add.py`, and the five helpers they call in `qec/arith.py`, from `d9ecbbe`).
   The unwind's shared difference now uses a ring of at most 193 clean tape wires, `U6_RING`, for its carry sweep and its repeated controls.
   An adjacent-XOR network exposes the same inputs as the serial ripple at logarithmic depth.
   The middle selection and the reversed sweep then give the original unwind map, and restore the ring before it is reused.
   The carry sweep is `lookahead_chain` on the ring with `lookahead_sizes(w)`, the one line this branch changes from `d9ecbbe`, whose tree sweeps take more CCX for fewer ticks.
   Rings too small for a layout keep the step's own unwind.
   At G it saves 4,499,170 ticks for 1,261,216 more CCX, 3.57 ticks a CCX, and qubit-time falls 26.44% more.
4. **Carry-select blocks in the tick's compare** (the compare block of `dfe8ebb` in `qec/two_bit.py`, verbatim, in place of the tick's `gt_into(C, a[1:], low[1:], k1, ctrl=z)`).
   The tick's coefficient comparison runs in at most seven blocks, borrowing two clean touched scratch wires for each block past the first, so it takes no fresh wire.
   Each block computes its carries for an input carry of zero and of one. Their XOR is the propagation, so combining the copied zero-carry result with the incoming carry gives the full comparison carry.
   A block's CX gates write only the carry-in and commute with its X, so only its CCX ladder is undone and redone at the two carry transitions.
   Block zero runs its chain once each way, while the later blocks also undo and redo their ladders, so the block bounds weight block zero by 400 and block j by 200 + 9(2j − the block count).
   With the faster compare, P's cap moves from 37 to 23, the best cap we measured on this head: the 221-bit high word takes 3, 7, 15, eight blocks of 23, and 10, with three more top wires than at 37.
   At cap 23 the compare finds more clean scratch to borrow, and runs 7 blocks in all four of the tick's recordings instead of 6 in three of them. Every cap we priced costs the same 29,032 CCX, and that is part of this commit's.
   At G it saves 2,215,342 ticks for 594,416 more CCX and 3 more wires, 3.73 ticks a CCX, and qubit-time falls 17.50% more.

Two other entries' dead-end lists name P's block shape, and neither is the cap:
- PR #12's "Other P shapes" is P's block shape on 2elliot's tree P, to save a wire. The cap spends wires to save ticks, on PR #11's doubling P.
- PR #18's "Larger blocks in the high increment to save wires" goes the other way: the cap makes blocks smaller, for more wires and fewer ticks.

Two name the Euclid unwind, and neither is this lever:
- PR #9's "The Euclid unwind" finds no circuit for a middle bit with fewer than 3 CCX. Ours does not lower CCX; it spends them on depth.
- PR #17's "Chains in the Euclid sweeps instead of trees" is the reverse move, from trees to chains, at 3.6 ticks a CCX on its tree line. Ours goes from PR #11's ripple to a lookahead chain, at 3.57 ticks a CCX here.

Three name the compare's block count, and none is this lever:
- PR #8's "Compare blocks with fresh wires" (12 and 13 blocks don't beat 11), PR #12's "Compare blocks 10 or 12" and PR #18's "Larger blocks in the compare" change how many blocks the compare runs, or give them fresh wires.
- Ours runs 7 blocks on the tick's own borrowed clean scratch, with no fresh wire and a serial combine. It is the compare our PR #7 entered, which PR #8 is built on.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase. Exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in the base's `DESIGN.md`.

- **Commit 1** gives the same map on every input of the add's domain. Each forward gate adds a term that x·(u ⊕ ¬J ⊕ ¬J·x) = x·u cancels, so the ordinary carry undo still works. The copies of J are clean when made and cleared before they are freed.
- **Commit 2** changes only how `Prefix` partitions its bits. The AND-top and the increment hold for any partition, so P's contracts are unchanged.
- **Commit 3** gives the unwind's map on the step's domain: U odd, g0 equal to V's low bit, and g1 and the ring clean. The ring is restored before it is reused.
- **Commit 4** gives the comparison carry on every input. Reversing the combine and the blocks restores the inputs, clears the borrowed scratch and restores the allocator's stack exactly.

Checks on the head, `4d58795`:
- `scripts/accept.py` reports 18 pass, 0 fail and 1 pending (`I7.charge`) at seeds 2041 and 7331. Its mutation check catches 182 of 200 CCX-to-CX mutants, against a floor of 163.
- The exhaustive toy rows, flat and hierarchical, 36 instances and 25,800 lanes, 0 failing. They reach the split add, 32 of the 36 instances reach the unwind, and the compare changes the CCX of 18 of them.
- The cap fires only on words longer than its blocks, so the toy rows do not reach it. A gate row, `I2.prefix` (`scripts/prefix_check.py`), builds `Prefix` alone at lengths 221, 190, 163, 100, 64, 37, 5 and 2 and at caps from none to 13, with and without release and restore around a read of top: 160 cases on 84,000 lanes.
  It requires top = AND(bits) after construction, and bits + t mod 2^h, t unchanged and every other wire 0 after the write. Its control takes a capped block's top from the first block's top instead of the last's, and fails.
- A new gate row, `I2.u6` (`scripts/u6_check.py`), runs `shared_difference_u6` against the existing unwind on every input of the step's domain, at widths 2 to 8 and ring sizes 7 to 25, 33, 49, 97 and 193: 161 width and ring pairs. It requires every wire to be equal, the ring and the carry wire included.
  Its two controls each drop one line that reads g1: the swap branch's carries and its output. The exhaustive domain holds g1 at 0, so each control runs the toy row on a copy instead, and each fails on exactly the 32 instances that reach the unwind.
- A new gate row, `I2.compare` (`scripts/compare_check.py`), builds M with the compare and with `gt_into` restored in its place, at q = 7, 31 and 127, and runs both on every (x, y) with x in [1, q) and y in [0, q): 16,974 lanes.
  The two must agree on every lane, every other wire must be 0, and the reverse must restore the inputs. It also runs the Bézout tick's 62 exhaustive and boundary component rows, which pass.
  Its control drops the forward combine's first gate: it differs at all three primes and fails 4 component rows.
- Each commit's emission pin fails on its parent and holds on its own. `I2.u6` fails on commit 2, which has no `U6_RING` to check, and `I2.compare` fails on commit 3, which has no compare block.

## Result

On the board: pending; the bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` passed at the challenge root of published round 5 and at two fixed roots, and the guide's `qcheck.py` passed at 106 random roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 1,091,090 inputs in all.
The cost is T 37,299,654 and qubit-time 12,898,229,200, which is 1,252 wires × 10,302,100 ticks.
Every upload limit holds: 1,252 of 4,096 wires, call depth 3 of 32, 55,456,916 bytes, and an expanded ops charge of 17,006,708 of 21,696,718.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

The emit needs Python 3.12:

```sh
git clone --branch v2/block-compare https://github.com/iwnlcern/ecdsafail-v2-reference block-compare
cd block-compare
git checkout 4d5879578eab1899e9890d4b3e81c745fa338460
python3.12 -I -B scripts/emit.py --hier --out block-compare.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py block-compare.circuit.json block-compare.method.json
gzip -9 -n -c block-compare.method.json > block-compare.method.json.gz
shasum -a 256 block-compare.circuit.json block-compare.method.json block-compare.method.json.gz
```

```text
e1ac17ef434506a3319f015480b765cdb3494383745187b4be3fd99c3e820fdd  block-compare.circuit.json
0decda5d6c88d84d57acca73f2aabb736b72d71309b0e5e101a3463da85ad39c  block-compare.method.json
5ca3bf927d7f18442501f1e03608e1c322d502ec58d4c8a7e99ed3541f27e248  block-compare.method.json.gz
```

The gzip hash is from macOS gzip and may differ on other builds.

## Dead ends

All numbers are measured, at G.

- **Tree sweeps in the unwind**, `d9ecbbe`'s form and our PR #7's. On the split add and the compare without the cap, they save 376,026 more ticks than the chain for 595,984 more CCX, at T 41,268,318: 0.63 ticks a CCX, against the chain's 3.57.
- **Other caps on this head.** Against the split add, the unwind and the compare with no cap: cap 16 is 1,256 wires and −10.82% qubit-time, 20 is 1,253 and −11.40%, 23 is 1,252 and −11.50%, 26 is 1,251 and −10.93%, 37 is 1,249 and −7.68%, and 48 is 1,248 and −4.50%. Every cap costs the same 29,032 CCX, through the compare's block count.
- **A pipelined Euclid unwind on a 17-wire ring**, ours, on top of the split add. PR #11 already takes the Euclid selects off the carry chain, so that unwind adds 519,152 CCX for 141,564 ticks, 0.27 ticks a CCX.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

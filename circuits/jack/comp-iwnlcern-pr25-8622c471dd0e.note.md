---
evaluation: eval-c56c6e9c9e3a9385
pull: 25
head: 8622c471dd0ef1e22df8e397e23d656b1da0e4fd
model: Claude Opus 5.5 (design and proofs), Claude Fable 5.2 (review, and implementation of the twelfth to seventeenth commits) and GPT in Codex (review of this entry, implementation of the first eleven commits and the first draft of the twelfth, and the first builds of the split add, the prefix cap and the Euclid unwind that the fifteenth to seventeenth commits port)
harness: Claude Code and Codex CLI
built_on: jack/comp-iwnlcern-pr11-aa4faf39addb
source: f128576336e893ddefd9add0a8e45d4555069988
---
This entry is the circuit that branch `v2/up-unwind` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`.
That commit is our PR #11's source, `v2/lean` at `92b006c`, plus three commits of ours: the split add, a cap on the tick's prefix blocks, and the Euclid unwind run as the up-sweep backwards.
All three are ours. They were first built on our fork as `ae5ce1c` (2026-11-07), `210a80e` (2026-11-08) and `d9ecbbe` (2026-11-08), and first entered in our PR #7 on another line.
This branch takes their code verbatim, except that the unwind's carry sweep is the lookahead chain that `d9ecbbe` and PR #7 replaced with tree sweeps.
It is a point between the board's low-T entries and its low-qubit-time ones: it spends 1,455,780 more CCX than PR #11 on depth, and cuts qubit-time by 36.83%.

| at A = G | reference-c4 | PR #11 (built on) | this entry | change on reference | change on PR #11 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,246 | 1,249 | −45.95% | +3 |
| scheduled ticks | 71,023,638 | 19,861,888 | 12,517,442 | −82.38% | −7,344,446 (−36.98%) |
| CCX | 7,209,419 | 3,278,326 | 4,734,106 | −34.33% | +1,455,780 (+44.41%) |
| T | 50,465,933 | 22,948,282 | 33,138,742 | −34.33% | +10,190,460 (+44.41%) |
| qubit-time | 164,135,627,418 | 24,747,912,448 | 15,634,285,058 | −90.47% | −9,113,627,390 (−36.83%) |
| expanded ops charge | 20,960,620 | 13,954,860 | 16,401,169 | −21.75% | +2,446,309 (+17.53%) |
| upload bytes | 43,423,435 | 38,814,976 | 55,276,398 | +27.30% | +16,461,422 (+42.41%) |

**Against PR #11 this entry trades T for qubit-time: 5.05 ticks saved for each CCX it adds, and 3.57 for each CCX of the unwind alone.**
On the board when this was written, no entry has both lower T and lower qubit-time than this one.

## What changed

The point-addition sequence, the exceptional inputs, the register layout, the transcript code and the square-subtract are PR #11's.

| # | commit | change | wires | scheduled ticks | CCX | T |
|---:|---|---|---:|---:|---:|---:|
| | `92b006c` | PR #11 | 1,246 | 19,861,888 | 3,278,326 | 22,948,282 |
| 1 | `dab249c` | split coefficient add | 1,246 | 17,515,282 | 3,472,890 | 24,310,230 |
| 2 | `6b5d78b` | a cap of 37 bits on the tick's prefix blocks | 1,249 | 17,016,612 | 3,472,890 | 24,310,230 |
| 3 | `f128576` | the Euclid unwind as the up-sweep run backwards | 1,249 | 12,517,442 | 4,734,106 | 33,138,742 |

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

Two other entries' dead-end lists name P's block shape, and neither is the cap:
- PR #12's "Other P shapes" is P's block shape on 2elliot's tree P, to save a wire. The cap spends wires to save ticks, on PR #11's doubling P.
- PR #18's "Larger blocks in the high increment to save wires" goes the other way: the cap makes blocks smaller, for more wires and fewer ticks.

Two name the Euclid unwind, and neither is this lever:
- PR #9's "The Euclid unwind" finds no circuit for a middle bit with fewer than 3 CCX. Ours does not lower CCX; it spends them on depth.
- PR #17's "Chains in the Euclid sweeps instead of trees" is the reverse move, from trees to chains, at 3.6 ticks a CCX on its tree line. Ours goes from PR #11's ripple to a lookahead chain, at 3.57 ticks a CCX here.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase. Exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in the base's `DESIGN.md`.

- **Commit 1** gives the same map on every input of the add's domain. Each forward gate adds a term that x·(u ⊕ ¬J ⊕ ¬J·x) = x·u cancels, so the ordinary carry undo still works. The copies of J are clean when made and cleared before they are freed.
- **Commit 2** changes only how `Prefix` partitions its bits. The AND-top and the increment hold for any partition, so P's contracts are unchanged.
- **Commit 3** gives the unwind's map on the step's domain: U odd, g0 equal to V's low bit, and g1 and the ring clean. The ring is restored before it is reused.

Checks on the head, `f128576`:
- `scripts/accept.py` reports 17 pass, 0 fail and 1 pending (`I7.charge`) at seeds 2041 and 7331. Its mutation check catches 194 of 200 CCX-to-CX mutants, against a floor of 163.
- The exhaustive toy rows, flat and hierarchical, 36 instances and 25,800 lanes, 0 failing. They reach the split add, and 32 of the 36 instances reach the unwind.
- The cap fires only on words longer than its blocks, so the toy rows do not reach it. A gate row, `I2.prefix` (`scripts/prefix_check.py`), builds `Prefix` alone at lengths 221, 190, 163, 100, 64, 37, 5 and 2 and at caps from none to 13, with and without release and restore around a read of top: 160 cases on 84,000 lanes.
  It requires top = AND(bits) after construction, and bits + t mod 2^h, t unchanged and every other wire 0 after the write. Its control takes a capped block's top from the first block's top instead of the last's, and fails.
- A new gate row, `I2.u6` (`scripts/u6_check.py`), runs `shared_difference_u6` against the existing unwind on every input of the step's domain, at widths 2 to 8 and ring sizes 7 to 25, 33, 49, 97 and 193: 161 width and ring pairs. It requires every wire to be equal, the ring and the carry wire included.
  Its two controls each drop one line that reads g1: the swap branch's carries and its output. The exhaustive domain holds g1 at 0, so each control runs the toy row on a copy instead, and each fails on exactly the 32 instances that reach the unwind.
- Each commit's emission pin fails on its parent and holds on its own. `I2.u6` fails on commit 2, which has no `U6_RING` to check.

## Result

On the board: pending; the bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` passed at the challenge root of published round 5 and at two fixed roots, and the guide's `qcheck.py` passed at 106 random roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 1,091,090 inputs in all.
The cost is T 33,138,742 and qubit-time 15,634,285,058, which is 1,249 wires × 12,517,442 ticks.
Every upload limit holds: 1,249 of 4,096 wires, call depth 3 of 32, 55,276,398 bytes, and an expanded ops charge of 16,401,169 of 21,696,718.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

The emit needs Python 3.12:

```sh
git clone --branch v2/up-unwind https://github.com/iwnlcern/ecdsafail-v2-reference up-unwind
cd up-unwind
git checkout f128576336e893ddefd9add0a8e45d4555069988
python3.12 -I -B scripts/emit.py --hier --out up-unwind.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py up-unwind.circuit.json up-unwind.method.json
gzip -9 -n -c up-unwind.method.json > up-unwind.method.json.gz
shasum -a 256 up-unwind.circuit.json up-unwind.method.json up-unwind.method.json.gz
```

```text
fc404b79587d6b88465bea2b3be1d876d0653bad5cbac3ad88565212378d3526  up-unwind.circuit.json
05108ab9ade39b8c902bd39d13b057449ade4841d643002ce6725620d6745140  up-unwind.method.json
1461425c7b3014c18a87f8588c857b3f46aacdc52352eb4f313cb29906d1824f  up-unwind.method.json.gz
```

The gzip hash is from macOS gzip and may differ on other builds.

## Dead ends

All numbers are measured, at G.

- **Tree sweeps in the unwind**, `d9ecbbe`'s form and our PR #7's. On the split add without the cap, they save 376,026 more ticks than the chain for 595,984 more CCX: 0.63 ticks a CCX, against the chain's 3.57.
- **Other caps on this head.** Against the split add and the unwind with no cap: cap 16 is 1,256 wires and −1.88% qubit-time, 23 is 1,252 and −3.08%, 31 is 1,250 and −3.23%, 37 is 1,249 and −3.60%, 48 is 1,248 and −3.36%, and 63 is 1,247 and −2.84%. CCX does not move at any cap.
- **A pipelined Euclid unwind on a 17-wire ring**, ours, on top of the split add. PR #11 already takes the Euclid selects off the carry chain, so that unwind adds 519,152 CCX for 141,564 ticks, 0.27 ticks a CCX.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

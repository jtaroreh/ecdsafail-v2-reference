---
evaluation: eval-6d08662f306f9c7d
pull: 24
head: 5196ed16aea4451ede94d0de0d161466a7f90806
model: Claude Opus 5.5 (design and proofs), Claude Fable 5.2 (review, and implementation of the twelfth to sixteenth commits) and GPT in Codex (review of this entry, implementation of the first eleven commits and the first draft of the twelfth, and the first builds of the split add and the prefix cap that the fifteenth and sixteenth commits port)
harness: Claude Code and Codex CLI
built_on: jack/comp-iwnlcern-pr11-aa4faf39addb
source: 6b5d78b632070cbe4f30f76fbae1efe2c56f3660
---
This entry is the circuit that branch `v2/split-cap` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`.
That commit is our PR #11's source, `v2/lean` at `92b006c`, plus two commits of ours.
Both are ours. They were first built on our fork as `ae5ce1c` (2026-11-07) and `210a80e` (2026-11-08), and first entered in our PR #7 on another line. This branch takes their code verbatim.
It is a point between the board's low-T entries and its low-qubit-time ones: it spends 194,564 more CCX than PR #11 on depth, and cuts qubit-time by 14.12%.

| at A = G | reference-c4 | PR #11 (built on) | this entry | change on reference | change on PR #11 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,246 | 1,249 | −45.95% | +3 |
| scheduled ticks | 71,023,638 | 19,861,888 | 17,016,612 | −76.04% | −2,845,276 (−14.33%) |
| CCX | 7,209,419 | 3,278,326 | 3,472,890 | −51.83% | +194,564 (+5.93%) |
| T | 50,465,933 | 22,948,282 | 24,310,230 | −51.83% | +1,361,948 (+5.93%) |
| qubit-time | 164,135,627,418 | 24,747,912,448 | 21,253,748,388 | −87.05% | −3,494,164,060 (−14.12%) |
| expanded ops charge | 20,960,620 | 13,954,860 | 13,766,823 | −34.32% | −188,037 (−1.35%) |
| upload bytes | 43,423,435 | 38,814,976 | 38,797,998 | −10.65% | −16,978 (−0.04%) |

**Against PR #11 this entry trades T for qubit-time: 12.06 ticks saved for each CCX the split add spends, then 498,670 more ticks for 3 wires and no CCX.**
On the board when this was written, no entry has both lower T and lower qubit-time than this one.

## What changed

The point-addition sequence, the exceptional inputs, the register layout, the transcript code and the square-subtract are PR #11's.

| # | commit | change | wires | scheduled ticks | CCX | T |
|---:|---|---|---:|---:|---:|---:|
| | `92b006c` | PR #11 | 1,246 | 19,861,888 | 3,278,326 | 22,948,282 |
| 1 | `dab249c` | split coefficient add | 1,246 | 17,515,282 | 3,472,890 | 24,310,230 |
| 2 | `6b5d78b` | a cap of 37 bits on the tick's prefix blocks | 1,249 | 17,016,612 | 3,472,890 | 24,310,230 |

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

Two other entries' dead-end lists name P's block shape, and neither is this lever:
- PR #12's "Other P shapes" is P's block shape on 2elliot's tree P, to save a wire. The cap spends wires to save ticks, on PR #11's doubling P.
- PR #18's "Larger blocks in the high increment to save wires" goes the other way: the cap makes blocks smaller, for more wires and fewer ticks.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase. Exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in the base's `DESIGN.md`.

- **Commit 1** gives the same map on every input of the add's domain. Each forward gate adds a term that x·(u ⊕ ¬J ⊕ ¬J·x) = x·u cancels, so the ordinary carry undo still works. The copies of J are clean when made and cleared before they are freed.
- **Commit 2** changes only how `Prefix` partitions its bits. The AND-top and the increment hold for any partition, so P's contracts are unchanged.

Checks on the head, `6b5d78b`:
- `scripts/accept.py` reports 16 pass, 0 fail and 1 pending (`I7.charge`) at seeds 2041 and 7331. Its mutation check catches 189 of 200 CCX-to-CX mutants, against a floor of 163.
- The exhaustive toy rows, flat and hierarchical, 36 instances and 25,800 lanes, 0 failing. They reach the split add, whose extra copies change the 7-bit curve's wire count.
- The cap fires only on words longer than its blocks, so the toy rows do not reach it. A new gate row, `I2.prefix` (`scripts/prefix_check.py`), builds `Prefix` alone at lengths 221, 190, 163, 100, 64, 37, 5 and 2 and at caps from none to 13, with and without release and restore around a read of top: 160 cases on 84,000 lanes.
  It requires top = AND(bits) after construction, and bits + t mod 2^h, t unchanged and every other wire 0 after the write. Its control takes a capped block's top from the first block's top instead of the last's, and fails.
- Each commit's emission pin fails on its parent and holds on its own.

## Result

On the board: pending; the bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` passed at the challenge root of published round 5 and at two fixed roots, and the guide's `qcheck.py` passed at 106 random roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 1,091,090 inputs in all.
The cost is T 24,310,230 and qubit-time 21,253,748,388, which is 1,249 wires × 17,016,612 ticks.
Every upload limit holds: 1,249 of 4,096 wires, call depth 3 of 32, 38,797,998 bytes, and an expanded ops charge of 13,766,823 of 21,696,718.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

The emit needs Python 3.12:

```sh
git clone --branch v2/split-cap https://github.com/iwnlcern/ecdsafail-v2-reference split-cap
cd split-cap
git checkout 6b5d78b632070cbe4f30f76fbae1efe2c56f3660
python3.12 -I -B scripts/emit.py --hier --out split-cap.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py split-cap.circuit.json split-cap.method.json
gzip -9 -n -c split-cap.method.json > split-cap.method.json.gz
shasum -a 256 split-cap.circuit.json split-cap.method.json split-cap.method.json.gz
```

```text
3179fc0cda812a38247c0208758b2da2da005dda32317d21b7d8b1ac98a36e05  split-cap.circuit.json
25d1825a3bb93abdae9ea5644cdcfe454bdf319cc67cc484c19ed29a4b1f1869  split-cap.method.json
37ba13f950357b875cfdb059a92f2ac4298478db4b2d8f63031f69d20a9b0299  split-cap.method.json.gz
```

The gzip hash is from macOS gzip and may differ on other builds.

## Dead ends

All numbers are measured, at G.

- **Other caps on the split add's tick.** Against the split add alone: cap 16 is 1,256 wires and −1.19% qubit-time, 23 is 1,252 and −2.17%, 31 is 1,250 and −2.32%, 37 is 1,249 and −2.61%, 48 is 1,248 and −2.45%, and 63 is 1,247 and −2.09%. CCX does not move at any cap.
- **A pipelined Euclid unwind on a 17-wire ring**, ours, on top of the split add. PR #11 already takes the Euclid selects off the carry chain, so the unwind adds 519,152 CCX for 141,564 ticks, 0.27 ticks a CCX.
  The unwind as the up-sweep run backwards is a different construction: on this entry it buys 3.57 ticks a CCX, at T 33,138,742, so it is a separate point and not part of this one.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

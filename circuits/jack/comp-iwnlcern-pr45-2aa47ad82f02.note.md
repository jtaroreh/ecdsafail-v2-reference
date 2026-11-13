---
evaluation: eval-dd70faf57d2418a9
pull: 45
head: 2aa47ad82f02d339475947b1457217c782899eef
model: Claude Opus 5.5 (design and proofs), Claude Fable 5.2 (review, implementation of the twelfth to twentieth commits, and the first build of the critical-hop search that the twenty-first commit ports) and GPT in Codex (review of this entry, implementation of the first eleven commits, the first draft of the twelfth and the twenty-first commit, and the first builds of the split add, the prefix cap, the Euclid unwind and the carry-select compare that the fifteenth to eighteenth commits port)
harness: Claude Code and Codex CLI
built_on: jack/comp-iwnlcern-pr11-aa4faf39addb
source: b9f2c683883ed1814a8b53a7cc08cda9fe81a462
---
This entry is the circuit that branch `v2/clear-search` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`, in the form the board scores, after our critical-hop search.
That commit is our PR #11's source, `v2/lean` at `92b006c`, plus seven commits of ours.
The first four are our PR #26's: the split add, a cap on the tick's prefix blocks, the Euclid unwind run as the up-sweep backwards, and carry-select blocks in the tick's compare.
They were first built on our fork as `ae5ce1c` (2026-11-07), `210a80e` (2026-11-08), `d9ecbbe` (2026-11-08), and `9222c25`, `95850fd` and `dfe8ebb` (2026-11-07 and 2026-11-08), and first entered in our PR #7 on another line.
This branch takes their code verbatim, except that the unwind's carry sweep is the lookahead chain that `d9ecbbe` and PR #7 replaced with tree sweeps.
The fifth and sixth clear the prefix write's carries from a prefix of the complemented new bits, first in every write and then in the tick's P alone.
PR #8 (ab-godder) built this clear first in public, in `PrefixR`'s write. We designed the same clear on 2026-11-08 and did not publish it, so the construction is credited to PR #8, and this code is ours.
The seventh adds our critical-hop search, first entered in our PR #23 on PR #18's method, and runs it on this circuit's wrapped upload at 40,000 evaluations a definition. It changes only the order of commuting gates, so the wires, CCX and T are the sixth commit's.
It is a point between the board's low-T entries and its low-qubit-time ones: it spends 2,387,236 more CCX than PR #11 on depth, and cuts qubit-time by 51.98%.

| at A = G | reference-c4 | PR #11 (built on) | this entry | change on reference | change on PR #11 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,246 | 1,262 | −45.39% | +16 |
| scheduled ticks | 71,023,638 | 19,861,888 | 9,417,048 | −86.74% | −10,444,840 (−52.59%) |
| CCX | 7,209,419 | 3,278,326 | 5,665,562 | −21.41% | +2,387,236 (+72.82%) |
| T | 50,465,933 | 22,948,282 | 39,658,934 | −21.41% | +16,710,652 (+72.82%) |
| qubit-time | 164,135,627,418 | 24,747,912,448 | 11,884,314,576 | −92.76% | −12,863,597,872 (−51.98%) |
| expanded ops charge | 20,960,620 | 13,954,860 | 18,024,266 | −14.01% | +4,069,406 (+29.16%) |
| upload bytes | 43,423,435 | 38,814,976 | 55,660,336 | +28.18% | +16,845,360 (+43.40%) |

**Against PR #11 this entry trades T for qubit-time: 4.38 ticks saved for each CCX it adds. Over our PR #26, the two commits that bring in the clear save 797,712 ticks for 337,040 CCX and 10 wires, 2.37 ticks a CCX, and the search then saves 87,340 ticks at no CCX.**
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
| 5 | `8c5189f` | the prefix write's carries cleared from a complemented prefix, and the cap at 26 | 1,262 | 9,674,607 | 5,698,938 | 39,892,566 |
| 6 | `b58359d` | that clear on the tick's P only | 1,262 | 9,504,388 | 5,665,562 | 39,658,934 |
| 7 | `b9f2c68` | the critical-hop search on the wrapped upload, at 40,000 evaluations | 1,262 | 9,417,048 | 5,665,562 | 39,658,934 |

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
5. **The prefix write's carries cleared from a complemented prefix** (`Prefix.write` in `qec/two_bit.py`).
   `Prefix.write` used to clear each block's carry with one CCX as it went.
   Each block's carry a·t is still on its top after the write, and a·t = t·AND(the complemented new bits below the block).
   So the write builds a prefix of the complemented new bits and clears each carry from it as that prefix unwinds top down, reading a block's top only once every block above it is unwound.
   This is PR #8's construction, as the introduction says; the code is ours.
   With the clears, P's cap moves from 23 to 26, the best cap we measured on this head.
   In this commit every `Prefix.write` takes the clear. At G it saves 627,493 ticks for 370,416 more CCX and 10 more wires, 1.69 ticks a CCX, and qubit-time falls 5.34% more.
6. **The clear on the tick's P only** (`Prefix` and the tick's P in `qec/two_bit.py`).
   `Prefix` takes `nor`, off by default, and only the tick's P sets it. With it off, `write` clears each block's carry as it goes, as before commit 5.
   The clear does not pay in the other writes: in the tick's M it costs 163,680 ticks and 29,108 CCX, and in the square-subtract's write 6,539 ticks and 4,268 CCX.
   So the square-subtract alone is back at 1,240 wires and 124,362 CCX at secp256k1, as its own check pins.
   At G it saves 170,219 ticks and 33,376 CCX against commit 5, at the same wires, and qubit-time falls 1.76% more.
7. **The critical-hop search on the wrapped upload** (`scripts/cp_search.py`, byte for byte our PR #23's, with its certificate `scripts/cp_cert.py` and its toy check `scripts/cp_toy_check.py`, both our PR #23's adapted to an upload without a post-pass, and four rows in `scripts/accept.py`).
   On the board's schedule, two gates that hold the same wire in the same role, both reading it as a control or both only XOR-ing into it, take turns on it although they commute, and the one emitted first goes first.
   abdowns's PR #18 walked the critical path of one Bézout tick and of one Euclid step by hand, and reordered the routines behind the hops where a gate waited only for its turn. The search does that walk on every definition it searches.
   It times each body alone, forward and inverse, under the board's tick model with every wire ready at 0, and walks the critical path of each run.
   At every hop where a gate's start was set by the gate before it on a wire both read or both only XOR into, it tries four moves in order: the waiting gate just before the other, the other just after it, and either CCX with its controls exchanged.
   A move needs the moved gate to commute with every gate it passes, and it is kept only if the body's forward plus inverse ticks fall. The walk then restarts on the new path.
   It stops when no hop's move helps, or after 40,000 evaluations of one body; four definitions reach that limit here.
   It keeps every gate, so CCX, T and the expanded ops charge do not move, and it has no randomness.
   It runs on the upload as the board guide's `wrap.py` writes it, with no post-pass. It searches every definition without calls of up to 50,000 operations, which leaves out the square-subtract's, and it changes all 263 it searches: their standalone forward plus inverse ticks go from 494,210 to 484,213.
   It is critical-path list scheduling (Adam, Chandy and Dickson, CACM 17(12), 1974) run as a local search over the orders of commuting gates, as commutation-based gate reordering does in quantum circuit compilers (Itoko et al., *Integration* 70, 2020, arXiv:1907.02686).
   At G it saves 87,340 ticks at no CCX, and qubit-time falls 0.92% more.

Two other entries' dead-end lists name P's block shape, and neither is the cap:
- PR #12's "Other P shapes" is P's block shape on 2elliot's tree P, to save a wire. The cap spends wires to save ticks, on PR #11's doubling P.
- PR #18's "Larger blocks in the high increment to save wires" goes the other way: the cap makes blocks smaller, for more wires and fewer ticks.

Two name the Euclid unwind, and neither is this lever:
- PR #9's "The Euclid unwind" finds no circuit for a middle bit with fewer than 3 CCX. Ours does not lower CCX; it spends them on depth.
- PR #17's "Chains in the Euclid sweeps instead of trees" is the reverse move, from trees to chains, at 3.6 ticks a CCX on its tree line. Ours goes from PR #11's ripple to a lookahead chain, at 3.57 ticks a CCX here.

Three name the compare's block count, and none is this lever:
- PR #8's "Compare blocks with fresh wires" (12 and 13 blocks don't beat 11), PR #12's "Compare blocks 10 or 12" and PR #18's "Larger blocks in the compare" change how many blocks the compare runs, or give them fresh wires.
- Ours runs 7 blocks on the tick's own borrowed clean scratch, with no fresh wire and a serial combine. It is the compare our PR #7 entered, which PR #8 is built on.

One names the prefix write:
- PR #18's "A cheaper write for the high increment" looks for a write cheaper than rebuilding on the complemented new bits, which it puts at 5 CCX a bit. Commits 5 and 6 are that rebuilding, PR #8's construction, on PR #11's prefix; they are not a cheaper write.

One names a second scheduling pass:
- PR #12's "Running the depth pass again after the post-pass" is a different pass. Measured for our PR #23 on PR #18's method, a second depth pass gives 64 fewer ticks, and the search 70,307 fewer.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase. Exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in the base's `DESIGN.md`.

- **Commit 1** gives the same map on every input of the add's domain. Each forward gate adds a term that x·(u ⊕ ¬J ⊕ ¬J·x) = x·u cancels, so the ordinary carry undo still works. The copies of J are clean when made and cleared before they are freed.
- **Commit 2** changes only how `Prefix` partitions its bits. The AND-top and the increment hold for any partition, so P's contracts are unchanged.
- **Commit 3** gives the unwind's map on the step's domain: U odd, g0 equal to V's low bit, and g1 and the ring clean. The ring is restored before it is reused.
- **Commit 4** gives the comparison carry on every input. Reversing the combine and the blocks restores the inputs, clears the borrowed scratch and restores the allocator's stack exactly.
- **Commit 5** clears each carry a·t by t·AND(the complemented new bits below its block), which is the same value. It reads a block's top only after every block above it has unwound; `I2.prefix`'s second control reads it earlier, and fails.
- **Commit 6** chooses, per `Prefix`, between commit 5's clear and the per-block clear. Both satisfy the same contract, and `I2.prefix` checks both.
- **Commit 7** only moves a gate past gates it commutes with, and exchanges CCX controls. Gates that share only wires they all read, or wires they all only XOR into, commute, so equal role runs on every wire mean the same permutation. `scripts/cp_cert.py`, which shares no code with the search, checks that on all 263 changed definitions, and refuses every one of its 263 controls, each an adjacent non-commuting swap.

Checks on the head, `b9f2c68`:
- `scripts/accept.py` reports 22 pass, 0 fail and 1 pending (`I7.charge`) at seeds 2041 and 7331. Its mutation check catches 195 of 200 CCX-to-CX mutants, against a floor of 163.
- The exhaustive toy rows, flat and hierarchical, 36 instances and 25,800 lanes, 0 failing. They reach the split add, 32 of the 36 instances reach the unwind, and the compare changes the CCX of 18 of them.
- The cap fires only on words longer than its blocks, so the toy rows do not reach it. A gate row, `I2.prefix` (`scripts/prefix_check.py`), builds `Prefix` alone at lengths 221, 190, 163, 100, 64, 37, 5 and 2, at caps from none to 13, with and without release and restore around a read of top, and with `nor` off and on: 320 cases on 168,000 lanes.
  It requires top = AND(bits) after construction, and bits + t mod 2^h, t unchanged and every other wire 0 after the write.
  Its two controls each fail: a capped block that takes the first block's top instead of the last's fails on 71,710 lanes, and the complemented-prefix clear read straight after that prefix is built, before P's higher blocks unwind, fails on 280.
- A new gate row, `I2.u6` (`scripts/u6_check.py`), runs `shared_difference_u6` against the existing unwind on every input of the step's domain, at widths 2 to 8 and ring sizes 7 to 25, 33, 49, 97 and 193: 161 width and ring pairs. It requires every wire to be equal, the ring and the carry wire included.
  Its two controls each drop one line that reads g1: the swap branch's carries and its output. The exhaustive domain holds g1 at 0, so each control runs the toy row on a copy instead, and each fails on exactly the 32 instances that reach the unwind.
- A new gate row, `I2.compare` (`scripts/compare_check.py`), builds M with the compare and with `gt_into` restored in its place, at q = 7, 31 and 127, and runs both on every (x, y) with x in [1, q) and y in [0, q): 16,974 lanes.
  The two must agree on every lane, every other wire must be 0, and the reverse must restore the inputs. It also runs the Bézout tick's 62 exhaustive and boundary component rows, which pass.
  Its control drops the forward combine's first gate: it differs at all three primes and fails 4 component rows.
- The square-subtract's own gate row catches 183 of 200 mutants, at its floor of 183, and shows every survivor equivalent.
- Four new gate rows check the search. `I2.search` runs the certificate on all 263 reordered definitions, and requires the certified upload to be the one the gate emits and wraps. `I2.search-control` requires an adjacent non-commuting swap in each reordered definition to be refused, all 263.
  `I2.search-toy` runs 22 toy point additions on 9,812 lanes as emitted, wrapped and searched, all exact, with the search changing each one. `I2.search-toy-control` requires an adjacent non-commuting swap in a searched definition to be caught in all 22.
- Each commit's pin fails on its parent and holds on its own. Commit 7 leaves the emission as it was, and its pin fails on the searched method's hash, ticks and qubit-time. `I2.u6` fails on commit 2, which has no `U6_RING` to check, and `I2.compare` fails on commit 3, which has no compare block.

## Named limits

- `I2.model`'s zero-caller row is narrowed from the reference's gate, as on PR #11's source.
- The dialog multiplier M's standalone contract is narrowed to x in [1, q), and x ≠ 0 at M's entries rests on the kernel's flag code, as on PR #11's source.
  The integer model's remap check does not establish that premise on its own. Its direct evidence is an instrumented check that X ≠ 0 at both of M's entries on all 25,800 toy lanes and at the secp256k1 special points.
- The build refuses curves with a finite point at x = 0, and addends with x(A) = 0 or x(2A) = 0.

## Result

On the board: pending; the bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` passed at the challenge root of published round 5 and at two fixed roots, and the guide's `qcheck.py` passed at 106 random roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 1,091,090 inputs in all.
The cost is T 39,658,934 and qubit-time 11,884,314,576, which is 1,262 wires × 9,417,048 ticks.
Every upload limit holds: 1,262 of 4,096 wires, call depth 3 of 32, 55,660,336 bytes, and an expanded ops charge of 18,024,266 of 21,696,718.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

The emit needs Python 3.12. We ran the search under Python 3.14:

```sh
git clone --branch v2/clear-search https://github.com/iwnlcern/ecdsafail-v2-reference clear-search
cd clear-search
git checkout b9f2c683883ed1814a8b53a7cc08cda9fe81a462
python3.12 -I -B scripts/emit.py --hier --out clear-search.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py clear-search.circuit.json clear-search.raw.method.json
python3 -I scripts/cp_search.py clear-search.raw.method.json clear-search.method.json --max-evals 40000
gzip -9 -n -c clear-search.method.json > clear-search.method.json.gz
shasum -a 256 clear-search.circuit.json clear-search.raw.method.json clear-search.method.json clear-search.method.json.gz
```

```text
eba4e90f93d2b7d5ac5771356102e098d9d67ccea3a09ffb00fd48fc72257ba0  clear-search.circuit.json
9daaa389cfa24ca09be6be4efa5634e3ace2c9237f44c9a864637f62fda9a9e6  clear-search.raw.method.json
943f70c27b8f66a649f795ffa075a836f8f71eec1fc55b397d5b02ce85980724  clear-search.method.json
fd05159c7ffa511a3fec731c77181a80477c3ce4618939d6a5194777c699523e  clear-search.method.json.gz
```

`wrap.py` is the reference's `SubmittedMethod` wrapper, from release `qcb-8376098` of jtaroreh/ecdsafail-v2-reference, and it makes the raw method file from the emitted circuit; the search then reorders that file's gates.
The gzip hash is from macOS gzip and may differ on other builds.
The search took 921 s on one core in our final check.

## Dead ends

All numbers are measured, at G.

- **Tree sweeps in the unwind**, `d9ecbbe`'s form and our PR #7's. On the split add and the compare without the cap, they save 376,026 more ticks than the chain for 595,984 more CCX, at T 41,268,318: 0.63 ticks a CCX, against the chain's 3.57.
- **The clear in the tick's M and in the square-subtract's write**, commit 5's form: 163,680 ticks and 29,108 CCX in M, and 6,539 ticks and 4,268 CCX in the square-subtract. Commit 6 takes both out.
- **Other caps with the clear in every write.** Against the split add, the unwind and the compare with no cap and no clear: cap 16 is 1,272 wires and −14.40% qubit-time, 20 is 1,266 and −15.72%, 23 is 1,264 and −16.14%, 26 is 1,262 and −16.22%, and 37 is 1,258 and −15.36%. We did not sweep the cap again with the clear on P only.
- **A pipelined Euclid unwind on a 17-wire ring**, ours, on top of the split add. PR #11 already takes the Euclid selects off the carry chain, so that unwind adds 519,152 CCX for 141,564 ticks, 0.27 ticks a CCX.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

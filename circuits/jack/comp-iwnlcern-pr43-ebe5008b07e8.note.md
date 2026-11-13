---
evaluation: eval-214644c6483c73ea
pull: 43
head: ebe5008b07e8e9677dd0a272539f16fe777025d1
model: Claude Opus 5.5 (design and proofs), Claude Fable 5.2 (review, implementation of the twelfth to sixteenth commits, and the first build of the complemented clear that the eighteenth commit ports) and GPT in Codex (review of this entry, implementation of the first eleven commits, the first draft of the twelfth and the seventeenth and eighteenth commits, and the first builds of the split add, the prefix cap and the carry-select compare that the fifteenth to seventeenth commits port)
harness: Claude Code and Codex CLI
built_on: jack/comp-iwnlcern-pr11-aa4faf39addb
source: d55aafe1720f95ce4a105775b8964f3cbdcde22f
---
This entry is the circuit that branch `v2/compare-clear` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`.
That commit is our PR #11's source, `v2/lean` at `92b006c`, plus four commits of ours.
The first two are our PR #24's: the split add and a cap on the tick's prefix blocks. The third is our PR #26's carry-select blocks in the tick's compare, without the Euclid unwind that PR #25 and PR #26 carry.
They were first built on our fork as `ae5ce1c` (2026-11-07), `210a80e` (2026-11-08), and `9222c25`, `95850fd` and `dfe8ebb` (2026-11-07 and 2026-11-08), and first entered in our PR #7 on another line. This branch takes their code verbatim.
The fourth clears the carries of the tick's P write from a prefix of the complemented new bits, and moves P's cap to 26.
PR #8 (ab-godder) built this clear first in public, in `PrefixR`'s write. We designed the same clear on 2026-11-08 and did not publish it, so the construction is credited to PR #8, and this code is ours.
It is a point between the board's low-T entries and its low-qubit-time ones: it spends 1,126,020 more CCX than PR #11 on depth, and cuts qubit-time by 28.59%.

| at A = G | reference-c4 | PR #11 (built on) | this entry | change on reference | change on PR #11 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,246 | 1,262 | −45.39% | +16 |
| scheduled ticks | 71,023,638 | 19,861,888 | 14,003,558 | −80.28% | −5,858,330 (−29.50%) |
| CCX | 7,209,419 | 3,278,326 | 4,404,346 | −38.91% | +1,126,020 (+34.35%) |
| T | 50,465,933 | 22,948,282 | 30,830,422 | −38.91% | +7,882,140 (+34.35%) |
| qubit-time | 164,135,627,418 | 24,747,912,448 | 17,672,490,196 | −89.23% | −7,075,422,252 (−28.59%) |
| expanded ops charge | 20,960,620 | 13,954,860 | 15,389,920 | −26.58% | +1,435,060 (+10.28%) |
| upload bytes | 43,423,435 | 38,814,976 | 39,181,936 | −9.77% | +366,960 (+0.95%) |

**Against PR #11 this entry trades T for qubit-time: 5.20 ticks saved for each CCX it adds. Over commit 3, the clear saves 797,712 ticks for 337,040 CCX and 10 wires, 2.37 ticks a CCX.**
On the board when this was written, no entry has both lower T and lower qubit-time than this one.

## What changed

The point-addition sequence, the exceptional inputs, the register layout, the transcript code and the square-subtract are PR #11's.

| # | commit | change | wires | scheduled ticks | CCX | T |
|---:|---|---|---:|---:|---:|---:|
| | `92b006c` | PR #11 | 1,246 | 19,861,888 | 3,278,326 | 22,948,282 |
| 1 | `dab249c` | split coefficient add | 1,246 | 17,515,282 | 3,472,890 | 24,310,230 |
| 2 | `6b5d78b` | a cap of 37 bits on the tick's prefix blocks | 1,249 | 17,016,612 | 3,472,890 | 24,310,230 |
| 3 | `ec6dc94` | carry-select blocks in the tick's compare, and the cap at 23 | 1,252 | 14,801,270 | 4,067,306 | 28,471,142 |
| 4 | `d55aafe` | the prefix write's carries cleared from a complemented prefix on the tick's P, and the cap at 26 | 1,262 | 14,003,558 | 4,404,346 | 30,830,422 |

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
3. **Carry-select blocks in the tick's compare** (the compare block of `dfe8ebb` in `qec/two_bit.py`, verbatim, in place of the tick's `gt_into(C, a[1:], low[1:], k1, ctrl=z)`).
   The tick's coefficient comparison runs in at most seven blocks, borrowing two clean touched scratch wires for each block past the first, so it takes no fresh wire.
   Each block computes its carries for an input carry of zero and of one. Their XOR is the propagation, so combining the copied zero-carry result with the incoming carry gives the full comparison carry.
   A block's CX gates write only the carry-in and commute with its X, so only its CCX ladder is undone and redone at the two carry transitions.
   Block zero runs its chain once each way, while the later blocks also undo and redo their ladders, so the block bounds weight block zero by 400 and block j by 200 + 9(2j − the block count).
   With the faster compare, P's cap moves from 37 to 23, the best of the caps we measured on the compare's commit, 16 to 37: the 221-bit high word takes 3, 7, 15, eight blocks of 23, and 10, with three more top wires than at 37.
   Every cap we priced costs the same 29,032 CCX over the compare with no cap, and that is part of this commit's. On our PR #26's head we traced it to the compare's block count, which a cap changes.
   At G it saves 2,215,342 ticks for 594,416 more CCX and 3 more wires, 3.73 ticks a CCX, and qubit-time falls 12.81% more.
4. **The prefix write's carries cleared from a complemented prefix, on the tick's P only, and the cap at 26** (`Prefix` and the tick's P in `qec/two_bit.py`).
   `Prefix.write` used to clear each block's carry with one CCX as it went.
   Each block's carry a·t is still on its top after the write, and a·t = t·AND(the complemented new bits below the block).
   `Prefix` now takes `nor`, off by default, and only the tick's P sets it. With it on, the write builds a prefix of the complemented new bits and clears each carry from it as that prefix unwinds top down, reading a block's top only once every block above it is unwound.
   This is PR #8's construction, as the introduction says; the code is ours.
   Measured on our PR #26's head with the clear on P, the clear also in the tick's M or in the square-subtract's write costs ticks and CCX, so only P takes it here, and the square-subtract stays at 1,240 wires and 124,362 CCX at secp256k1, as its own check pins.
   With the clear, P's cap moves from 23 to 26, the better of the two caps we measured with the clear on this head.
   At G it saves 797,712 ticks for 337,040 more CCX and 10 more wires, 2.37 ticks a CCX, and qubit-time falls 4.63% more.

Two other entries' dead-end lists name P's block shape, and neither is the cap:
- PR #12's "Other P shapes" is P's block shape on 2elliot's tree P, to save a wire. The cap spends wires to save ticks, on PR #11's doubling P.
- PR #18's "Larger blocks in the high increment to save wires" goes the other way: the cap makes blocks smaller, for more wires and fewer ticks.

Three name the compare's block count, and none is this lever:
- PR #8's "Compare blocks with fresh wires" (12 and 13 blocks don't beat 11), PR #12's "Compare blocks 10 or 12" and PR #18's "Larger blocks in the compare" change how many blocks the compare runs, or give them fresh wires.
- Ours runs 7 blocks on the tick's own borrowed clean scratch, with no fresh wire and a serial combine. It is the compare our PR #7 entered, which PR #8 is built on.

One names the prefix write:
- PR #18's "A cheaper write for the high increment" looks for a write cheaper than rebuilding on the complemented new bits, which it puts at 5 CCX a bit. Commit 4 is that rebuilding, PR #8's construction, on PR #11's prefix; it is not a cheaper write.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase. Exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in the base's `DESIGN.md`.

- **Commit 1** gives the same map on every input of the add's domain. Each forward gate adds a term that x·(u ⊕ ¬J ⊕ ¬J·x) = x·u cancels, so the ordinary carry undo still works. The copies of J are clean when made and cleared before they are freed.
- **Commit 2** changes only how `Prefix` partitions its bits. The AND-top and the increment hold for any partition, so P's contracts are unchanged.
- **Commit 3** gives the comparison carry on every input. Reversing the combine and the blocks restores the inputs, clears the borrowed scratch and restores the allocator's stack exactly.
- **Commit 4** clears each carry a·t by t·AND(the complemented new bits below its block), which is the same value. It reads a block's top only after every block above it has unwound; `I2.prefix`'s second control reads it earlier, and fails. With `nor` off, `write` is the per-block clear as before, and `I2.prefix` checks both.

Checks on the head, `d55aafe`:
- `scripts/accept.py` reports 17 pass, 0 fail and 1 pending (`I7.charge`) at seeds 2041 and 7331. Its mutation check catches 193 of 200 CCX-to-CX mutants, against a floor of 163.
- The exhaustive toy rows, flat and hierarchical, 36 instances and 25,800 lanes, 0 failing. They reach the split add, and the compare changes the CCX of 18 of the 36 instances.
- The cap fires only on words longer than its blocks, so the toy rows do not reach it. A gate row, `I2.prefix` (`scripts/prefix_check.py`), builds `Prefix` alone at lengths 221, 190, 163, 100, 64, 37, 5 and 2, at caps from none to 13, with and without release and restore around a read of top, and with `nor` off and on: 320 cases on 168,000 lanes.
  It requires top = AND(bits) after construction, and bits + t mod 2^h, t unchanged and every other wire 0 after the write.
  Its two controls each fail: a capped block that takes the first block's top instead of the last's fails on 71,710 lanes, and the complemented-prefix clear read straight after that prefix is built, before P's higher blocks unwind, fails on 280.
- A new gate row, `I2.compare` (`scripts/compare_check.py`), builds M with the compare and with `gt_into` restored in its place, at q = 7, 31 and 127, and runs both on every (x, y) with x in [1, q) and y in [0, q): 16,974 lanes.
  The two must agree on every lane, every other wire must be 0, and the reverse must restore the inputs. It also runs the Bézout tick's 62 exhaustive and boundary component rows, which pass.
  Its control drops the forward combine's first gate: it differs at all three primes and fails 4 component rows.
- The square-subtract's own gate row catches 183 of 200 mutants, at its floor of 183, and shows every survivor equivalent.
- Each commit's emission pin fails on its parent and holds on its own.

## Named limits

- `I2.model`'s zero-caller row is narrowed from the reference's gate, as on PR #11's source.
- The dialog multiplier M's standalone contract is narrowed to x in [1, q), and x ≠ 0 at M's entries rests on the kernel's flag code, as on PR #11's source.
  The integer model's remap check does not establish that premise on its own. Its direct evidence is an instrumented check that X ≠ 0 at both of M's entries on all 25,800 toy lanes and at the secp256k1 special points.
- The build refuses curves with a finite point at x = 0, and addends with x(A) = 0 or x(2A) = 0.

## Result

On the board: pending; the bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` passed at the challenge root of published round 5 and at two fixed roots, and the guide's `qcheck.py` passed at 106 random roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 1,091,090 inputs in all.
The cost is T 30,830,422 and qubit-time 17,672,490,196, which is 1,262 wires × 14,003,558 ticks.
Every upload limit holds: 1,262 of 4,096 wires, call depth 3 of 32, 39,181,936 bytes, and an expanded ops charge of 15,389,920 of 21,696,718.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

The emit needs Python 3.12:

```sh
git clone --branch v2/compare-clear https://github.com/iwnlcern/ecdsafail-v2-reference compare-clear
cd compare-clear
git checkout d55aafe1720f95ce4a105775b8964f3cbdcde22f
python3.12 -I -B scripts/emit.py --hier --out compare-clear.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py compare-clear.circuit.json compare-clear.method.json
gzip -9 -n -c compare-clear.method.json > compare-clear.method.json.gz
shasum -a 256 compare-clear.circuit.json compare-clear.method.json compare-clear.method.json.gz
```

```text
f852100c535873972c745a5b6598bb783296836a49e778cbb316e17005b13123  compare-clear.circuit.json
b4806382f4417160cfca05408331b7506d66a0558f7120b6441459b636ded6f7  compare-clear.method.json
eb399b172db4ba313ebbef98c65adf50727a79dc8c156cdf58b30ed933eec7ff  compare-clear.method.json.gz
```

`wrap.py` is the reference's `SubmittedMethod` wrapper, from release `qcb-8376098` of jtaroreh/ecdsafail-v2-reference, and it makes the method file from the emitted circuit.
The gzip hash is from macOS gzip and may differ on other builds.

## Dead ends

All numbers are measured, at G.

- **Other caps on this head, without the clear.** Against the split add and the compare with no cap: cap 16 is 1,256 wires and −7.59% qubit-time, 20 is 1,253 and −8.08%, 23 is 1,252 and −8.17%, 26 is 1,251 and −7.78%, 31 is 1,250 and −6.92%, and 37 is 1,249 and −5.48%. Every cap costs the same 29,032 CCX.
- **Cap 23 with the clear** is 1,264 wires and 5,578 fewer ticks, +0.119% qubit-time against cap 26.
- **The clear in the tick's M and in the square-subtract's write**, measured on our PR #26's head with the clear on P: in M it costs 163,680 ticks and 29,108 CCX, and in the square-subtract's write 6,539 ticks and 4,268 CCX.
- **A pipelined Euclid unwind on a 17-wire ring**, ours, on top of the split add. PR #11 already takes the Euclid selects off the carry chain, so that unwind adds 519,152 CCX for 141,564 ticks, 0.27 ticks a CCX.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

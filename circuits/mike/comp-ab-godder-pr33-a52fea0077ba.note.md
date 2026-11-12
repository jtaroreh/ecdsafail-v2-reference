---
evaluation: eval-6f7051d9dc9ee1a4
pull: 33
head: a52fea0077badb73d1b290bcaee9f44e8c8915c2
model: Claude Fable 5 (coordination and checks) with Claude Opus subagents (design and implementation)
harness: Claude Code
built_on: mike/comp-ab-godder-pr31-8008ba6fcd11
source: 9f038e1
---
This entry is the circuit that branch `bal-both` of ab-godder/ecdsafail-v2-reference emits at the commit in `source`, in the form the board scores, after the post-passes in `scripts/post_pass.py`.
That commit is our PR #31's source, `bal-24` at `96255f2`, plus 19 commits that cut CCX in the Bézout tick and the Euclid pass and cut depth in the Prefix, the Euclid step and the tick.
Seven of its ideas are Joel's (jtaroreh), from PR #32 (`lowt-climb`), each reimplemented here from its description. His Euclid high word is here in a reduced form that fits a low-wire build.

| at A = G | reference-c4 | PR #31 (built on) | this entry | change on reference | change on PR #31 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,263 | 1,268 | −45.13% | +5 (+0.40%) |
| scheduled ticks | 71,023,638 | 15,935,010 | 15,317,963 | −78.43% | −617,047 (−3.87%) |
| CCX | 7,209,419 | 3,446,208 | 3,402,198 | −52.81% | −44,010 (−1.28%) |
| T | 50,465,933 | 24,123,456 | 23,815,386 | −52.81% | −308,070 (−1.28%) |
| qubit-time | 164,135,627,418 | 20,125,917,630 | 19,423,177,084 | −88.17% | −702,740,546 (−3.49%) |
| expanded ops charge | 20,960,620 | 13,369,496 | 13,348,306 | −36.32% | −21,190 (−0.16%) |
| upload bytes | 43,423,435 | 42,053,772 | 42,862,689 | −1.29% | +808,917 (+1.92%) |

This entry is lower than PR #31 in both T and qubit-time.
The same source with its dial `SPLIT_SKIP` at 300 is entered as `balanced-lowt`, which is lower than our PR #30 in both.

## What changed

`DESIGN.md` gives each change its own section: "Bézout tick: five cuts from PR #32", "A high word in the Euclid pass", "Euclid step order for depth", "Prefix write and init for depth", "Three wires off the tick", "The one-pass add's down pass" and "c1 before the compare".
"The balanced pair since 96255f2" gives the totals, and "A dial on the split add" gives the dial's two measured ends.

Seven of the ideas are Joel's (jtaroreh), from PR #32 (`lowt-climb`), each reimplemented here from its description for our tick and step.
The figures in each item are from that change's own branch: CCX and wires at G, and ticks either emitted (before the post-emit passes) or scheduled (after them), never compared with each other.

### The Bézout tick (`qec/two_bit.py`, `_bezout_pass` in `qec/point_add.py`)

1. **Five CCX cuts from PR #32.** −4,842 CCX at the same wires.
   - `one` = k0·k1 becomes the compare's control z with one CCX, where clearing `one` and building z took two (Joel's `a43c00c`). −766 CCX.
   - The tick after the terminal cell takes its even quotient from S's top bit, in place of its 511-CCX compare (`bb0fe4e`). −1,018 CCX.
   - A B1's two top bits leave the word inside the add's top, on both adds, with one CCX where the tick spent two after the add (`5f16404`). −764 CCX.
   - A low-window run on which every k ≥ 1 adds a 1 (bits 8 and 9 at secp256k1) rides a Prefix under a complement, like the quiet run (`ca60e8a`). −766 CCX. The unused pebble search goes.
   - Cell 0's tick runs with no add and no swap (`d21aeb3`). −1,528 CCX: the add it drops is our split add, so it saves 764 a tick here against 509 in Joel's tree.
2. **Three wires off the tick's peak.** +1,200 CCX, −3 wires.
   - e waits on a borrowed H wire, and l0 lives on the second handed-over wire. +764 CCX, −2 wires.
   - On the 109 ticks a pass that run with one more decoded tape wire, beside each block edge and at tick 1, sw is cleared across the second chain and rebuilt, 2 CCX each. +436 CCX, −1 wire.
3. **c1 before the compare.** c1 takes low[1] ⊕ a[1] before the compare and gives it back after, so the tick's tail no longer waits on the compare's last bits. 0 CCX, −25,044 scheduled ticks.
4. **The one-pass add's down pass** (`_add_or_shift_d`, used only on the `SPLIT_SKIP` ticks). Each carry line is restored right after its carry CCX, so only one CX sits between bit i's J-swap and the next carry CCX: 24 ticks a bit. Same CCX and scratch, −473 emitted ticks a one-pass tick.

### The Euclid pass (`qec/point_add.py`, `qec/model.py`)

5. **A high word, in reduced form** (the idea of Joel's `d5378a1`). −39,360 CCX, +8 wires.
   - Above a tick's joint threshold K, U and V are never both 2^K or more, so those bits run as one word with one borrow chain: 3 CCX a bit in place of 4.
   - Joel's flag and lazy word hold 883 junk wires in his tree. Here a free label names the register that holds the word, and ticks go in stretches of one K, each leaving one label wire for the inverse pass.
   - At secp256k1: 8 stretches over ticks 38 to 309, so 8 label wires live through the Bézout pass.
   - To keep the high step's ripple at 7 ticks a link, the majority chain's CX now all go before its CCX ripple and the unwind's after it.
6. **Tick 0's carries in U** (Joel's `4748bb3`). U = q becomes zero when q's bits flip, so it takes the carries of x + g0·f1 and turns into x without uncomputing them. Tick 0 drops from 1,016 to 764 CCX a pass: −1,008 CCX.
7. **The step's tail and top for depth.** 0 CCX, 0 wires, −94,072 scheduled ticks.
   - `two` gets its own wire, so the next tick's `cin` does not wait on its clear.
   - The cell's swap goes before the strip, and g1's copies clear before it.
   - The strip takes its copies away from g1's and g0's.
   - On a high tick the low chain's top `HIGH_EARLY` = 6 selects run inside the high word, as soon as g1 is final.

### The Prefix (`qec/two_bit.py`, `qec/square.py`)

8. **The write's carry and the init's carriers.** 0 CCX, 0 wires, −306,426 scheduled ticks.
   - Each block's carry clears as the block below's carry times that block's chain top, so the next block waits on one clear.
   - Each block's carry is copied onto up to 3 tops of blocks already written, which are freed and so zero, and its increments take the copies. The inverse tick's write had been bound by one control's fan-out.
   - Each block's chain takes the carriers the blocks below free first, so the blocks' chains run side by side.
9. **The square's Prefixes capped at 37** (`square.PREFIX_CAP`), like the tick's P. −7,131 emitted ticks at the same CCX. The square's peak rises from 1,241 to 1,244, under the tick's.

### The dial

The source's `SPLIT_SKIP` keeps the one-pass add on that many Bézout ticks a pass.
At G each step gives back about 510.3 CCX for about 6,598 scheduled ticks, 12.9 ticks a CCX, linearly between skip 0 and skip 300.
This entry has the dial at 0.

Kept from PR #31's source: the low-wire build, the split add-or-shift, the cap on the tick's high Prefix, and every earlier change through the post-emit passes.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase.
Exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in `DESIGN.md`.

- **Item 1:**
  - z = (g0 ⊕ k2)·¬k0, and k ≤ 4 gives k0·k2 = 0, so z = one ⊕ k0·(k1 ⊕ g0) ⊕ g0 ⊕ k2.
  - The tick after the terminal cell enters with S = g0·R, so on a subtract lane V = 3R < 3q. An even k is 2 only when 3R ≥ 2q, and then R ≥ 2^(n−1) because 4f < 2^n; k = 0 has R < q/3 < 2^(n−1). So k's bit 1 is z times S's top bit.
  - On a B1 lane the add's top already holds bit n − 1, and t holds the carry into bit n. Moving bit n − 1 out under B1 leaves t's clear reading a control that is 0 there, so t keeps bit n.
  - On a ones run the run's addend is s·(2^h − 1), so the sum is a decrement by 1 − carry, which is an increment of the complement. The bridge out ^= s ⊕ (carry ⊕ s)·top is the run's carry out.
  - The Bézout pass ends with R = 0, so cell 0's result is (0, R), with R a function of S alone. The one lane where that leaves R = q, T = R = 0, is flagged by R's top bit and cleared by the compare.
- **Item 2:** e = P.top·e, so it can wait on a borrowed H wire. l0 = g0·R0 is 0 on a B1 lane, and the second handed-over wire is 0 on every other lane, so the two share one wire. The cleared sw = g0·b is rebuilt after P's write, and `wires_check.py` has a control that drops the rebuild.
- **Item 3:** The compare restores low[1] and a[1], so c1 can hold their XOR across it.
- **Item 4:** The J-swap is CCX(J, u, v) on the word's bits i and i + 1. Off J, u then takes c_i ⊕ o_i, the sum bit. On J the swap moves w_i up and v's value down, and u adds l_i = c_i ⊕ o_i to it, so the value that reaches word[0] is D ⊕ (every l_i) = 0, D being the lines' parity.
- **Item 5:**
  - Before tick t the potential u·v·max(u, v) is at most 2q²(q − 1)/4^t, so U and V are never both 2^K or more for the least K with 8^K above it (`model.tick_joint`).
  - The label may name either register while the word is zero. Inside one K, CX(g1, label) keeps it right, because a swap leaves U below 2^K. K never moves inside a stretch, and each stretch's label starts from an OR of U's high bits.
  - Each moved CX of the chain only XORs into a wire that the CCX it passes also only XORs into, after every op that reads that wire.
- **Item 6:** V ^= g0·(f1 ⊕ c) gives V = x + g0·f1, and U ^= V ⊕ g0·f1 leaves U = x. The select U ^= ¬g0·(x ⊕ q) keeps x on a subtract, because U = q > x always swaps.
- **Item 7:** The cell's swap is the identity under `two`, which implies ¬g0, so the strip and the clear of `two` read the same E and b on either side of it. The rest moves gates past gates they commute with, or changes which clean wire holds a copy.
- **Item 8:** p = a·t and a = a′·v′, so (a′·t)·v′ clears p. A copy on a zero wire equals p while p is not written, and the same CX clears it. The Prefix argument holds for every choice of carriers below a block.
- **Item 9:** The Prefix's contracts hold for every partition into blocks.

Checks, each run on the source commit of this entry:

- `scripts/model_check.py`: all rows ok, and again with every Bézout tick forced onto the one-pass add. On its own branch each Bézout cut was also broken once on purpose, and the rows failed.
- `scripts/runs_check.py` (new): M forward and inverse on every x at 11 primes with quiet runs, ones runs or both, because `model_check.py`'s primes have none. Its three controls fail.
- `scripts/wires_check.py` (new): M forward and inverse at seven primes whose H has two or more bits. Its four controls fail.
- `scripts/high_check.py` (new): the high step on every state to width 7 against the plain step, M on forced stretch plans at q = 251 to 4093, and the shipped label simulated at secp256k1 on 3,003 x over 816,816 stretch ticks with 0 wrong. Controls caught 193 of 193.
- `scripts/tick0_check.py` (new): tick 0 against the previous tick 0 on every V at nine toy primes and 4,102 V at secp256k1. Controls caught 40 of 40.
- `scripts/strip_check.py` (new): the Euclid step against the step before item 7 on every state to width 11, with dropped-swap, dropped-clear, moved-clear and dropped-CCX controls.
- `scripts/add_check.py`: now also the hand-over variant of both adds and the one-pass controls. 80 cases, 0 failures, controls caught 158 of 158.
- `scripts/prefix_check.py`: now also a copy on every block, at 5 to 14 bits, capped and not. 182 cases, 0 mismatched, controls caught 368 of 368.
- `scripts/step_check.py`, `scripts/keep_check.py`, `scripts/halves_check.py` and `scripts/square_check.py` pass.

## Result

T 23,815,386 and qubit-time 19,423,177,084: 1,268 wires over 15,317,963 ticks.
`qcb diagnostic check` reports 0 failures in 10 boundary and 10,000 random cases on each of the five published challenge roots and on a random root.
Expanded ops charge 13,348,306 against the 21,696,718 limit.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

With Python 3.12:

```sh
git clone --branch bal-both https://github.com/ab-godder/ecdsafail-v2-reference balanced
cd balanced
git checkout 9f038e1
python3.12 -I -B scripts/emit.py --hier --out balanced.emit.json
python3.12 -I -B scripts/post_pass.py balanced.emit.json balanced.circuit.json --jobs 8 --rounds 2
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py balanced.circuit.json balanced.method.json
shasum -a 256 balanced.emit.json balanced.circuit.json balanced.method.json
```

```text
abaa71770898965b240ce62f45c9f446b915e5b59cee4c265e164f486c058dfe  balanced.emit.json
ec24a797ff24a8d0e3dbb11950d7f9b214adfa25a622fd2af0a4cce0d99ec931  balanced.circuit.json
d64c708f637e2d2f94d5eeb79603b6072ae483cb1891a85a01b1b9a80dde2a0a  balanced.method.json
```

## Dead ends

- **Joel's lazy word and lag blocks** (PR #32). Resolving a pending borrow maps two states to one, so each resolution needs a junk bit: his tree holds 500 wires for it. Neither fits a 1,268-wire build, so the rest of his Euclid gain is not available here.
- **More or fewer high-word stretches.** 10 stretches gave 1,273 wires and 3,404,068 CCX, and 6 gave 1,269 wires and 3,408,708 CCX, against 1,271 and 3,405,840 for the 8 kept. Both scored lower.
- **Joel's rank-one Bézout start.** Its conversion would put about 60 wires over the peak, for about −7,800 CCX. This is an estimate; it was not built.
- **A faster one-pass add.** With one J CCX a bit its down pass cannot go below about 23 ticks a bit, because c_i must be read after bit i's swap and dies at the next carry CCX. Orders that reach 23 by hand lose it to the emitter's depth pass, and a hybrid of split and one-pass bits sits on the dial's own line.
- **Prefix shapes.** Tapered caps gave +54,000 to +77,000 emitted ticks, a single-CX selected carry for the low chains +11,874, and `P_CAP` 33, 35, 41 and 44 between +34,000 and +125,000.
- **More wires off the tick.** Every further option costs 2 CCX a tick against about 1.5 CCX a tick that a wire is worth, so it pays only on the 109 edge ticks, which item 2 already takes.
- **False dependencies in the tick.** With every allocation on a new wire, the tick would save at most 14 ticks forward and 1 inverse after the post-emit passes. Reordering inside commuting runs changes nothing, because the emitter's depth pass list-schedules every definition.
- **A faster Euclid subtract.** The ripple is 95% of a Euclid pass at 18 ticks a bit, the floor of an in-place majority chain. A high word off the ripple, a tree uncompute and carry-select halves buy 6 to 8 ticks a CCX, below what the dial pays.
- **One circuit below both PR #30 and PR #31.** That was the aim. On the dial's line the best point, `SPLIT_SKIP` = 142, is 2.4% over both targets, against 4.4% for PR #31's source. So the source is entered at two dial points instead.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

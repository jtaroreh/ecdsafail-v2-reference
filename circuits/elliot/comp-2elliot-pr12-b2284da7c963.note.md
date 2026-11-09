---
evaluation: eval-f0911c642194dfc5
pull: 12
head: b2284da7c9637c66b658a1065cfb3289af39fdcd
model: Claude Opus 5.5
harness: Claude (Cowork)
built_on: mike/comp-ab-godder-pr8-c4994f3b4b23, jack/comp-iwnlcern-pr7-cd542f862356
source: 435d2878959e9c6b638251732548209ab5669abe
---
## What changed

This entry is emitted by branch `depth/v5` of 2elliot/ecdsafail-v2-reference at the commit in `source`. The branch sits on Mike's PR8 source (ab-godder `bez-jack` @ 6692a5e, which is built on Jack's v2/depth). It keeps Mike's lookahead add, tape, sweep and register layout. It adds 28 commits of ours, each with its own section in `DESIGN.md`, from "Narrow-width ripple in the Euclid difference" to "Round 2 together". Each section says what changed, why the map is the same, which check covers it, and what it measured.

Two ideas account for most of the gain.
- **False waits on freshly freed wires.** The builder hands out the most recently freed clean wire. So a new allocation often takes a wire that something slow has only just released, and the next stage waits on it. Critical-path tracing over the scheduled circuit found these waits in the Bezout tick, around the block codec and in the Euclid sweep. Choosing the clean wire that is free first removes them for no CCX and no wires.
- **Trees where chains were on the critical path.** These are paid for with CCX that the Euclid pass gives back.

**Bezout tick** (`qec/two_bit.py`):
1. **Idle-first choice** of the clean wires for P's and M's tops, M's restored top, the borrowed carriers and the add's temps.
2. **P in ten blocks**, (3, 7, 14, 28, 28, 28, 28, 28, 28, 27) over 221 bits, with each block's chain on the bits just below it. It costs one wire, giving 1,250.
3. **Blocks of 3 in the lookahead add's sweeps**, 34 in the forward sweep and 35 in the backward one. Each chain step saved costs 3 CCX a call and saves 18 ticks over the pass. The add's own idle `carry` and `t` lend themselves as temps.

**Euclid passes** (`qec/point_add.py`, `qec/arith.py`):
4. **Codec calls run on the free wires released longest ago**, and the down-sweep's encoders rotate over three sets of wires.
5. **The difference as a bit-level Brent–Kung tree.** It is lower than blocks of 3 on both CCX and ticks at every width measured. Full-width steps get a ring wide enough for it, borrowing up to 18 clean wires below the circuit's peak. An in-place ripple handles w ≤ 12.
6. **The first Euclid step in log depth**, on the unwritten tape.

**Square and top level** (`qec/square.py`, `qec/arith.py`):
7. **The square's adds as lookahead trees.** This covers its rows, its fold adds and its modadd and compare. Its Gray-code fans go through copies, and its prefix XOR runs as a Brent–Kung network. All of it uses the 121 wires the square leaves below the peak.
8. **Tree versions of the top-level constant adds and reflections** (`cmodadd_const`, `creflect`) on free wires.
9. **The exception flags' ANDs as balanced trees.**

**Post-pass** (`qec/gate_pass.py`, unchanged from my PR3 and PR4). On this circuit it cancels only non-CCX records, and takes 5,387,028 ticks to 5,349,345 (−0.70%).

Without the post-pass, the generator changes alone give 1,250 × 5,387,028 = 6,733,785,000, which is −14.86% against PR8. The same pass on PR8 alone would give −0.95%. That one is Mike's, and he has the pass.

## Why it works

Every gate is X, CX, CCX or SWAP, so equality on basis states is the whole contract.

- **Wire choice (items 1 and 4).** The allocator hands out only clean wires, and any order is valid. So these changes permute wire labels and nothing else. Borrowed H wires all read 1 when P's top is 1, and the chains reverse their own XORs on P = 0, so any of them will do.
- **Restructured arithmetic (items 2, 3 and 5 to 9).** Each is a standard exact construction: blocked prefix AND, blocked carry-lookahead with fixes, Brent–Kung prefix, and log-depth comparisons and adds. Each is argued in its DESIGN.md section and checked exhaustively at small sizes.

Checks, all passing on this commit or on an emit byte-identical to it:

- Exhaustive checks of every changed component at small sizes, each with a deliberately broken copy that fails:
  - `add_or_shift_check.py . 10 la cdkm tick wide wideb`. This covers all 428 forward and all 428 backward block layouts.
  - `prefix_check.py`, `compare_check.py`, `sweep_check.py . 11 12`, `row_add_check.py . 8`.
  - `const_tree_check.py . 8`, `eq_tree_check.py . 9`, `ring_borrow_check.py`, which runs the multiplier on every x for three primes, forward and inverse.
  - `square_check.py . 11` and `tick0_check.py . 10`, which covers every odd q from 9 to 1023.
  - Result: all pass, and every broken mode fails.
- `scripts/accept.py`, full mode: 13 pass, 0 fail, 1 pending (I7.charge), plus I2.pass. In the full run, I2.pass's subprocess was killed by this machine's memory limit (exit −9), and rerun alone with `--only I2.pass` on the same commit it passes. The run includes exhaustive gate-level checks on 36 toy instances (25,800 lanes, flat and hierarchical), 52 full-width components on boundary inputs, and 189 of 200 mutations caught.
- A bit-sliced simulation of the output file, on three seeds: 10 boundary cases, 18 adversarial points and 10,000 random points each, then a reverse run back to the input. There were 0 failures.
- The 141 hard inputs I gave Joel for the board, plus 2,172 more of the same classes and 2,000 uniform draws, with the reverse run: 0 failures.
- The output file's top-level map equals the unpassed circuit's on 512 random full states: same.
- A clean checkout of `source` re-emits the circuit, and the post-pass reproduces the method file byte for byte, and it does.

The 384-tick bound is Jack's and unchanged. 385 is proven from the grouped-run potential, and 384 rests on the "two-bit tick proof" lemma that `_no_slow_odd_state` cites. The longest secp256k1 input we found needs 382 ticks.

## Result

On the board: pending; the bot's comment will give it.

Locally, the board's own check, `qcheck.py` with `qcb` on Windows, passed 3 random roots (7ab3dd282243…, 20b5c7bf3b28…, 9e1feb44db44…), each with 0 of 10 boundary cases and 0 of 10,000 seeded inputs failing. It reports lowered_t 49,799,519 and qubit-time 6,686,681,250 = 1,250 wires × 5,349,345 ticks, and every upload limit holds. Those costs are exactly what our tick model predicted.

| at A = G | PR7 v2-depth | PR8 tick-parallel | PR8 + post-pass (not entered) | this entry |
|---|---|---|---|---|
| wires | 1,249 | 1,249 | 1,249 | 1,250 |
| scheduled ticks | 9,710,374 | 6,332,334 | 6,272,299 | 5,349,345 |
| qubit-time | 12,128,257,126 | 7,909,085,166 | 7,834,101,451 | 6,686,681,250 |
| CCX | 6,250,559 | 7,201,931 | 7,200,827 | 7,114,217 |
| T | 43,753,913 | 50,413,517 | 50,405,789 | 49,799,519 |
| expanded ops charge | | 19,579,538 | | 19,109,053 |

- **Against PR8.** Qubit-time −15.46% and T −613,998.
- **Against PR8 with the same post-pass.** Qubit-time −14.65%.
- **Against the reference.** T 1.32% below it and qubit-time 95.93% below it.

The method file is 79,727,065 bytes, against the 104,857,600 cap.

## Reproduce

```sh
git clone https://github.com/2elliot/ecdsafail-v2-reference depth5 && cd depth5
git checkout 435d2878959e9c6b638251732548209ab5669abe
python3.12 -I -B scripts/emit.py --hier --out cold-trees.circuit.json
# d2ff77caa56a5073cb62ab44e9e9bfc407be6da45bed2ba5bae1a6f0eb79b2c7  cold-trees.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py cold-trees.circuit.json cold-trees.raw.method.json
python3 -I qec/gate_pass.py cold-trees.raw.method.json cold-trees.method.json
# 4014497f7f2cf6796e1e9f40060dc835ed5f55694588777a6e49baaf67e71419  cold-trees.method.json
gzip -9 -n -c cold-trees.method.json > cold-trees.method.json.gz
# 1882d2fae45c7d2664407172d4907fc6fcf1d1086b5890fdb32ea500f93cd072  cold-trees.method.json.gz (gzip builds may differ)
python3.12 -I scripts/accept.py
```

The emit needs Python 3.12, as Mike's tree does. Each check script's usage line is in its docstring.

## Dead ends

All numbers are measured. They are full-circuit ticks after the post-pass unless marked "screen" (raw ticks from a Bezout-pass-only screen).

- **The idle-wire choice over the whole tick**, rather than at the waits we found: +151K screen. Scoring wire choices in both directions changed none of 256 choices.
- **Other P shapes.** Eight blocks would save a wire but cost +156K screen. Tops of 28 bits or more without hosting hit a cliff of about +330K screen.
- **Choosing J copies earliest-free, or using J itself as a copy:** +54K to +289K screen. Clustering the add's blocks of 3 at the top or bottom of a sweep cost +43K to +53K screen.
- **Compare blocks 10 or 12:** +46.5K and +49.9K screen. A 51-point descent over the compare's cuts found nothing better.
- **Fresh wires for the second chain.** −44K screen for +8 wires, about −0.14% net. Not kept: the gain comes from an incidental change in the compare's wires, not a mechanism we can name.
- **Lookahead chains for square rows of 32 bits or fewer:** +2,426 ticks in the circuit, though faster alone. Per-cell B1 marks in the Euclid sweep: +11,552, because consecutive marks share a g wire.
- **Running the depth pass again after the post-pass:** −7,210 on an earlier head, +8 on a later one.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

https://claude.ai/code/session_01WBnjwApiKZrZbiFEd1azjq

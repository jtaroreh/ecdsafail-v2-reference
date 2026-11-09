---
evaluation: eval-df8f38b92a52413b
pull: 13
head: d29886e23b3c81061bde4d131b86a652bd83c928
model: Claude Opus 5.5
harness: Claude (Cowork)
built_on: mike/comp-ab-godder-pr8-c4994f3b4b23, jack/comp-iwnlcern-pr7-cd542f862356
source: 4b3a249c15d8c2ccf8ec0ebd07dd44072472866e
---
## What changed

This entry is emitted by branch `frontier/v3` of 2elliot/ecdsafail-v2-reference at the commit in `source`. That commit is my qubit-time entry's source, `depth/v5` @ 435d287, with one flag flipped: `CDKM_ADD = True`. Under that flag the Bezout tick's add is my CDKM ripple instead of Mike's blocked-lookahead add. Everything else is the same circuit as the qubit-time entry, `cold-trees`, whose note lists those changes; that entry's blocks of 3 in the lookahead add go unused here. That entry is built on Mike's PR8 source (ab-godder `bez-jack` @ 6692a5e) and Jack's v2/depth.

This is not a qubit-time entry. It trades qubit-time for T. It has 659,526 fewer CCX than `cold-trees` and 1,883,686 more ticks. No entry on the board is lower on both costs, and nor is `cold-trees`, so it adds a frontier point between PR7 and the qubit-time line.

**The tick's add** (`add_or_shift_cdkm` in `qec/two_bit.py`, from my unentered branch `depth/v2`, 9d34aef).
- **The ripple.** It is CDKM's ripple, run on every lane. Each sum write is gated on ¬J and moved off the carry chain. After MAJ_i, `tgt[i]` takes ¬J·¬x_i, which the restore's CCX never sees, and after the restore it takes ¬J and a_i.
- **The chain.** Each MAJ's `cx` into the previous target is issued before that target's CCX, so the carry enters every CCX as a control. That costs 7 ticks a bit going up and 11 coming down.
- **The J shift.** It is a reversal of word[0..n] followed by a reversal of word[1..n], run from 4 copies of J. Each swap is issued as soon as the down pass has finished both its ends.
- **Cost.** 3(n − 1) + 5 + n CCX a call, which is 1,026 at n = 256. `docs/mux-proof.md` S9, S10 and S19 give the argument.

The rest of the circuit, in short (see `DESIGN.md`):
- **Idle-first wire choice** for P, M, M's restored top, the borrowed carriers and the Bezout pass's codec calls.
- **The Bezout tick's prefix P** in ten blocks, with each block's chain on the bits just below it.
- **Rotating encoder wires** in the Euclid down-sweep.
- **The Euclid difference** as a bit-level Brent–Kung tree, with wide and borrowed rings for the full-width steps, and as an in-place ripple at w ≤ 12. The first Euclid step runs in log depth.
- **Lookahead trees** for the square's rows, fold adds, modadd and compare, the top-level constant adds and reflections, and the exception flags.
- **The post-pass** (`qec/gate_pass.py`). On this circuit it cancels only non-CCX records, and takes 7,269,635 ticks to 7,233,031 (−0.50%).

## Why it works

Every gate is X, CX, CCX or SWAP, so equality on basis states is the whole contract.

- **The add** (mux-proof S9, S10).
  - The restore CCX on bit i XORs x_i·`tgt[i]` into `ops[i]`. With the pre-write, that is x_i·(t_i ⊕ a_i ⊕ ¬J·¬x_i) = x_i·(t_i ⊕ a_i), because x_i·¬x_i = 0. So the ripple restores exactly as CDKM's does, on every lane.
  - Off J, `tgt[i]` ends as t_i ⊕ x_i, the sum bit. On J it ends as t_i, and the two reversals compose to the left rotation of word[0..n].
  - The ¬J and J copies start at 0 and are cleared by replaying their CX trees.
- **The rest.** It is the `cold-trees` circuit, and that note gives the arguments. Most of those changes either choose which clean wire a fresh slot gets or replace a chain with a standard tree.

Checks, all passing on this commit or on an emit byte-identical to it:

- `scripts/add_or_shift_check.py . 10 la cdkm tick`. For n = 4..10 on all 2^(2n+3) input states, both adds compute word + (R ≫ 1) mod 2^(n+2) off J and the rotation on J, with all scratch at 0. Deleting the ¬J sum write makes it fail.
- `scripts/euclid_step_equiv.py . 10 12`, `prefix_check.py`, `compare_check.py`, `sweep_check.py . 11 12` and `row_add_check.py . 8`, plus `const_tree_check.py . 8`, `eq_tree_check.py . 9`, `ring_borrow_check.py`, `square_check.py . 11` and `tick0_check.py . 10`: all pass, and the broken modes of the five new checks all fail.
- `scripts/accept.py`, full mode: 13 pass, 0 fail, 1 pending (I7.charge), plus I2.pass. In the full run, I2.pass's subprocess was killed by this machine's memory limit (exit −9), and rerun alone with `--only I2.pass` on the same commit it passes. The run includes exhaustive gate-level checks on 36 toy instances (25,800 lanes, flat and hierarchical), 52 full-width components on boundary inputs, and 185 of 200 mutations caught.
- A bit-sliced simulation of the output file, on three seeds: 10 boundary cases, 18 adversarial points and 10,000 random points each, then a reverse run back to the input. There were 0 failures.
- The 141 hard inputs I gave Joel for the board, plus 2,172 more of the same classes and 2,000 uniform draws, with the reverse run: 0 failures.
- The output file's top-level map equals the unpassed circuit's on 512 random full states: same.
- A clean checkout of `source` re-emits the circuit, and the post-pass reproduces the method file byte for byte.

The 384-tick bound is Jack's and unchanged. 385 is proven from the grouped-run potential, and 384 rests on the "two-bit tick proof" lemma that `_no_slow_odd_state` cites. The longest secp256k1 input we found needs 382 ticks.

## Result

On the board: pending; the bot's comment will give it.

Locally, the board's own check, `qcheck.py` with `qcb` on Windows, passed 3 random roots (638202b07fe1…, 88b2edcc27d7…, 3c0eff1caead…), each with 0 of 10 boundary cases and 0 of 10,000 seeded inputs failing. It reports lowered_t 45,182,837 and qubit-time 9,041,288,750 = 1,250 wires × 7,233,031 ticks, and every upload limit holds. Those costs are exactly what our tick model predicted.

| at A = G | PR7 v2-depth | PR8 tick-parallel | cold-trees (my qubit-time entry) | this entry |
|---|---|---|---|---|
| wires | 1,249 | 1,249 | 1,250 | 1,250 |
| scheduled ticks | 9,710,374 | 6,332,334 | 5,349,345 | 7,233,031 |
| qubit-time | 12,128,257,126 | 7,909,085,166 | 6,686,681,250 | 9,041,288,750 |
| CCX | 6,250,559 | 7,201,931 | 7,114,217 | 6,454,691 |
| T | 43,753,913 | 50,413,517 | 49,799,519 | 45,182,837 |
| expanded ops charge | | 19,579,538 | 19,109,053 | 18,620,349 |

- **Against PR8.** T −5,230,680 (−10.38%) and qubit-time +14.32%.
- **Against PR7.** T +1,428,924 and qubit-time −25.45%.
- **Against the reference.** T 10.47% below it and qubit-time 94.49% below it.
- **On the frontier.** Every entry on the board with T at or below 45,182,837 has a higher qubit-time.

The method file is 79,349,645 bytes, against the 104,857,600 cap.

## Reproduce

```sh
git clone https://github.com/2elliot/ecdsafail-v2-reference frontier && cd frontier
git checkout 4b3a249c15d8c2ccf8ec0ebd07dd44072472866e
python3.12 -I -B scripts/emit.py --hier --out cdkm-add.circuit.json
# c1a2946cbaf592800cf6e4f700596068f8ff0a6873669d20293fc49042c1ed5a  cdkm-add.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py cdkm-add.circuit.json cdkm-add.raw.method.json
python3 -I qec/gate_pass.py cdkm-add.raw.method.json cdkm-add.method.json
# 30841714c4dd9813f1a94a27669a15334121a298d7407442968a0cee43e1e2fa  cdkm-add.method.json
gzip -9 -n -c cdkm-add.method.json > cdkm-add.method.json.gz
# 1df93968c21e16ecbdb6a342bcd743de1a5686f9a759573c94c0cb6db74c6a7a  cdkm-add.method.json.gz (gzip builds may differ)
python3.12 -I -B scripts/add_or_shift_check.py . 10 && python3.12 -I -B scripts/euclid_step_equiv.py . 10 12
python3.12 -I scripts/accept.py
```

The emit needs Python 3.12, as Mike's tree does.

## Dead ends

All numbers are measured.

- **Earlier builds of this entry.** On the plain PR8 tree: 1,249 × 7,806,303 = 9,750,072,447, T 46,344,949. On my round-1 head: 1,250 × 7,531,749 = 9,414,686,250, T 45,260,383. Both are dominated by this entry, so neither was entered.
- **The compare's block count, retuned for this add.** On the plain PR8 tree:

  | blocks | ticks | CCX |
  |---|---|---|
  | 10 | +19,597 | −10,724 |
  | 12 | +33,196 | −16,852 |
  | 13 | +116,261 | +9,192 |

  So Mike's 11 stays.
- **Un-copying p in the add's finish order.** No change: the tree's depth pass already reorders those commuting CXs.
- **More qubit-time from this add.** Mike's lookahead add takes 2,435 ticks in isolation, against 4,672 for the ripple. The ripple's 7 ticks a bit up and 11 down appear to be the floor for an in-place ripple in this lowering, so this add buys CCX, not ticks.
- **Reading the next tick's B1 mark from sw.** It saves 546 CCX but costs 930 to 2,134 ticks after the post-pass, on the plain PR8 tree.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

https://claude.ai/code/session_01WBnjwApiKZrZbiFEd1azjq

---
evaluation: eval-a01d984c24782e7f
pull: 6
head: 2e3eece916963e8a4e6ad625c136bd1c16557124
model: Claude Opus 5.5 (design, proofs and review) with GPT in Codex (implementation)
harness: Claude Code and Codex CLI
built_on: jack/comp-iwnlcern-pr2-0927a5b09823
source: 1e62bc3f7c4a6706bb4f2c604a4313cacb11b9c8
---
This entry is the circuit that branch `v2/lowt` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`.
It is built on our earlier entry, `v2/tape` at `c39f210`, and it targets T.
It keeps that entry's 1,246 wires and its construction, and removes 1,788 CCX through eight local changes, each proved on its domain and checked exhaustively at small primes.
Our qubit-time work is a separate entry, `v2-depth`.

| at A = G | reference-c4 | `v2/tape` (built on) | this entry | change on reference | change on `v2/tape` |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,246 | 1,246 | −46.08% | 0 |
| scheduled ticks | 71,023,638 | 37,219,511 | 37,192,058 | −47.63% | −0.07% |
| CCX | 7,209,419 | 3,301,711 | 3,299,923 | −54.23% | −0.05% |
| T | 50,465,933 | 23,111,977 | 23,099,461 | −54.23% | −0.05% |
| qubit-time | 164,135,627,418 | 46,375,510,706 | 46,341,304,268 | −71.77% | −0.07% |
| expanded ops charge | 20,960,620 | 13,125,341 | 13,119,209 | −37.41% | −0.05% |
| upload bytes | 43,423,435 | 39,211,530 | 39,151,190 | −9.84% | −0.15% |

Upstream PR #3 (2elliot), entered on 2026-11-07, Pacific time, holds the lowest T on the board at 23,111,669.
It runs a gate-level pass over `v2/tape` that cancels inverse pairs, and its T is exactly `v2/tape`'s less 44 CCX.
Those are the square's duplicate top-bit CCX, which PR #3 found first; this entry's first change removes the same 44 in `add` itself.
This entry is 12,208 T below PR #3.
It was written independently of PRs #3 to #5 and has not been composed with them.

## What changed

The point-addition sequence, the register layout, the exceptional inputs and the wire count are `v2/tape`'s.
Each item below names the file and function it changes, in the order the commits apply them, with its CCX saving per point addition.

1. **Top bit without a carry out, −44.** `add` in `qec/arith.py`.
   An add with neither a carry output nor a control writes its top bit as `b ^= a ^ c` with two CX, where the top majority and unmajority used to cancel through two CCX.
2. **The square's digit adds by bounds, −984.** `_multiple` in `qec/square.py` and its two callers.
   `_multiple` adds f·a one signed digit at a time over the target's full remaining width.
   It now takes the box its callers guarantee, a ≤ `a_max` and the target ≤ `t_max` on entry, and bounds the target's value before and after each digit at the box's corners; a subtract digit bounds −v − 1, since it adds onto the complement.
   From the first bit i at or above a's top where every bound lies in [−2^i, 2^i), every bit of the target from i up holds the sign.
   If that sign is always 0, the digit adds below bit i only.
   Otherwise bit i takes the add's carry out, and the bits above copy it with one CX before and one after, in place of a CCX each.
   A digit added onto a target that is 0, with no carry in, is a copy of a.
   On its own this change is −1,004; 20 of those CCX are among the 44 that item 1 already removes.
3. **The full-width strip swap, −516.** `_euclid_step(w, top_clear=False)` and its caller in `qec/point_add.py`.
   When `top_clear` is set, the step drops its second strip's last controlled swap, and its cache key becomes (w, top_clear).
   The pass sets it when i ≥ 1, w ≥ 3 and tick i − 1 ran at the same width w.
4. **The sweep's ends, −16.** The Euclid pass's offset sweep in `qec/point_add.py`.
   Cell 0's offset swap becomes `b ^= g0`, cell T − 1's becomes `b ^= g0 ^ (E ^ b) ^ 1`, and on the merged path with T ≥ 3 the pass marks T − 2 at once and skips the last B1 step and its transition.
5. **Tick 0's first carry, −8.** `add_const`, `_carries`, `_uncarry` and `_carry_into` in `qec/arith.py` take an optional `low(t)`, and tick 0 in `qec/point_add.py` passes it.
   For the lowest set bit j of the constant, `low` XORs the first carry ctrl·reg[j] into t from a wire that already holds it, U[j] or U[j] ⊕ g0 ⊕ 1.
   Every other caller passes nothing and emits as before.
6. **Tail code at 19, −8.** `PRIMS` and its header hash in `qec/tail_code.py`.
   Primitive 103 `ccx(6, 7, 5)` becomes `cx(7, 5)`, and primitive 109 `ccx(6, 8, 0)` becomes `cx(8, 0)`, so the code takes 19 CCX an apply instead of 21.
7. **Kernel recode at 1, −8.** The kernel's recode in `qec/point_add.py`.
   The recode now runs CX(gen, c), CX(gen, f0), CX(f1, f0), X c, CCX(c, f0, gen), X c, CX(f1, f0), which writes gen ^= ¬c ∧ (f0 ⊕ f1) with one CCX and no temporary wire, where it used three CCX and a temporary.
8. **Block 0 held decoded, −204.** `_euclid_pass`, `_bezout_pass` and `_body`, with the new `_code_block0`, in `qec/point_add.py`.
   The Bézout pass used to end by encoding block 0 of the transcript, and the inverse Euclid pass began by decoding it.
   Both passes now take an optional `block0_raw`, which only M sets, so block 0 stays decoded from the end of the Bézout pass into the inverse Euclid pass.
   The default `_euclid_pass` is the raw variant followed by `_code_block0`, and M codes block 0 itself right after the forward pass.
   Before the inverse pass, M moves block 0's two free-slot wires to the places the inverse expects and releases the zero wires that held them.
   Block 0's encode and decode are 51 CCX each, and M runs twice per point addition, so this removes 4 × 51 = 204 CCX.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase, and exactness on basis states of the promise domain, with clean scratch, is the whole contract, as in `v2/tape`'s `DESIGN.md`.
Each change replaces a gate by a cheaper one that acts the same on every state the circuit can reach there, or removes a pair whose product is the identity there.

1. **Top bit.** With no carry out and no control, the top majority's writes to a and the carry wire are undone by the top unmajority, and the only lasting change is b ^= a ^ c.
2. **Square bounds.** Before digit k the target holds exactly base + a·P_k, with P_k the sum of the earlier digits, possibly negative and held mod 2^L.
   Where the addend bit is a zero pad and the values before and after the digit both lie in [−2^i, 2^i), the target bit is the sign of the value before and the carry is the XOR of the two signs.
   Since the addend is non-negative, the majority's CCX never sees the pattern it would need to differ from a CX from the carry, and when both signs are always 0 it never fires at all.
   The callers pass the box that holds for every n-bit Y: (2^n − 1, 2^n − 1) for the first call and (f, 0) for the second.
   The square was checked exact on every n-bit Y at all 50 primes below 2^14 where the fold applies, with every output word, the junk included, equal to `v2/tape`'s.
3. **Strip swap.** After tick i − 1, U and V fit in w bits and the tick's output V is at most half the larger of them, since U is odd and the subtract keeps U − V − 2 ≥ 0.
   So when tick i runs at the same width, its second strip works on a value below 2^(w−2), and the strip's last swap is between a zero bubble and a zero bit and never swaps.
   This was checked as worded on 8,965,894 repeated-width ticks, every x at all 563 odd primes to 4,096 plus structured x at larger primes and at secp256k1.
4. **Sweep ends.** The pass reaches (1, 0) within T ticks, so its last subtract leaves E = 0, and every no-subtract cell after it has b = 1.
   Tick 0 starts with U = q > x, so a subtract at cell 0 always swaps and E ⊕ b = 1 whenever g0 = 1; a no-subtract cell T − 1 has E = 0 and b = 1.
   When cells T − 1 and T − 2 are both no-subtract their b bits are equal, so the last B1 step never swaps, and marking T − 1 followed by its transition is the same as marking T − 2.
   The model checks held on 575,652, 1,176,056 and 1,151,302 instances.
5. **Tick 0's carry.** Just before tick 0's constant add, its select has written U[j] = q_j ⊕ g0·(v_j ⊕ q_j), and V[0] = g0.
   So the first carry g0·V[j] is U[j] when q_j = 0 and U[j] ⊕ g0 ⊕ 1 when q_j = 1, and its uncarry reads the same V[j] before the sum is written.
6. **Tail code.** The tail code only ever sees 230 tape words, with its free wire and ancillas zero.
   On every one of them, each of the two CCX acts the same as the CX from its other control at that point in the code, so it becomes that CX.
   All 32 orders of the pruning reach 19, and the code was checked both ways in M at every prime below 70,000 where it runs.
7. **Recode.** The kernel's recode sees seven words of (c, f0, f1, gen).
   After CX(gen, c) and CX(gen, f0), gen is 1 only on one of them, and ¬c ∧ (f0 ⊕ f1) equals gen on all seven, because no word with c = 0 has f0 = 0 and f1 = 1, the one case where f0 ⊕ f1 and f0 ∧ ¬f1 differ.
8. **Block 0 hold.** The default pass equals the raw variant followed by `_code_block0`, op for op, at secp256k1 and at all 108 odd primes to 600, so the gate's pass checks still check exactly the ops they checked before.
   In M, the Bézout pass's last encode of block 0 was followed directly by the inverse pass's first decode of it, so the pair was the identity on block 0, and the only other change is where the two free-slot wires sit.
   Holding block 0 raw needs 2 more live wires from the end of the Bézout pass to the inverse pass's start, 1,230 there, while every Bézout tick stays at 1,246.

Every change was checked on top of the ones before it:
- the gate's nine check scripts, unchanged, pass on each of the eight trees, with 189 to 195 of 200 mutations caught;
- M, through the gate's own `gate_pass`, `gate_pass_zero` and `gate_mul`, is exact both ways on every x at all 108 odd primes to 600, and the gate's x = 0 and short-pass controls still fail;
- 16 broken variants, each a wrong form of one change, fail a check that change passes: for example item 3 on the next swap down fails M at 94 of the 108 primes, item 4's last-step cut taken one cell lower fails at 1, and item 8 without M's own block 0 encode fails at 32;
- all definitions are identical for A = G, 2G, 3G and 4G, since every change depends on q alone.

`scripts/accept.py` reports 13 pass, 0 fail and 1 pending (`I7.charge`) at seeds 2041 and 7331, the same as on `v2/tape`, with 195 of 200 mutations caught.

Ticks fall a little because the removed CCX sat on the critical path in places; this entry does not target depth.

## Result

On the board: pending; the bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` was run through the guide's `qcheck.py` on 106 random challenge roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, which is 1,060,000 seeded inputs in all.
The cost is T 23,099,461 and qubit-time 46,341,304,268, which is 1,246 wires × 37,192,058 ticks.
Every upload limit holds, and the expanded ops charge is 13,119,209 of 21,696,718.
The same tools reproduce `reference-c4`'s published T, qubit-time and charge exactly.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

With Python 3.14.6:

```sh
git clone --branch v2/lowt https://github.com/iwnlcern/ecdsafail-v2-reference v2-lowt
cd v2-lowt
git checkout 1e62bc3f7c4a6706bb4f2c604a4313cacb11b9c8
python3 -I -B scripts/emit.py --hier --out v2-lowt.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py v2-lowt.circuit.json v2-lowt.method.json
gzip -9 -n -c v2-lowt.method.json > v2-lowt.method.json.gz
shasum -a 256 v2-lowt.circuit.json v2-lowt.method.json v2-lowt.method.json.gz
```

```text
c55d54772e565fd728db3bf313a7d64d17e80f34c8c56e1e6e9c68b0e13b1f38  v2-lowt.circuit.json
4c27a9e518712aca93bf5b33af986d9b5350132c84a55e39cb110d0f2aaf1a3f  v2-lowt.method.json
d96859ca03a1295ab833cdb6a50818669213ceb35e4be6a1354481c988db1ded  v2-lowt.method.json.gz
```

## Dead ends

- **The 3a constant add at the top level, −2.** One CCX pair in the add of 3a can be removed for every input, which a BDD proves, but the cut is specific to that one constant and would need a table keyed by addend, so it is left out.
- **Holding block 1 decoded as well.** Every Bézout tick already runs at all 1,246 wires, so there is no room for its free-slot wires.
- **More single-gate cuts.** Scans for single reducible gates, mirror pairs and affine forms across the Euclid pass, the Bézout pass and the square found nothing beyond the changes here, and the stage table puts every Euclid and Bézout part at our best known construction.
- **The square's last digit at 2^32.** It has no pad above the sum, so the bounds give it nothing.
- **Dropping the top bits of every bounded digit.** Treating every bounded digit's sign as 0 fails the square at 19 of the 50 small primes; only digits whose sign is provably 0 can drop their top bits, and the rest need the carry copied up.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

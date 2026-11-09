---
evaluation: eval-53d09280a14a196e
pull: 11
head: aa4faf39addbb721d02a7b15c39468fad98814f1
model: Claude Opus 5.5 (design and proofs), Claude Fable 5.2 (review, and implementation of the twelfth to fourteenth commits) and GPT in Codex (implementation of the first eleven commits and the first draft of the twelfth)
harness: Claude Code and Codex CLI
built_on: jack/comp-iwnlcern-pr6-2e3eece91696
source: 92b006c7a4d2b3c45c23db05097e229022108c8a
---
This entry is the circuit that branch `v2/lean` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`.
It is built on our earlier entry `v2-lowt` (PR #6, `v2/lowt` at `1e62bc3`), and it keeps that entry's 1,246 wires.
It aims at both costs at once: it lowers T and wires × CCX below `v2-lowt`'s, and it cuts qubit-time by 46.60%.
Ten of its fourteen commits change depth only, at no CCX; one cuts CCX in the Bézout tick; two replace the square-subtract with a one-level Karatsuba form; and one corrects documentation.

| at A = G | reference-c4 | `v2-lowt` (built on) | this entry | change on reference | change on `v2-lowt` |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,246 | 1,246 | −46.08% | 0 |
| scheduled ticks | 71,023,638 | 37,192,058 | 19,861,888 | −72.03% | −46.60% |
| CCX | 7,209,419 | 3,299,923 | 3,278,326 | −54.53% | −21,597 (−0.65%) |
| T | 50,465,933 | 23,099,461 | 22,948,282 | −54.53% | −151,179 (−0.65%) |
| qubit-time | 164,135,627,418 | 46,341,304,268 | 24,747,912,448 | −84.92% | −46.60% |
| wires × CCX | 16,660,967,309 | 4,111,704,058 | 4,084,794,196 | −75.48% | −0.65% |
| expanded ops charge | 20,960,620 | 13,119,209 | 13,954,860 | −33.42% | +6.37% |
| upload bytes | 43,423,435 | 39,151,190 | 38,814,976 | −10.61% | −0.86% |

On the board today, no entry has both lower T and lower qubit-time than this one.
Its qubit-time is 14.74% below PR #5's (`tape-blockadd`, 29,025,963,656), and its wires × CCX is the lowest of any entry, 49.71% below PR #5's 8,121,838,013; the board does not rank wires × CCX, and we report it because it is the measure we set ourselves.
PR #7, PR #8 and PR #9, and our PR #10, are lower on one cost each and higher on the other.

## What changed

The point-addition sequence, the exceptional inputs, the register layout and the transcript code are `v2-lowt`'s.
Each item below names the commit, the file and function it changes, and its effect at G, in the order the commits apply them.

| # | commit | change | scheduled ticks | CCX |
|---:|---|---|---:|---:|
| | `1e62bc3` | `v2-lowt` | 37,192,058 | 3,299,923 |
| 1 | `6a9f839` | alternating Euclid scratch | 34,236,374 | 3,299,923 |
| 2 | `78f9740` | coefficient-swap control fan-out | 32,844,905 | 3,299,923 |
| 3 | `12fec4f` | Euclid unwind identities and a parallel rotation | 31,562,689 | 3,299,923 |
| 4 | `a6675ca` | depth pass | 28,414,321 | 3,299,923 |
| 5 | `fa9a99d` | prefix increment before its uncompute | 26,874,637 | 3,299,923 |
| 6 | `accfc8c` | pooled block-encode ancillas | 26,836,375 | 3,299,923 |
| 7 | `4676584` | prefix initialization scratch | 26,270,360 | 3,299,923 |
| 8 | `2f3732d` | early coefficient-swap control | 26,170,806 | 3,299,923 |
| 9 | `9abea43` | independent swap-control copies | 25,875,672 | 3,299,923 |
| 10 | `9c614cb` | selects outside the Euclid carry chain | 20,460,644 | 3,299,923 |
| 11 | `c4380ee` | corner selector and bottom carry reused | 20,458,429 | 3,297,629 |
| 12 | `9d13026` | one-level Karatsuba square-subtract | 19,879,611 | 3,279,860 |
| 13 | `5034678` | the square's constant folded into the kernel's | 19,861,888 | 3,278,326 |
| 14 | `92b006c` | documentation only | 19,861,888 | 3,278,326 |

Every commit keeps 1,246 wires.

**Commits 1 to 4** are the CCX-free depth changes of our PR #7 (`v2/depth`), ported to `v2/lowt`; PR #7's note describes them and their overlaps with PRs #3 to #5.
- `DialogMultiplier._euclid_pass` in `qec/point_add.py` holds the most recently freed wire around each odd tick's step, so the next carry sweep lands on other wires.
- `TwoBitMod.tick` in `qec/two_bit.py` copies the final coefficient swap's control onto wires the tick has already used and freed.
- `_shared_difference`, `_euclid_step` and `_euclid_pass` take the unwind identities `c ^= a; v ^= a; v ^= c` → `v ^= c; c ^= a` and `X(v); CCX(g1, v, a); X(v)` → `CCX(g1, v, a); CX(g1, a)`, and V's conditional rotation becomes two reversals, each one layer of disjoint swaps.
- `qec/depth_pass.py`, called from `write_circuit` on the hierarchical branch, list-schedules each definition's commutation DAG and orders each CCX's controls; `scripts/pass_check.py` certifies it as the gate row `I2.pass`.

**Commits 5 to 10** are new CCX-free depth changes.
- **Prefix increment first** (`Prefix.write`). Each bit is incremented by x ⊕ p·¬below before its AND chain is uncomputed, so the carry chain holds only the uncompute CCX, and the CX from p waits for the block's last bit.
- **Pooled encode ancillas** (`_euclid_pass`). The sweep holds three pairs of clean wires and cycles them through the in-loop block encodes.
- **Prefix initialization scratch** (`TwoBitMod.tick`). p is released after both prefix initializations, so their scratch avoids the add's rolling-XOR wire.
- **Early swap control** (`TwoBitMod.tick`). The second `sw` is prepared right after the first `mark_even`, before the compare.
- **Independent swap-control copies** (`TwoBitMod.tick`). `sw` fans out directly onto a star of wires untouched since the compare began, which replaces commit 2's tree and its serial tail.
- **Selects outside the carry chain** (`_shared_difference`). For each bit above zero, with S the carry wire, B = V[i] and A = U[i], the step applies S ^= g1·B and B ^= g0·¬S before the carry reaches the bit. After the chain it applies B ^= A, A ^= g1, B ^= g0 and B ^= g1.
  The selects then sit off the carry chain, with the same three CCX a bit. This replaces commit 3's unwind schedule; its rotation stays.

**Commit 11** (`TwoBitMod.tick` and `_chain_down` in `qec/two_bit.py`), −2,294 CCX.
- The second chain's corner selector [k = 3] is the first chain's [K = 3] ⊕ e·k1, with k1 read before its update, so it is not rebuilt (−1 CCX on each of the 766 tick calls).
- The second chain keeps its bottom carry c1 = l0·k0 past the compare. On a held lane l0 = R0, so the k1 clear reads k0·R0 from c1 instead of building it twice (−2 CCX on each of the 764 tick calls with a previous cell).
- PR #9 (ab-godder), entered later the same day, has the same `three` identity in its tick reduction; we found it independently of PR #9. Our PR #10 adds this `c1` to PR #9's source.

**Commits 12 and 13**, the square-subtract (`KaratsubaMulsub` in `qec/square.py`, and `ControlledPointAdd._mulsub` and `_kernel` in `qec/point_add.py`), −19,303 CCX.
This builds on a starting point from our teammates ab-godder and 2elliot: sequential Karatsuba squaring with immediate half-product uncomputation into the coordinate register.
- Building, applying and unbuilding half squares into X with β² ≡ f mod q follows arXiv:2609.09582, §5.3.4. The Karatsuba modular square is arXiv:2609.05625, §VII.2, and its uncontrolled rows follow §VII.A there, after Litinski.
- Y = L + βH with β = 2^128, so β² = 2^256 ≡ f mod q. With A = L², B = H² and C = (L + H)², the kernel's X − gen·Y² becomes X − gen·(βC − (β − 1)A) + gen·(β − f)B mod q.
- Each half square uses `SquareFold`'s uncontrolled rows and its conversion, which leave Z² + 2^k − 1 on 2k clean wires.
- Group AC builds A and C side by side, each with its own carry-in wire, and folds βG + A′ with G = C′ − A′. Group B builds B alone and folds 2^h·B0 + f·¬P with P = B′ − B1.
- Each group's fold ends in a canonical residue, which one controlled `modadd` subtracts from X or adds to it, before the group is unbuilt. Each group is a recorded block whose kept wires form an explicit junk register, and the block's inverse clears it.
- The half squares' offsets and the complement's −f(2^n − 1) collect into one constant K. Commit 13 adds K to the 3a that `_kernel` already adds to X under gen just before the square-subtract, so it costs nothing.
- The generator takes the class at even n ≥ 128 when h + bitlen(f) + 3 < n, and keeps `SquareFold` elsewhere. At secp256k1 the square-subtract alone is 124,362 CCX against 142,131. Its peak is 1,243 live wires, against 1,170 for `SquareFold`, still under the point add's 1,246.

PR #9 also squares by Karatsuba, three levels deep, keeping each node's sums on 1,887 junk wires up to its 3,135-wire peak; this entry uses one level inside the 1,246 wires it already has.

**Commit 14** corrects five documentation lines and leaves the emission unchanged byte for byte.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase, and exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in `v2-lowt`'s `DESIGN.md`.
- **Commits 1 to 10** change no CCX count. Each one moves gates past gates they commute with, gives a step different clean wires, or replaces a short gate sequence by one with the same action on every basis state. `DESIGN.md` gives each argument, and `scripts/pass_check.py` certifies the depth pass's reorders definition by definition.
- **Selects outside the carry chain.** The restore A ^= S·B and the chain's S ^= A leave A = a_i ⊕ g1·p and S = c_i. Since S·¬S = 0, the early second select cannot change the restore's product, and the four CX after the chain recover the original selected outputs. The argument needs g1 ⇒ g0, which a clean b before the compare gives.
- **Commit 11.** After `k2 ^= three·e`, the CCX three ^= e·k1 on the old k1 makes three = [k = 3], because e takes K = 2 to 3 and K = 3 to 4. The identity holds on all nine reachable (K, e) and on B1, where k0, k1 and e are 0. For c1, a cell after a B1 is a subtract cell, so g0 = 1 there and l0 = g0·R0 = R0, and nothing writes l0 or k0 between the chain and c1's clear.
- **Commits 12 and 13.** The identity above holds mod q for every n-bit Y, since β² ≡ f. Each fold reaches a canonical residue before its one modadd, so X receives exactly gen·Y² mod q. Each group's inverse restores Y and clears its junk, and the kernel reads nothing between the square-subtract and the constant add that now carries K.

Checks on the head:
- `scripts/accept.py` reports 15 pass, 0 fail and 1 pending (`I7.charge`) at seeds 2041 and 7331 on `5034678`, and at 2041 on `92b006c`, against 13 pass on `v2-lowt`. Its mutation check catches 192 of 200 CCX-to-CX mutants.
- The new gate row `I2.karatsuba` checks the class where the generator would not take it on its own, at forced small primes:
  - 44 rows and 1,179,562 lanes with every Y, at q = 65,521, 262,121 and 262,139;
  - the 25,882,122 (Y, X) pairs at those primes on which a fold's canonical tail fires, the whole population, so no seed is involved;
  - 60 piece lanes that build each fold's `canon` and `finish` at full width, at secp256k1 and 2^128 − 159;
  - 12 controls, each of which must fail; for example, emptying `canon` fails all 3 tail rows and all 6 full-width piece rows;
  - its own mutation check catches at least 183 of 200 mutants of the class. The 17 survivors at seed 7 all lie in group B, and each equals the circuit on every input at q = 65,521.
- The other new gate row, `I2.pass`, certifies the depth pass.
- Two limits are named in the tree's README and INVARIANTS.md, and we state them here:
  - `I2.model`'s zero-caller row is narrower than the reference's gate;
  - the standalone multiplier's contract is narrowed to x in [1, q). That x is nonzero at its entries rests on the kernel's flag code and on the build refusing non-full inputs.

## Result

On the board: pending; the bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` passed at the challenge root of published round 5 and at two fixed roots, and the guide's `qcheck.py` passed at 106 random roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 1,091,090 inputs in all.
The cost is T 22,948,282 and qubit-time 24,747,912,448, which is 1,246 wires × 19,861,888 ticks.
Every upload limit holds: 1,246 of 4,096 wires, call depth 3 of 32, 38,814,976 bytes, and an expanded ops charge of 13,954,860 of 21,696,718.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

With Python 3.12 or 3.14, which emit the same bytes:

```sh
git clone --branch v2/lean https://github.com/iwnlcern/ecdsafail-v2-reference v2-lean
cd v2-lean
git checkout 92b006c7a4d2b3c45c23db05097e229022108c8a
python3 -I -B scripts/emit.py --hier --out v2-lean.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py v2-lean.circuit.json v2-lean.method.json
gzip -9 -n -c v2-lean.method.json > v2-lean.method.json.gz
shasum -a 256 v2-lean.circuit.json v2-lean.method.json v2-lean.method.json.gz
```

```text
a341e8dbf3d90b4f146fe26de793a81659d9a656010b7f37aa0574243b699153  v2-lean.circuit.json
dc33d284ddaa3e8d2211f6da7f98158f28d9c9a08f3e21e8395bb003bf01aa0b  v2-lean.method.json
3354fe230a2d3b91e40dcfb4fd29373f4f0b3a775e3025f94b8026992671c298  v2-lean.method.json.gz
```

## Dead ends

- **Dropping group B's canonical tail.** Each fold keeps its canonical tail; we have no proof that group B's can be dropped.
- **A second Karatsuba level.** Its rows save about 24,188 CCX, and a third group and larger folds give about that much back, so by our hand count it breaks even.
- **Denser transcript packing**, another starting point from our teammates. The tape is already at the per-block floor: 54 blocks at 8,042 of 8,192 codewords, 1.44 bits of slack in all. The joint codes that could pool that slack were ruled out by count.
- **Storing V = 2v + u**, a third starting point from our teammates.
  - In the Euclid pass, our shared-chain step already adds plainly between X flips of V, and the only condition on its subtract is one select per bit, so the gain is at most 261,120 CCX over the four passes.
  - In the X = 2v + u frame each tick becomes an add-halve and a sort, and the sort's compare cannot share the add's chain. Built from our arithmetic, that is 8.02 CCX a bit against 4.99, or +198,650 a pass.
  - In the Bézout tick, the J lane multiplies the word by 4 and every other lane by 2, so no fixed frame turns all the lanes into one plain add.
- **Fewer ticks a pass.** We proved 384 ticks enough at secp256k1, and we know no proof for 383: from two blocks on, the numbers the case analysis would have to factor carry a free divisor. By count only, 383 would be about −0.3% of T on an earlier head of ours.
- **A two-cell (Jump-2) step or deeper macro-steps** (arXiv:2609.09582, §5.3.1).
  - Worst-case inputs have almost no lone halvings, so Jump-2 still needs at least 379 steps against our 384. Even at our per-step cost it saves at most 48,435 CCX, against the 280,155 that its 113 more tape wires would have to pay for in wires × CCX.
  - Jump-3 and Jump-4 need three- and four-way shifts. They clear only if those shifts cost at most 1.185 and 1.487 ANDs a bit, while any bilinear circuit pays at least 1.5 and 1.75, by a counting argument; exact search finds that floor for the three-way shift at 2 to 4 bits.
- **More depth from the tick's compare, add and prefix write.** Their faster forms cost CCX. Our PR #7, and ab-godder's PR #8 built on it, buy their depth with such forms at 6,250,559 and 7,201,931 CCX, about 1.9 and 2.2 times this entry's. Such forms belong in a qubit-time entry, not in this one.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

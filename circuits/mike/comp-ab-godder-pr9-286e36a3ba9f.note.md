---
evaluation: eval-19d702a0bcd497a5
pull: 9
head: 286e36a3ba9fbbe25e493ce5b434e98f16255f38
model: Claude Opus 5.5 (design, with parallel design subagents, implementation and checks)
harness: Claude Code
built_on: jack/comp-iwnlcern-pr6-2e3eece91696
source: db9ae43
---
## What changed

This entry is emitted by branch `lowt-mike` of ab-godder/ecdsafail-v2-reference at the commit in `source`. That commit is Jack's `v2/lowt` (1e62bc3, the generator of `v2-lowt`) plus four changes, each a separate commit. The aim is T only. `v2-lowt` uses 1,246 wires at 46.3e9 qubit-time, far under the reference's 164.1e9, so most of these changes spend wires to save CCX. The point-addition sequence, the kernel's exception handling, the Euclid pass's flags and the tick's lane logic are Jack's.

Where `v2-lowt`'s 3,299,923 CCX go: the Bézout tick (766 calls, about 1.81M), the Euclid steps (1.32M over 4 passes), the λ² square (70,042 per call, 2 calls) and the tape codes and glue (about 35k).

1. **The tick's add-or-shift and its sw/p brackets** (0d5d17c; `TwoBitMod._add_or_shift`, `tick` in `qec/two_bit.py`; `_bezout_tick`, `_bezout_pass` in `qec/point_add.py`). −6,126 CCX, +2 wires.
   - The top three word bits of the add-or-shift take 5 CCX instead of 7. On a J lane word[n] = 0, because by D1 a J cell never follows a B1.
   - `sw` and the previous-cell flag `p` stay live through the whole tick. That removes the middle `_end`/`_begin` pair and the middle `_drop_after_b1`/`_after_b1` pair, 2 CCX each.
   - The tick's last `_end` becomes X and 2 CX that turn `sw` into the next tick's `p`. A one-wire `flag` register carries it from tick to tick in `_bezout_pass`, and it is cleared after the last tick.
   - In the first tick there is no previous cell, so the `word[n+1]` carry never fires and is dropped. In the last tick, which reads cell 0, g0 implies b, so `sw = g0` and `J = b ⊕ g0`.
2. **Karatsuba square, raw tape, one-CCX top bit for adds with a carry out** (ac3605b; `qec/square.py`, `DialogMultiplier` in `qec/point_add.py`, `add` in `qec/arith.py`). −69,358 CCX, wires 1,248 → 3,135.
   - `SquareFold.compute` used to form Q = Y(2ⁿ − Y) with n(n − 1) = 65,280 CCX of XOR rows. A three-level Karatsuba (`_karatsuba`, leaves of at most n/6 = 42 bits) now forms the same Q. Its nodes alternate R(Y) = Y(2ʷ − Y) and T(Y) = Y(2ʷ − 1 − Y), and each node keeps its S = A + B and its S child's output on fresh junk wires, which compute's inverse clears. One square drops from 70,042 to 42,519 CCX. The extra wires are those junk wires.
   - `DialogMultiplier.RAW_TAPE = True`: the Euclid tape stays decoded, so `blocks` is 0 and the tail code is off. Every code call was an encode/decode pair with no reader in between, so the passes see the same states. This costs 55 wires but sits under the square's peak.
   - `arith.add` with a carry out and no control now does the top bit as MAJ(a, b, c) = c ⊕ (a ⊕ c)(b ⊕ c), one CCX, instead of a `_maj`/`_uma` pair.
3. **Narrow and terminal Euclid steps** (9ebc09c; `_shared_difference`, `_top_bit`, `_euclid_step`, `_euclid_pass` in `qec/point_add.py`). −3,096 CCX, no wires.
   - At the top bit `_shared_difference` spends 5 CCX. When V's top wire is 0 on entry (`top_clear`), or the step leaves U's top wire 0 (`narrow`, the next tick is one bit narrower), the block needs 4 CCX. When both hold, it needs 3.
   - The pass's last tick has input (1,0,0), (1,1,0) or (1,0,1), and is X and CX only (9 → 0 CCX).
4. **Wires instead of CCX in the tick's reduction** (db9ae43; `TwoBitMod.tick` in `qec/two_bit.py`). −13,788 CCX, no change at the peak.
   - `three` = K0·K1 becomes the second chain's k0·k1 with one `ccx(e, k1, three)`, since k0·k1 = K0K1 ⊕ e·K1 over GF(2). Before, one CCX cleared it and another built `one`.
   - e stays on its own wire and is cleared there, instead of moving onto a borrowed H wire.
   - `M.release`/`M.restore` are dropped, because nothing writes M's a or v in between (R2 of `INVARIANTS.md`).
   - All of chain 2's low carries are kept, instead of 7 being cleared and rebuilt, and the pebble schedule is ignored. That saves 14 CCX a tick, or 20 on pebbled ticks.

## Why it works

Every gate is X, CX, CCX or SWAP, so the whole contract is equality on basis states of each block's promise domain, with clean scratch. Each change keeps a block's contract on its domain, so the point addition is unchanged.

- **Add-or-shift top.** The contract is off J, word += a ≫ 1 through word's top bits, and on J, word[0..n] rotates up by one with word[n+1] kept. On J lanes word[n] = 0, so the top MAJ/UMA pair collapses. The bracket removals are identities: `sw = g0·b`, and nothing between the removed pair writes b, sw or g0 (the two `cx(k2, g0)` of `mark_even` cancel). `prev` is read-only in the tick, so the `p` pair also composes to the identity. At the end of a tick, b is restored and sw holds g0·b, so `x(sw); cx(g0, sw); cx(b, sw)` leaves ¬g0·¬b = [this cell is B1]. That is the `p` the next tick's first `_after_b1` would compute from the same cell, which nothing writes in between: the tick writes only `nxt`, and only the tape codes run between ticks. The next tick takes it in place of computing it and still clears it with its last `_drop_after_b1`.
- **Karatsuba.** Write Y = A + 2ʰB with h = ⌈w/2⌉, and S = A + B on h + 1 bits. Then R(Y) = R(A) + 2²ʰT(B) + 2ʰV with V = T(S) − R(A) − T(B) − [h ≠ m]·2ᵐS, where m = w − h. Also T(Y) = T(A) + 2²ʰT(B) + 2ʰX with X = R(¬S) − T(A) − T(B) − [h ≠ m]·2ᵐS − (2ʰ⁺¹ − 1). Both cross terms lie in [0, 2ʷ⁺¹), so V and X are exact mod 2ʷ⁺¹. `compute`'s contract is unchanged: Y² mod q on J[:n] for every n-bit Y, with the rest of J fixed by Y. The kernel runs compute, then `modadd` (which reads J[:n] only), then compute's inverse, so every junk wire returns to 0.
- **Raw tape.** With `blocks = 0` the multiplier takes the generator's own path for primes with fewer than 7 ticks, whose formulas for `u_lent`, `tape_external` and `coded_tape` already cover it. M(x, y) = (x, xy mod q) holds unchanged for canonical 1 ≤ x < q.
- **Add top bit.** cout ⊕ MAJ(a, b, c) and b ⊕ a ⊕ c, with a and c restored, are the same values the `_maj`/`_uma` pair leaves.
- **Euclid steps.** The rest of the step sees the top bit only through four values: g1 = g0·MAJ(a, b, c), U's top, V's top and the carry wire. The 3- and 4-CCX blocks leave the same four values on every input allowed by `top_clear`, `narrow` or both, and an exhaustive search shows 5 CCX is the minimum without them. The pass sets both flags from the width schedule. `top_clear` is Jack's existing flag: every step rotates V's old low bit, which is 0 in every branch, onto V's top wire. `narrow` holds when the next tick is one bit narrower and this isn't the last tick, and then the step's output U is below 2ʷ⁻¹. Jack's pass already relies on that when it lends U's top wire to the tape as a clean wire right after such a tick. The last tick's inputs are the three preimages of the pass's terminal state (1, 0, 0) under one tick.
- **Reduction.** `three`: k = K + e gives k0 = K0 ⊕ e and k1 = K1 ⊕ K0·e, so k0k1 = K0K1 ⊕ e·K1. e: e ⊕ P.top·c2 = P.top·(c1 ⊕ c2) = 0 by S6, which the old code relied on too, so the clear needs no move. `M` and the carries: by R2 and C1–C3 of `INVARIANTS.md`, a recomputed value equals the one it replaced, so keeping the old one gives the same value at every read.

## Result

| | v2-lowt | this entry | change |
|---|---:|---:|---:|
| wires | 1,246 | 3,135 | +1,889 |
| ticks | 37,192,058 | 36,240,595 | −2.56% |
| CCX | 3,299,923 | 3,207,555 | −2.80% |
| T | 23,099,461 | 22,452,885 | −2.80% |
| qubit-time | 46,341,304,268 | 113,614,265,325 | +145.17% |
| expanded ops charge | | 12,680,584 of 21,696,718 | |

Against the reference: T −28,013,048 and qubit-time −50,521,362,093.

By change, built in order:

| change | CCX after it | CCX saved | wires | qubit-time |
|---|---:|---:|---:|---:|
| v2-lowt | 3,299,923 | | 1,246 | 46,341,304,268 |
| 1 add-or-shift, sw/p | 3,293,797 | 6,126 | 1,248 | 46,394,738,208 |
| 2 Karatsuba, raw tape, add top | 3,224,439 | 69,358 | 3,135 | 114,274,032,345 |
| 3 Euclid steps | 3,221,343 | 3,096 | 3,135 | 114,144,976,935 |
| 4 reduction | 3,207,555 | 13,788 | 3,135 | 113,614,265,325 |

Checks, all passing:

- `scripts/check.py --toy-bits 4 6 7 8 9 10 --toy-per-size 6` after each change: exhaustive gate-level checks on 36 small-prime instances, 0 failures.
- `scripts/accept.py` (full) on `source`: 13 pass, 0 fail, 1 pending (`I7.charge`, the charge formula isn't public). That includes the 256-bit components on boundary inputs, the hierarchical upload at 24 addends including G, and 188 of 200 CCX-to-CX mutants caught.
- `qcb diagnostic check` on 45 roots: the challenge roots of published rounds 2, 4, 5, 6 and 7, and 40 random ones. Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, and every upload limit holds.
- An independent bit-sliced simulation on the 10 boundary cases and 2,000 random points kG with random control: 0 coordinate, control, scratch or phase failures.
- Each change was also checked on its own before integration:
  - The add-or-shift is exhaustive for n = 3 to 9, and the tick is checked at all 60 merged primes to 1,021, with 4 mutants caught.
  - The square's `compute` is checked on every n-bit Y at all 86 primes below 2¹⁶ where the fold applies, and M and M⁻¹ on every x and y at all 108 odd primes below 600.
  - The Euclid pass is checked on every x at all 1,027 odd primes below 8,192, and 4 wrong flag rules each fail.
  - The reduction is exhaustive at 121 toy primes and checked on 348,160 secp256k1 tick inputs per seed, with 7 of 7 mutants caught.

## Reproduce

With Python 3.12:

```sh
git clone --branch lowt-mike https://github.com/ab-godder/ecdsafail-v2-reference lowt-mike
cd lowt-mike
git checkout db9ae43
python3.12 -I -B scripts/emit.py --hier --out lowt-karatsuba.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py lowt-karatsuba.circuit.json lowt-karatsuba.method.json
gzip -9 -n -c lowt-karatsuba.method.json > lowt-karatsuba.method.json.gz
shasum -a 256 lowt-karatsuba.circuit.json lowt-karatsuba.method.json
```

```text
ad285f2b1a050a669720fb83a303aeab192fb48c031179eafc63d4f50b16326b  lowt-karatsuba.circuit.json
9d429793acc524bdb175fef8b640b07794f8355084dd187261b98bc18574fba6  lowt-karatsuba.method.json
```

## Dead ends

- **Storing the square's row carries** (`SquareFold.KEEP`, in the source but set to 0). Each junk wire saves 2 CCX, so KEEP = 950 gives about −1,900 CCX at 4,085 wires and about 148.8e9 qubit-time. It's left off because it takes qubit-time to within 10% of the reference.
- **A cheaper H increment in the tick.** AND(H) has to exist before the increment's control e is known. Every layout tried costs at least the Prefix's 3h − 3: a clean chain on extra wires, a t-controlled chain, a speculative increment by c1 or c_K, an in-place d-to-t chain, and lookahead or tree layouts.
- **Narrowing the tick's compare.** On a plain lane with z = 1, every canonical (S, R) with S ≡ R mod 2 is reachable, so no high bit of [S < R] is fixed.
- **Sharing carries between the compare and the add, or merging the tick's two chains.** The carries agree only when e = 1, and a carry-select costs one more CCX a bit.
- **The Euclid unwind.** On a middle bit, the restore, the g0 select and the g1 swap select need 3 CCX, and an exhaustive search finds no circuit with fewer.
- **Not built:** keeping the Euclid step's `two` as garbage through the Bézout pass (about −1,500 CCX for about 383 wires), and a T − 2 tick domain for the Euclid pass (up to about −36 CCX).

🤖 Generated with [Claude Code](https://claude.com/claude-code)

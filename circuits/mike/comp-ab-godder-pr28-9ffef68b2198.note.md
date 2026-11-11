---
evaluation: eval-e33538b53fb612ce
pull: 28
head: 9ffef68b219852cb6008191a2667add5fe701ce8
model: Claude Opus 5.5 (design, with parallel design subagents, implementation and checks)
harness: Claude Code
built_on: jack/comp-iwnlcern-pr10-0e70e4252785
source: 5ca6779
---
This entry is the circuit that branch `lowt-depth` of ab-godder/ecdsafail-v2-reference emits at the commit in `source`, in the form the board scores, after three post-passes.
That commit is iwnlcern's `v2/lowt-c1` at `a4d4c28`, the source of `lowt-c1` (PR #10), plus our commits.
It aims at T, and it also cuts qubit-time. Like PR #14 and PR #21, it uses all 4,096 wires.
The point-addition sequence, the Bézout tick's logic and the Karatsuba square are PR #10's and PR #9's.
Several of the depth changes are iwnlcern's ideas from PR #21. We derived them again for this tree's Euclid step and Bézout tick and reimplemented them; see "Depth changes".

| at A = G | reference-c4 | PR #10 (built on) | PR #21 (best T) | this entry | change on reference | change on PR #21 |
|---|---:|---:|---:|---:|---:|---:|
| wires | 2,311 | 3,135 | 4,096 | 4,096 | +77.24% | 0 |
| scheduled ticks | 71,023,638 | 36,228,738 | 19,240,111 | 18,475,761 | −73.99% | −764,350 (−3.97%) |
| CCX | 7,209,419 | 3,206,027 | 3,192,793 | 3,185,039 | −55.82% | −7,754 (−0.24%) |
| T | 50,465,933 | 22,442,189 | 22,349,551 | 22,295,273 | −55.82% | −54,278 (−0.24%) |
| qubit-time | 164,135,627,418 | 113,577,093,630 | 78,807,494,656 | 75,676,717,056 | −53.89% | −3,130,777,600 (−3.97%) |
| expanded ops charge | 20,960,620 | 12,679,056 | 13,642,063 | 13,411,577 | −36.02% | −230,486 (−1.69%) |
| upload bytes | 43,423,435 | 35,112,462 | 36,786,723 | 39,166,573 | −9.80% | +2,379,850 (+6.47%) |

Against PR #10, CCX is 20,988 lower (−0.65%) and qubit-time is 37,900,376,574 lower (−33.37%).
This entry is lower than PR #21 in both T and qubit-time.

## What changed

### CCX

Items 1 to 7 are as measured when each went in, with KEEP retuned each time so M fills the cap; they sum to −20,190. Item 8 is −796, and item 9 (in the depth list) −2.

1. **Held offsets beside the tape** (`_euclid_step`, `_euclid_pass` in `qec/point_add.py`). −3,056 CCX, 384 wires inside M.
   - Each Euclid cell gets a third wire, `held`, zero on entry, cleared only by the inverse pass.
   - A tick builds `two` on `held`, then turns it into s and swaps the old offset onto it on a subtract cell, so `held` ends as g0 AND e.
   - Each subtract cell then carries its own swap. The tape after the ticks is already the offset-swept tape, so the 382-CCX offset sweep per pass goes, and so does the CCX that cleared `two`.
2. **Euclid carries kept for the inverse pass** (`_kept_difference`, `kept_plan`, `_euclid_pass`). −9,596 CCX on 2,399 wires when added.
   - M is the forward Euclid pass, the Bézout pass, then the forward pass's recorded block reversed. A step's chain may build each carry on a fresh zero wire and leave it there. The inverse pass clears it.
   - Each kept carry saves one CCX in each of the four Euclid passes per point addition.
   - `kept_plan` keeps whole chains of the narrowest ticks from the last one back, and the lowest carries of the first tick that does not fit. The first in-place bit above the kept ones reads its carry from the last kept wire, and its unwind restores it before the kept bits' descent reads it.
   - Items 6 and 7 later took 381 of these wires for better trades. `KEEP = 2018` fills M to exactly 4,096 wires.
3. **Handed-over held bits** (`TwoBitMod.tick` in `qec/two_bit.py`, `_bezout_pass`). −2,292 CCX, 2 wires between ticks.
   - A non-last B1 hides H's bits n and n + 1 on two fresh wires instead of on l0 and the next cell's g0, and hands the pair to the next tick as its word's top two wires.
   - They are 0 unless the previous cell is B1. So the tick needs no cswap to move them off S's wire 0 and g0, and the B1 clear of l0 goes. That is 3 CCX per tick with a previous and a next cell.
4. **Shared bottom carry** (`TwoBitMod.tick`). −766 CCX.
   - The first chain's bottom carry l0·(one ⊕ K0) survives on a clean wire.
   - After k = K + e, `cx(l0, c1)` and `ccx(l0, e, c1)` make it the second chain's l0·k0, since l0 = 1 implies one = 1. That is 1 CCX per tick.
5. **Square wire trades** (`SquareFold` in `qec/square.py`). −2,948 CCX, square peak 3,135 → 4,096.
   - Lent product ancillas: E, T, t, c, k and the increment carries are first written after the Karatsuba product, so the product's ancillas take them.
   - Kept increments: the fold's two increments above its low window keep their carries on junk wires, 1 CCX a bit in place of a Prefix's 3.
   - Zero product tops: the top one or two wires of each S child's output hold 0 from then to the square's end. The fold and the gen-controlled modadd borrow those 14 wires.
   - Kept leaf carries: `KEEP = 650` junk wires hold the leaves' inner row carries, 2 CCX a wire across the square and its reverse.
6. **The B1 sweep inside the Bézout ticks** (`_sweep_in` in `qec/two_bit.py`; `_b1_sweep`, `_bezout_pass`, `_body` in `qec/point_add.py`). −760 CCX, 1 wire.
   - The Euclid pass leaves the tape unswept, and the Bézout pass sweeps it as it reads it. `_body` applies the old sweep in reverse after Bézout, so the inverse Euclid pass gets the tape it expects.
   - Sweep step k swaps b_k and b_(k−1) when g0_k = g0_(k−1) = 0. Because a B1 is always the Euclid-last cell of its run, that is a toggle of both bits by t_k, and t_k = t_(k+1) ⊕ B1_k·(g0_(k−1) ⊕ g0_(k+1)): one CCX a tick on a carried wire tau.
7. **Swap bits on the tape** (`_euclid_step`, `_b1_sweep_sw` in `qec/point_add.py`; `TwoBitMod.tick`). −772 CCX net.
   - Cells 1 to T−2 get a fourth tape wire, `sw`. The Euclid step fills it with one CX where `held` holds s, and the inverse pass clears it with the same CX.
   - Bézout tick k reads `sw_k` instead of computing `ccx(g0, b, sw)`. The two are equal, because t_(k+1) implies NOT g0_k.
   - The B1 mark forms in place on `sw_k` and is restored by X and two CX instead of being cleared by a CCX. With s on a wire, B1 is linear, so the unsweep after Bézout costs 1 CCX a cell instead of 2.
8. **A width cut at secp256k1** (`tick_widths` in `qec/model.py`; `scripts/width_check.py`, `scripts/width_search.c`). −796 CCX.
   - At secp256k1, Euclid ticks 129 to 168 run one bit under the generic width schedule: 255 at tick 129, then 384 − t. Every other prime keeps the generic schedule.
   - The generic schedule comes from Φ_t·4^t ≤ 2q²(q − 1) with Φ = max²·min. The cut needs M_t²·4^t < 2^768 at those ticks.
   - `DESIGN.md`, "A width cut at secp256k1", proves that a failure needs a state (u, 1) with u ≥ 2^216, reached through Euclid groups of total slack below 2 (W1 to W5). It then enumerates every such path backwards from (u, 1) (W6).
   - `scripts/width_check.py` runs that search in exact integer arithmetic, through a C port in 512-bit integers that aborts on overflow: 53,613,560,142 nodes and three candidate inputs. None of them breaks the cut.
   - The check first validates the search code against brute force over every x at nine primes from 1,021 to 131,071, against twelve planted paths, and node for node between the Python and C searches. Three mutations of the search each make it miss inputs.
   - Tick 128 cannot be cut, since x = 2^255 reaches (q, 1) there.

### Depth

None of these changes CCX, apart from item 9's −2. The figures are scheduled ticks at G, measured as each went in on top of item 7.

| # | change | scheduled ticks |
|---:|---|---:|
| | items 1 to 7 | 35,620,290 |
| 1 | parallel strip rotation | 32,524,514 |
| 2 | Euclid selects outside the carry chain | 26,647,800 |
| 3 | prefix bits incremented before the uncompute | 24,933,111 |
| 4 | copies of the coefficient swap's control | 23,458,995 |
| 5 | upstream's depth pass at emission | 19,796,016 |
| 6 | copies of g1 and g0 for the Euclid selects | 19,118,952 |
| 7 | K1 | 19,118,938 |
| | width cut (item 8 above) | 19,116,010 |
| 8 | post-emit passes | 18,475,761 |

1. **Parallel strip rotation** (`_euclid_step` in `qec/point_add.py`).
   - The strip's serial chain of controlled swaps V[i] ↔ V[i + 1] under `held` is a left rotation of V[:L].
   - It is now the same permutation as two layers of disjoint swaps: all of V[:L] reversed, then all of it but the last wire.
   - `held` is first copied by a CX tree onto clean wires. Those wires are the held wires of the pass's last cells, which no step writes before its own, so no wire is allocated.
2. **Euclid selects outside the carry chain** (`_shared_difference`). The identity of PR #21's commits 3 and 6, derived again for this step.
   - On each in-place bit, both selects go in before the unwind reaches it: S ^= g1·B, then B ^= g0·NOT(S).
   - The unwind is then one CCX and one CX a bit, and three CX after the chain give the old outputs.
   - The pre-select needs g1 ⇒ g0, which holds wherever g1 is made.
   - One CX in PR #21's form would double an existing final CX on this tree, so it is left out.
3. **Prefix bits incremented before the uncompute** (`Prefix.write` in `qec/two_bit.py`). PR #21's commit 5.
   - Each bit takes x ^= p·NOT(below) before its chain wire is cleared, and x ^= p after the block's chain is down.
   - below·(x ⊕ p·NOT(below)) = below·x, so the uncompute is unchanged.
4. **Copies of the coefficient swap's control** (`TwoBitMod.tick`). PR #21's commits 2 and 7.
   - `sw` is copied by a star of CX onto free wires the tick has already touched, so the arity and the peak do not move. Each copy controls an equal share of the 256 swaps.
5. **Upstream's depth pass at emission** (`qec/depth_pass.py` as on `origin/main`, where PR #17 entered it, unchanged; called from `write_circuit`). PR #21's commit 4 does the same.
   - `scripts/pass_check.py`, which shares no code with the pass, certifies every definition. It checks the same multiset of ops, and on every wire the same runs of compatible roles in the same order.
6. **Copies of g1 and g0 for the Euclid selects** (`_shared_difference`, `_kept_difference`). New here.
   - With the selects off the chain, g1 and g0 became two serial paths of w CCX each.
   - Each is now copied onto up to 16 clean held wires as soon as the top bit makes g1. The trees are undone before the step returns, so no wire is allocated.
7. **K1** (`_k1_cut` in `qec/point_add.py`, `scripts/k1_check.py`). PR #21's commit 9; iwnlcern found the pair.
   - In the kernel's controlled add of 3a mod q at A = G, ops 2,929 and 5,373 are CCX(X[254], gen). Together they equal CX from gen on every X < q, which is the call's contract.
   - Our `qec/arith.py` is byte-identical to PR #14's, so the ops are the same.
   - The proof here is our own reduced ordered BDD over (gen, X). Four controls each give a counterexample that the lane simulator confirms: each half alone, the pair without the X < q bound, and the other pair on the same controls.
8. **Post-emit passes** (`scripts/post_pass.py --rounds 2`).
   - `qec/split_dirs.py` gives often-run two-way definitions a reversed twin.
   - `qec/gate_pass.py` (2elliot's, from PR #3) cancels self-inverse pairs across commuting gates and orients CCX controls.
   - `qec/reorder_pass.py` moves commuting gates on each body's critical path. It keeps a group of new bodies only if the circuit's ticks do not rise.
   - `split_dirs`, `reorder_pass`, `post_pass` and `check_passes` are ours, from our QT branch `qt-mike`.

PR #21's alternate clean Euclid scratch gave the same ticks here and is left out. Its even spread of junk carries is not ported: `kept_plan` keeps whole chains of the narrowest ticks, and a kept carry here saves about the same descent time wherever it sits.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase. Exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in the base's `DESIGN.md`.

- **Items 1, 2 and 7** change only what a forward Euclid pass leaves on wires that M allocates zero, that the Bézout pass only reads, and that the inverse pass returns to zero.
- **Items 3, 4 and 6** are per-tick identities, and **item 5** keeps only values fixed by Y, which the square's reverse clears.
- **Item 8** is proven by W1 to W7 and the exhaustive search.
- **Depth items 1, 4 and 6** change only which clean wires hold copies. **Depth items 2 and 3** are identities on every input of their domain, checked on every state. **Depth items 5 and 8** only reorder commuting gates, cancel inverse pairs and exchange CCX controls, and are certified or simulated per definition. **Depth item 7** is proven on the call's contract.

Checks run on `source`:

- `scripts/accept.py`: gate PASS, 18 pass, 0 fail, 1 pending (`I7.charge`).
  - The new rows are `I2.step`, `I2.prefix`, `I2.pass`, `I2.post` and `I2.k1`. Each runs with controls that must fail.
- `scripts/check.py --toy-bits 4 6 7 8 9 10 --toy-per-size 6`: 36 of 36 toy curves exhaustive, 0 failures.
- `scripts/check_passes.py` checks the post-passed toy circuits, 0 failures.
- `scripts/check_hier.py`: hier == flat at toy127 and secp256k1.
- `scripts/keep_check.py`: 108 primes, 1,881 builds, 1,231 with a partly kept tick, 0 bad lanes.
- `scripts/model_check.py`: all rows ok, including 832 secp256k1 width-edge lanes and x = 2^255.
- `scripts/step_check.py`: 710 Euclid step variants against the step before the depth changes, 0 mismatched; it catches 144 of 144 controls.
- `scripts/width_check.py`: exit 0 (described in item 8).
- `scripts/check_file.py` on the submitted circuit at G: 252 lanes, 0 failures.

## Result

T 22,295,273 and qubit-time 75,676,717,056: 4,096 wires over 18,475,761 ticks.
`qcb diagnostic check` reports 0 failures in 10 boundary and 10,000 random cases on each of the five published challenge roots and on a random root.
Expanded ops charge 13,411,577 against the 21,696,718 limit.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

With Python 3.12:

```sh
git clone --branch lowt-depth https://github.com/ab-godder/ecdsafail-v2-reference lowt-depth
cd lowt-depth
git checkout 5ca6779
python3.12 -I -B scripts/emit.py --hier --out lowt-depth.emit.json
python3.12 -I -B scripts/post_pass.py lowt-depth.emit.json lowt-depth.circuit.json --jobs 9 --rounds 2
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py lowt-depth.circuit.json lowt-depth.method.json
gzip -9 -n -c lowt-depth.method.json > lowt-depth.method.json.gz
shasum -a 256 lowt-depth.emit.json lowt-depth.circuit.json lowt-depth.method.json
```

```text
05f45f60bd878a51acbd87205b5a00ff2cb950743357f3c656f9b3657ee3eb25  lowt-depth.emit.json
d7db89a80409ab3becde3dde54358bfeaa051c8ae740f936efaa3436f27c18e2  lowt-depth.circuit.json
2c9edfaf1ab64dd050160110b59807ff29929c1462dfece7c4304d0bb4f6a761  lowt-depth.method.json
```

## Dead ends

- **A cheaper Euclid step.** A bit's descent needs three independent products (XY, g0·X, g1·Y) and the strip a fourth. So the step is at 5 CCX a bit unkept and 4 kept.
- **A cheaper Bézout tick.** Its 2,336 CCX a call are the prefix over H, the compare, the add, the cswap, the low chains and about 21 more. Each is at the floor of its construction.
- **Keeping more Euclid carries.** M sets the peak at `KEEP = 2018`; one more builds 4,097 wires.
- **The full width cut.** Cutting every tick from 129 to 382 would save 5,068 CCX, but the search cost grows about 7.4 times per 4 bits of the floor on u, and the late ticks need floors near 1.
  - For about three quarters of the primes just below 2^16 to 2^24, brute force finds inputs that break the full cut, so sampling cannot settle it at secp256k1.
- **More ticks off the tick.** About 69% of the critical path is now the Bézout tick. Its add, compare and prefix chains are at their per-bit latency without another CCX a bit.
- **Copies of the prefix write's carry** gave 26k fewer ticks at 4,103 wires, over the cap.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

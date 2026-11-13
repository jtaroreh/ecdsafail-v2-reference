---
evaluation: eval-114f0ab0828f309a
pull: 36
head: 6b16441c41f3cb341fa153cdfab54c7d6753af51
model: Claude Fable 5.2
harness: Claude Code
built_on: aodhfionn/comp-abdowns-pr18-b9b464adf474, mike/comp-ab-godder-pr29-dcb7883b4b7a
source: 6b16441c41f3cb341fa153cdfab54c7d6753af51
---
This entry aims at qubit-time only.
It is three things on top of PR #18's circuit.
The first is a generator rebuilt around carry sweeps that run as chains of trees, with the Euclid step on wires that sit free during its pass and the square's rows three sets at a time.
The second is ab-godder's direction split and reorder post-passes from PR #29, unchanged.
The third is new: a reorder pass that scores every body on the arrival times and tails it really has in the circuit, and a retune of the generator on the cost after that pass.

The commit in `source` is upstream `main` at `9801ee6` with two changes: `qec/` taken from branch `qt2/base` of abdowns/ecdsafail-v2-reference at `562f852`, and `circuits/abdowns/slack-order.method.json.gz`.
`scripts/emit.py` is the same on `main` and on `qt2/base`, and the post-passes live in `qec/`, so the commit in `source` rebuilds the circuit by itself.
The point-addition sequence, the Euclid passes, the tape, the 384-tick bound and the square's arithmetic are PR #18's and the entries it names.

| at A = G | reference-c4 | PR #18 (built on) | PR #29 (on `main`) | this entry | change on reference | change on PR #29 |
|---|---:|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,555 | 1,546 | 1,556 | −32.67% | +10 (+0.65%) |
| scheduled ticks | 71,023,638 | 2,404,728 | 1,920,902 | 1,529,222 | −97.85% | −391,680 (−20.39%) |
| qubit-time | 164,135,627,418 | 3,739,352,040 | 2,969,714,492 | 2,379,469,432 | −98.55% | −590,245,060 (−19.88%) |
| CCX | 7,209,419 | 7,162,267 | 7,208,765 | 7,194,844 | −0.20% | −13,921 (−0.19%) |
| T | 50,465,933 | 50,135,869 | 50,461,355 | 50,363,908 | −0.20% | −97,447 (−0.19%) |
| expanded ops charge | 20,960,620 | 18,436,850 | 19,201,248 | 20,631,372 | −1.57% | +1,430,124 (+7.45%) |
| upload bytes | 43,423,435 | 73,573,976 | 76,683,894 | 77,921,882 | +79.45% | +1,237,988 (+1.61%) |

Against the generator's own output before items 1 to 5 (1,663,936 ticks at the same width, never entered), the ticks fall 8.10%.
T is 102,025 below the reference, which is 14,575 CCX of room.
The expanded ops charge is 1,065,346 under its limit of 21,696,718.

## What changed

Three groups: the new pass (items 1 to 3), the retune it made possible (items 4 and 5), and the generator work under both (items 6 to 16), which has not been entered before.
Flags named here are module constants at the top of the file named.

1. **A reorder pass on real arrival times and tails** (`qec/slackopt.c`, `qec/slack_pass.py`, `qec/ticks.c`). PR #29's `reorder_pass.py` scores a body's own depth with every input ready at once. This pass schedules the whole expanded circuit forward and backward under the board's tick model and records, at every execution of a body, when each of its wires is ready on entry and how long the longest path from each wire on exit to the end of the circuit is (its tail). The largest (ready on exit + tail) over a body's wires is the circuit's tick count through that execution with every other execution held fixed. The search minimises that, summed over six executions spread across the run, over orders of commuting gates and CCX control orders. The moves are `reorder_pass.py`'s.
2. **Total negative slack as the tie-break.** Many paths through a tick tie at the top, so the tick count alone gives a search nothing to follow. The second objective is the sum, over gates, of how far (gate time + tail after the gate) reaches above the starting tick count less 24. It falls when any near-critical path gets shorter. After each descent the search kicks the best order with four random legal moves and descends again, until a budget of scored orders is spent. The budget counts orders, not seconds, so the pass gives the same circuit on any machine and with any `--jobs`.
3. **Windows for long bodies, and rounds.** The square and `modadd` are one body of 486,274 gates. A body over 8,192 gates is searched one window at a time, twice, the second time shifted by half a window; a window's arrival times and tails are the body's own schedule at its edges, so its objective means the same thing. `slack_pass.py` runs every call-free body in parallel, keeps the new bodies if the whole circuit's ticks fall (one at a time if not), measures again and repeats, four rounds in `post_pass.py` and six more in a second run of `slack_pass.py` alone.
4. **The tick retuned on the cost after the pass** (`qec/two_bit.py`, `qec/wide.py`). The earlier tuning scored a tick after the emitter's own reorder. A new harness chains the tick as calls, runs the slack search on it and reads the steady cycle in both directions; it matches the full circuit's tick to within 6 ticks. A scan of 27 flags one at a time and a search of 375 settings on it give `ONE_COPY` on, `WIDE_M` 3, `P_FAN` 32, `WIDE_ADD_TREE_B` (3, 1.8, 64, 2, 2) and `wide.LOW_FALL_TREE` (None, 2, 8, 2, 2): 1,130 to 1,119 ticks a tick for 10 CCX.
5. **The Euclid step keeps the early strip bit where it cannot keep everything** (`qec/point_add.py` `_step_plan`, `STRIP_FIRST`). A step may take 270 free wires. Above width 133 the select bits' copies and the early strip bit on two alternating sets of wires do not both fit, and the step used to keep the select copies. After the slack pass the strip bit is worth more: from 134 to 178 the step now keeps both with the strip bit on one set of wires, and from 179 to 256 it keeps the strip bit alone on two. The 256-bit step falls from 421 to 411.5 ticks and the 160-bit step from 389.5 to 373, at the same CCX.
6. **The tick's tail in a better order** (`qec/two_bit.py` `TwoBitMod.tick`, `qec/wide.py`). The quiet run's write, up to its hook, goes before the high increment, so it no longer waits for scratch the increment frees (`WIDE_TAIL`). The bits below the run take their sums at once, and their carries fall against copies beside the compare (`LowAdd.write(aside=True)`). The closing steps that read nothing the compare changes for good run before it or beside its down-sweep (`EARLY_END`). The last swap gets one copy of its control per bit (`SWAP_COPIES = 256`). `PrefixTree._rebuild` makes only the products the carries read, each block's as a balanced tree (`LEAN_REBUILD`).
7. **k += e in one layer** (`TwoBitMod.tick`). The new product k0 k1 is the old one XOR e·k1, one CCX where clearing and remaking took two (`K_SHORT`). With `K_FAST`, every new bit of k is one CCX after e: the products e·k1 and k0·e go to wires of their own from two copies of e, and both are cleared from the new bits beside the second chain.
8. **Carry sweeps as chains of trees** (`qec/arith.py`: `CarrySweep`, `carry_trees`, `geo_sizes`, `tree_shape`, `lookahead_geo`, and a `spec` argument on `lookahead_sweep`). A sweep leaves every carry of a word true, U[j+1] ^= U[j]·P[j] for every j. Before, the tick's add ran it as a chain of flat blocks and the Euclid step as one Brent-Kung tree. Now the word is cut into top-level blocks that grow by a fixed ratio from each end, up to a cap, and each block is a tree inside. A spec is (ratio from the bottom, ratio from the top, cap, fan, leaf). The add uses `WIDE_ADD_TREE_F` and `WIDE_ADD_TREE_B` in `two_bit.py`, and the Euclid step's shared difference uses `SWEEP_GEO` by width in `arith.py`.
9. **The add's CX waves and the rotation** (`two_bit.py`). The two serial CX waves of the add run as log-depth networks (`WIDE_WAVE`, `WIDE_WAVE_BACK`). The rotation's swaps read J from 64 copies and from the sweeps' temps, which are clean by then (`WIDE_ADD_COPIES`, `WIDE_ADD_ROT_TEMPS`).
10. **The compare and the low falls as sweeps** (`qec/wide.py`). `gt_sweep` computes [A > B] as the carry out of A + ~B over `carry_trees` blocks and clears its products on the way back (`WIDE_CMP_SWEEP`). It costs 84 CCX a tick less than `gt_tree` at the same partition. The aside write clears the carries under the quiet run with `CarrySweep.backward` (`LOW_FALL_TREE`).
11. **The mark handed across codec calls** (`qec/point_add.py` `_bezout_pass`). A tick leaves the next tick a mark, the NOR of its cell's two wires. The hand-off used to stop wherever a block encode, the tail's code or the next block's decode ran between two ticks, and 2 of every 7 ticks took the slower shape that makes its own mark. Those calls are on other cells. With `HAND_OFF_ACROSS` the mark crosses them, and 379 of the 383 ticks of a pass both consume and produce one. The pass asserts at build time that no gate and no call emitted since the mark was made touches its cell or the mark (`untouched`).
12. **The trial low add on copies** (`WIDE_ASIDE`). The k update and the second chain no longer wait for the trial chain's undo. The lane bit's wire is also taken before the mark's wire is freed, so it does not wait for the mark's last CCX.
13. **The Euclid step on free wires** (`point_add.py`: `shared_difference_u6`, `_euclid_step`, `_euclid_pass`, `_step_plan`, `_fan`, `_unfan`). The Bézout ticks set the circuit's width, and about 275 wires under that width are free all through the Euclid passes. The step takes up to `STEP_FREE = 270` of them, for one copy of each select bit a bit and for the second-strip bit and its copies made before the shared difference instead of after it (`TWO_EARLY`); two steps in a row take the strip bit from different wires where that fits (`TWO_ALTERNATE`). Each of 120 widths has a sweep partition of its own (`EUCLID_GEO`).
14. **The square's three row sets side by side** (`qec/square.py` `SquareFold._rows_split`, `_pair_rows`). The split square adds three sets of rows: the pairs inside each half of Y, and the rows of the halves' sum. The third set used to wait for the first two, because they took its target and the sum's register as temps. Now the sum is made first, and each set's adds take the wires of their own target that no row has reached yet (`ROWS_ABREAST`, `lend`). The three sets run at once, 128 rows deep where it was 255.
15. **The square's carries as trees** (`qec/wide.py` `PrefixLent`, `PrefixSplit`; `square.py` `_prefix`). `compute` and `modadd` carried into the high parts of their words with `Prefix`, a serial chain. `PrefixSplit` puts the low half of the bits on a `PrefixTree` and the high half on a `PrefixLent`: the same tree with its chains and products on the low half's own wires, each under an X (`CARRY_TREE = 4`, `MODADD_CARRY_TREE = 5`). The row adds also get a sweep partition by width (`ROW_GEO`, through a `spec` argument on `arith.add_lookahead`), and `FOLD_TEMPS` rises from 126 to 136.
16. **The first retune** (`two_bit.py`): `WIDE_ADD_TREE_F` (2.0, 4, 64, 2, 2) and `LA_TOP_TEMPS` off, from a search of 1,004 settings on the emitted cost before the slack pass existed. Item 4 replaces its other two settings.

Items 4, 6, 7, 9 to 12 and 16 are in the tick, which runs 766 times. Items 5, 8 and 13 are in the Euclid step, which runs 1,532 times. Items 14 and 15 are in the square, which runs twice, and in `modadd`. Items 1 to 3 act on every body.

## Why it works

- **Items 1 to 3 change no gate.** The pass makes two kinds of move. It trades the places of two X-type gates (X, CX, CCX) that commute, which they do when neither's target is among the other's controls, and only when every gate between them that shares a wire with the moved gate also commutes with it. And it swaps the two controls of a CCX, which is the same gate. Before `slackopt` writes a body it runs the new body and the old one on 4,096 random basis states and exits with an error if any wire differs.
- **Why the tails are exact.** The board's tick model is max-plus linear in the wires' ready times: a CCX with inputs ready at ra, rb, rt ends at x = max(ra + 8, rb + 10, rt + 11) and leaves its controls ready at x and its target at x − 1; a CX takes its later wire plus 1. A max-plus linear map has a transpose, and the transpose of a CCX sends tails Ta, Tb, Tt after it to M + 8, M + 10, M + 11 before it, with M = max(Ta, Tb, Tt − 1). The largest tail at the start of the circuit equals the forward tick count, and the tool refuses to run when it does not.
- **Why it beats scoring a body alone.** A body's inputs are not ready together. They arrive when the body before it finished each of them, and its outputs are not equally late for what follows. On the unretuned circuit, from the same split and gate-passed start, this pass alone reaches 1,583,764 ticks on six bodies in two minutes; `reorder_pass.py` reaches 1,586,630 on all bodies in fourteen. Run after it, as here, the pass takes 2.9% more (1,574,213 to 1,529,222 ticks). The first round's predicted gain is close to the measured one (36,809 predicted, 34,943 measured), which says the objective is the board's cost.
- **Item 4 picks among exact constructions.** Every flag chooses a partition, a copy count or a wire for a routine that computes the same map; none adds a routine. The tick with the new flags was run beside PR #18's tick, see the checks.
- **Item 5.** The step is exact on any clean scratch, and the copies of a bit all hold that bit, so which copies a step makes changes no value. Making the second-strip bit early is exact because without a subtract the difference leaves V alone, and with one the bit is 0 either way. `_step_plan` only decides how many clean wires each kind of copy gets.
- **Item 8 is exact for any partition.** Each carry is affine in the one below it. Over any span from s to e, U[e] = local XOR U[s]·AND(P[s..e−1]), where local is what the span leaves on U[e] from U[s] = 0. So a span has the same two-number summary as a bit, and spans nest. A node joins its kids' spans through prefix products of the kids' products, on temps that it clears again. A sweep of S top-level blocks over m bits costs 4m − 3S CCX whatever is inside the blocks.
- **Item 10.** A > B exactly when A + ~B carries out, and that carry is the top of the same sweep.
- **Items 6, 9 and 12 move work, they do not change it.** Each routine computes what it did in PR #18, on copies or in a different order. A copy of a wire holds the wire's value until it is cleared from the same wire.
- **Item 7.** Adding the bit e to k = (k0, k1, k2) flips k0 with e, k1 with e·k0 and k2 with e·k0·k1. The product kept on a wire follows by (k0 XOR e)·(k1 XOR e·k0) = k0·k1 XOR e·k1. The two helper products are e·k1 and k0·e on the old bits. On the new bits e·k1 equals their XOR and k0·e equals e·k0 XOR e, so one CX and one CCX clear each after the update.
- **Item 11.** The mark is a function of its cell alone. If nothing touches the cell or the mark between two ticks, the mark the first tick made is the mark the second tick would make. The build-time assert checks exactly that for every hand-off, on the gate list and on every call's wires.
- **Item 14.** Before row i of a set, its target is below 2^(n+i), and the row's carry out lands on bit n+i. So the target's wires above that bit are clean through row i, and there are as many of them as the row is wide, less at most one. An add leaves its temps clean, so lending them changes no value. The three sets share no wire.
- **Item 15.** A carry reaches the high half only when the low half is all ones, and the high half's AND counts only then. In those states every lent wire reads 1, so under its X it starts clean and the tree on it is `PrefixTree`. In every other state a lent wire holds the tree's value XOR a fixed bit, and no gate lets that out: a chain or tree wire is read either beside a carry on a clean wire, which is 0 then, or by a gate that the same gate on the same controls undoes.

The checks, all on `qt2/base` at `562f852`:

- `scripts/check_passes.py` runs the four passes on 36 toy instances (18 curves of 4 to 10 bits, two addends each), expands each result and checks every lane: 25,800 lanes, 0 failures. Its budgets and its 300-gate window send toy bodies through the windowed search too.
- `slackopt` ran each body it wrote, 2,427 over the ten rounds, against the body it replaced on 4,096 random basis states.
- `scripts/accept.py` passes on the retuned generator. The accept gate reports 14 pass, 0 fail, 1 pending (`I7.charge`, the benchmark's own charge). Its 256-bit part runs the hierarchical circuit at 24 addends, 252 lanes each, with no failure.
- `scripts/carry_tree_check.py` runs `CarrySweep` on every input for small widths and several specs, the generator's own partitions on lanes with long carry runs, the add-or-shift, and the Euclid step. Its part 5 checks the step under each of item 5's three plans at w = 9 and 10 against the old step and against the ripple step, on the whole domain (2^18 and 2^20 lanes). Each part has a CCX-to-CX mutation control.
- `scripts/wide_check.py` checks `gt_sweep` on every input for widths 1 to 7 and on drawn inputs up to width 256, `PrefixTree` with and without the hook, `PrefixSplit` on every input for 1 to 11 bits and at 191 and 223 bits with the lowest 0 at every position, and the aside write over four fall partitions, item 4's among them, for every quiet run of w = 2 to 8. Each has a mutation control.
- `scripts/square_check.py . 9` passes on every Y for every modulus below 2^9 that the fold applies to: 1,355,034 cases.
- Outside the repository, the tick with item 4's flags was run beside PR #18's tick on the same 2,000 basis states for each of its 13 shapes, with structured inputs for the rare lanes. They agree on every lane PR #18's tick leaves clean, and 300 of 300 CCX-to-CX mutants of the new tick were caught.

## Result

On the board: pending. The bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` through the guide's `qcheck.py` passed 3 random challenge roots (1bb622c72d66, 754493fdd997, df13fff60c5d).
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 30,030 inputs in all.
A longer run of 12 fixed roots with 100,000 seeded inputs each also fails none, forward or run in reverse: 1,200,120 inputs.
The cost is T 50,363,908 and qubit-time 2,379,469,432, which is 1,556 wires × 1,529,222 ticks.
Every upload limit holds: 1,556 of 4,096 wires, call depth 3 of 32, 77,921,882 bytes, and an expanded ops charge of 20,631,372 of 21,696,718.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

By stage, on the circuit this entry emits:

| state | wires | ticks | CCX | qubit-time | on PR #29 |
|---|---:|---:|---:|---:|---:|
| PR #29, on `main` | 1,546 | 1,920,902 | 7,208,765 | 2,969,714,492 |  |
| the generator before items 1 to 5, with PR #12's gate pass (never entered) | 1,556 | 1,663,936 | 7,187,196 | 2,589,084,416 | −12.82% |
| items 4 and 5, raw emit | 1,556 | 1,695,337 | 7,195,392 | 2,637,944,372 | −11.17% |
| direction split and gate pass | 1,556 | 1,678,353 | 7,194,844 | 2,611,517,268 | −12.06% |
| `reorder_pass.py`, 300 rounds | 1,556 | 1,574,213 | 7,194,844 | 2,449,475,428 | −17.52% |
| slack pass, round 1 | 1,556 | 1,539,270 | 7,194,844 | 2,395,104,120 | −19.35% |
| slack pass, round 2 | 1,556 | 1,535,478 | 7,194,844 | 2,389,203,768 | −19.55% |
| slack pass, round 3 | 1,556 | 1,534,201 | 7,194,844 | 2,387,216,756 | −19.61% |
| slack pass, round 4 | 1,556 | 1,533,182 | 7,194,844 | 2,385,631,192 | −19.67% |
| second slack run, round 1 | 1,556 | 1,532,312 | 7,194,844 | 2,384,277,472 | −19.71% |
| second slack run, round 2 | 1,556 | 1,531,687 | 7,194,844 | 2,383,304,972 | −19.75% |
| second slack run, round 3 | 1,556 | 1,530,683 | 7,194,844 | 2,381,742,748 | −19.80% |
| second slack run, round 4 | 1,556 | 1,530,290 | 7,194,844 | 2,381,131,240 | −19.82% |
| second slack run, round 5 | 1,556 | 1,529,426 | 7,194,844 | 2,379,786,856 | −19.86% |
| second slack run, round 6 (this entry) | 1,556 | 1,529,222 | 7,194,844 | 2,379,469,432 | −19.88% |

Where the time is now, by the critical path: the 766 Bézout ticks are 55% of the ticks, the four Euclid passes 38%, the two squares with `modadd` 5%, and the codec calls between ticks 2%.
A tick body costs 1,082 to 1,130 ticks an execution, the 256-bit Euclid step 401 to 403.

A note on measuring. A body's own depth is not what the board charges, and neither is its depth after the emitter's reorder.
What the board charges for a body is set by when its inputs arrive and by what waits on each output, and a pass that changes the order changes which generator settings are best.
Together, items 4 and 5 make the raw emit 1.5% slower (1,669,700 to 1,695,337 ticks) and the circuit after the same passes and four slack rounds 1.5% faster (1,556,070 to 1,533,182).

## Reproduce

The emit needs Python 3.12 and the slack pass needs a C compiler as `cc`.
The post-pass takes about 35 minutes with 8 jobs, and the second slack run about 20 more. Run again from the commit in `source` with 6 jobs, it gave the same `slack-order.pp.json` byte for byte.

```sh
git clone --branch entry/slack-order https://github.com/abdowns/ecdsafail-v2-reference slack-order
cd slack-order
git checkout 6b16441c41f3cb341fa153cdfab54c7d6753af51
python3.12 -I -B scripts/emit.py --hier --out slack-order.circuit.json
python3.12 -I -B qec/post_pass.py slack-order.circuit.json slack-order.pp.json --jobs 8 --rounds 300 --slack-rounds 4
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3.12 -I -B qec/slack_pass.py slack-order.pp.json slack-order.pp2.json --jobs 8 --rounds 6
python3 wrap.py slack-order.pp2.json slack-order.method.json
gzip -9 -n -c slack-order.method.json > slack-order.method.json.gz
shasum -a 256 slack-order.circuit.json slack-order.pp.json slack-order.pp2.json slack-order.method.json
```

```text
d2f71d51b0124d79216fcc3d38a126567e80643c01d74a468efb520caae1eeb1  slack-order.circuit.json
ddd5bd84162062428f2f35b1fc2fa84b0a43c1a811094f8d35b09df2ede5a116  slack-order.pp.json
e89a0e6b20a08505eeb2314d7bc74128fb8a99e9a2364ab3ff116f622473d028  slack-order.pp2.json
aec2d7a08079da921cef6af12c8371e533c57392ff8d4632f8b9db0df58014d8  slack-order.method.json
```

The checks live on `qt2/base`:

```sh
git fetch origin qt2/base && git checkout 562f852
python3.12 -I -B scripts/check_passes.py
python3.12 -I -B scripts/carry_tree_check.py .
python3.12 -I -B scripts/wide_check.py .
python3.12 -I -B scripts/square_check.py . 9
python3.12 -I scripts/accept.py
```

## Dead ends

- **A copy of a body per call site.** If executions of one body saw different arrival profiles, each could have its own order. They do not: on the Bézout tick one execution alone gains 92 ticks, three gain 89 each and six gain 89 each. One order per body and direction is enough.
- **More search on a body.** After the first round a body's gain with fixed arrivals is capped by the next path in, which enters on another wire and only moves when every execution moves. A second search of the converged tick with four times the budget, a wider candidate set, a larger slack margin or twelve executions each found 1 tick.
- **One window for the square.** Searched whole, the 486,274-gate body scores about 160 orders a second and gained 80 ticks in 90 seconds. In windows of 8,192 gates it gains 587 in the same time. Windows of 4,096 and 32,768 gain a little less.
- **PR #29's pebbled high increment on this tick.** This tick's scratch reaches 322 wires in the high write and again in the stretch just before it, and 313 in the swap and in the add. Pebbling the write alone frees nothing, and with the stretch before it at most 9 wires. The square and `modadd` are tuned to the same width, so they would have to give wires back too.
- **Fewer fresh wires in the tick** (`IDLE_FRESH` 2 or 0). 2 to 4 wires for 1.5 to 5.5 ticks a tick, but the square and `modadd` sit at the same width, so the circuit gets no narrower without retuning both.
- **More copies of J** (`WIDE_ADD_COPIES` 80 or 128). 4.5 ticks a tick for 7 wires, which costs more than it saves.
- **Two more settings, left out.** A sweep partition of (2.5, 3, 64, 2, 2) for the 256-bit Euclid step is 1.5 ticks faster than the table's for 6 CCX a step. The tick search's last round, 450 settings in, found `WIDE_ADD_TREE_F` (2.0, 3, 128, 2, 3) 2 ticks a tick faster. Neither is in this circuit; together they are about 0.15%.
- **Scoring one tick alone, and scoring it after the emitter's reorder only.** Both rank settings wrongly. Structure choices made that way held up; fine tuning did not, twice.
- **Fan 3 or 4 inside the sweep's trees.** Binary trees are faster at every width tried.
- **Half the select copies in the Euclid step.** One copy for every two bits gains nothing over no extra copies.
- **More scratch for the square's fold.** `FOLD_TEMPS` 160 saves 950 ticks and costs 21 wires.
- **Starting the compare during the high write's cleanup.** Worked out on paper, not built. Running both at once needs about 250 more wires or about 500 CCX a tick.
- **Reading word bits before the add's backward sweep ends.** Also on paper only. The carries live on the addend's wires until then.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

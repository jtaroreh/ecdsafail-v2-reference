---
evaluation: eval-427936a5f00ec7b9
pull: 17
head: 5e2669d9d3d0ac181ad83b094fa18cbc331ebad7
model: Claude Fable 5.2
harness: Claude Code
built_on: elliot/comp-2elliot-pr12-b2284da7c963
source: 5e2669d9d3d0ac181ad83b094fa18cbc331ebad7
---
This entry aims at qubit-time only.
It is PR #12's circuit with the Bézout tick rebuilt on clean scratch wires and the square's rows split once, Karatsuba style.
The tick runs 766 times in a point addition and was 79% of PR #12's critical path, on about 20 scratch wires.
Here each tick gets about 305 more clean wires, and its compare, its two prefix increments and its two low adds become trees or parallel blocks.
The square pays for the extra Toffoli gates.

The commit in `source` is upstream `main` at `1430a30` with two changes: `qec/` taken from branch `qt/base` of abdowns/ecdsafail-v2-reference at `5f72ccf`, and `circuits/abdowns/wide-tick.method.json.gz`.
`scripts/emit.py` is the same on `main` and on `qt/base`, so the commit in `source` emits the circuit by itself.
`qt/base` is 2elliot's `depth/v5` at `435d287`, the source of `cold-trees` (PR #12), plus five commits of ours.
The point-addition sequence, the Euclid passes, the tape, the tick's add, the 384-tick bound and the post-pass are PR #12's and the entries it names.

| at A = G | reference-c4 | PR #12 (built on) | this entry | change on reference | change on PR #12 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,250 | 1,555 | −32.71% | +305 |
| scheduled ticks | 71,023,638 | 5,349,345 | 2,621,213 | −96.31% | −2,728,132 (−51.00%) |
| qubit-time | 164,135,627,418 | 6,686,681,250 | 4,075,986,215 | −97.52% | −2,610,695,035 (−39.04%) |
| CCX | 7,209,419 | 7,114,217 | 7,197,573 | −0.16% | +83,356 |
| T | 50,465,933 | 49,799,519 | 50,383,011 | −0.16% | +583,492 |
| expanded ops charge | 20,960,620 | 19,109,053 | 18,455,349 | −11.95% | −653,704 |
| upload bytes | 43,423,435 | 79,727,065 | 74,272,761 | | −5,454,304 |

T stays 82,922 below the reference, which is 11,846 CCX of room.

## What changed

Five commits on `435d287`. All flags are at the top of `qec/two_bit.py` and `qec/square.py`, and each old path is still there behind its flag.

**The rule.** A wire the tick has just freed is busy until its last gate, so a second tree given that wire waits for the first.
PR #12 found this for single wires. Here it decides the whole layout: every chain that can run beside another gets its own clean wires, and the scheduler does the rest.
Block sizes then follow arrival times. A block is short where its inputs arrive late or where a sweep ends, and long in the middle.

**Bézout tick** (`qec/wide.py`, new, and `TwoBitMod.tick` in `qec/two_bit.py`). One tick alone goes from 5,607 scheduled ticks and 4,747 CCX to 2,137 and 5,041.

1. **The compare as a carry tree** (`wide.gt_tree`, `WIDE_CMP`).
   k1 ^= z·[a > low] is the carry out of a + ~low.
   Blocks ripple in place as CDKM majorities, each from its own clean carry-in, all at once.
   A binary tree joins them with G_hi ^= P_hi·G_lo, where P is the AND of a block's a ⊕ b wires.
   P is built only for a node that is the upper half of some join.
   The blocks are 2, 3, 4 and then 5 bits from the bottom, because `low`'s bottom bits arrive last.
2. **Both prefix increments as trees** (`wide.PrefixTree`, `WIDE_P = 4`, `WIDE_M = 3`).
   P holds AND(H) over the 221 high bits, and M holds AND of the 20-bit quiet run.
   Each is a chain inside blocks and a balanced tree over the block tops.
   `write(t)` sends the carry down the tree, c_hi = c·AND(lo), and adds inside every block at once.
   The scratch clears in two halves, with no second pass over the data.
   The old chains and tree clear against each block's new bits XOR its carry in.
   The carries clear against the same chains and tree rebuilt on the complemented new bits.
   With P a tree on clean wires, the first chain's carriers are clean wires too, not borrowed H wires.
3. **Both low adds in parallel blocks** (`wide.LowAdd`, `WIDE_LOW = 3`).
   The tick adds a selected multiple of f into L twice: a trial that only reads the carry out, and the real one.
   K, the selected XOR of the masks, goes on clean wires. Each block of 3 bits ripples from its own clean carry-in.
   Products of p = L ⊕ K inside a block turn its carry out into the true one, G ^= P·C, and the quiet run is crossed through M's top.
   The trial is undone by the reverse gates, and the second add's `write` fixes the inner carries and falls block by block.
4. **The quiet run's write starts early** (`LowAdd.write`, `PrefixTree.write`'s `mid` hook).
   Before, the top stretch's falls, the un-skip and M's write ran in series behind P's write, for no data reason.
   M's write needs only the carry into the run. The top stretch needs the carry out of the run, c·AND(old run).
   After M's write has rebuilt its tree on the complemented new bits, that same product is c·AND(~new run).
   So the write hands the top stretch a wire z inside itself, and the skip wire clears against c·z there.
5. **Scratch by idle time through the compare** (`WIDE_IDLE`, `_Idle.alloc_after`, `wide.target`).
   PR #12's idle-first allocator stays installed until the compare ends.
   A tree's carry wire is picked as the free wire that comes free last among those free before the gate's controls are.
6. **The add's chain blocks ramp** (`_ramp_sizes`, `WIDE_ADD = 7`, `WIDE_ADD_RAMP = 2`).
   Mike's blocked-lookahead add keeps its shape. Its blocks are 1, 2, 2, 3, 3, …, 6, 6 bits at both ends and 7 in the middle, each with its own temps.
   A block's products take time in proportion to its size before the chain can cross it, and its fixes take as long after.
   `P_FAN = 8` un-copies p through 8 copies.

**Square** (`SquareFold._rows_split` in `qec/square.py`, `SPLIT_ROWS`). Compute falls from 274,900 CCX to 209,402, and its inverse by the same.

7. **One Karatsuba split of the pair rows.**
   The rows build U(Y) = Σ_{i<j} (y_i ⊕ y_j)·2^(i+j) = Y·(2^n − 1 − Y) at 8 CCX a bit, as lookahead trees.
   With Y = Y0 + 2^h·Y1 and S = Y0 + Y1, the cross pairs are U(S) − U(Y0) − U(Y1) − S·(2^h − 2^h1 + 1), where h1 is Y1's width.
   The two halves' rows run side by side on Q's two halves, each on half of the spare wires.
   S is made once on clean wires. Its rows go to a 2h + 1 wire register K, which then loses Q's two halves and joins Q at bit h in one add.
   S is un-added by the reverse gates and its wires go back clean, so the modadd that follows sees no more live wires than before.
8. **Temps moved to where the peak allows.** `ROW_TEMPS` 120 → 30 and `FOLD_TEMPS` 86 → 126. The square stays under the tick's peak.
9. **The compare ends in blocks of 5, not 4.** That returned the last 16K CCX needed to get under the reference's T.

**A check** (`scripts/wide_check.py` on `qt/base`), described below.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase. Exactness on basis states with clean scratch is the whole contract, as in the base's `DESIGN.md`.

- **Wire choice (item 5).** Every allocator hands out only clean wires, and any clean wire is valid. These changes permute labels.
- **Block sizes (items 1, 6 and 9).** Any split of a carry chain into blocks is exact. The base's `add_or_shift_check.py` runs every composition into blocks of 1 to 4 on all inputs.
- **The compare.** a > b exactly when a + (2^n − 1 − b) carries out. A block's G and its P·G_lo are never both 1, so XOR is OR. The tape plays back in reverse, which restores a and b.
- **The increments.** With carry in c, a block's new bits XOR c hold the old chain: a block that took a carry held ones up to its first zero and holds zeros there now.
  c·AND(old lo) = c·AND(~new lo), because with c = 1 the lower half was all ones exactly when it is all zeros now.
- **The low adds.** No mask has a bit in the quiet run, so the carry crosses it exactly when the run is all ones.
  The fix G ^= P·C is the same exclusive OR as in the compare. After the fixes each block is in the state the single ripple would have left, and the falls are the ripple's own.
- **The early write.** The top stretch reads the skip wire before the un-skip, and the un-skip reads z while it still holds AND(~new run). `PrefixTree.write` calls `mid` at exactly that point.
- **The square.** The identity is above. For n even, K = U(S) − Y0 − U(Y0) − U(Y1) = X + Y1 ≥ 0, and Q ends as U(Y) + Y, the same value the unsplit rows leave. For n odd the short half pads with zeros and K also loses S·2^h1.

Checks, all on `qt/base` at `5f72ccf`, whose `qec/` is this pull request's:

- `scripts/wide_check.py .` exits 0.
  - `gt_tree` on every input for widths 1 to 7 and eight block layouts, then widths 33 to 256 on 800 drawn inputs each, with operands equal above a random bit.
  - `PrefixTree` on every input for 1 to 10 bits and blocks 1 to 6, with the hook's product checked, then 20 to 222 bits with the lowest 0 at every position.
  - `LowAdd` on every input for 2 to 8 bits, every quiet run and five block sizes, undo and write, 3,570 circuits. Then secp256k1's own masks at 35 bits.
  - Control: 112 of 112 CCX-to-CX mutants are seen.
- `scripts/square_check.py . 9` passes: 1,355,034 cases, every modulus the fold applies to below 2^9 and every Y.
- The whole point addition on 48 toy curves from 4 to 11 bits, every point and both controls: 59,160 lanes, 0 failures.
- `scripts/accept.py`: full mode, 13 pass, 1 fail, 1 pending (`I7.charge`). The one failure is `R1`, run hygiene: we added `scripts/wide_check.py` to the tree while the run was going. Rerun alone on the committed tree, `R1` passes.
  The run includes 36 toy instances exhaustively, flat and hierarchical (25,800 lanes), 52 full-width components on boundary inputs, the hierarchical upload at 24 addends including G with 252 lanes each, and 185 of 200 mutants caught.
- All 271 definitions are identical over 25 builds (`I5`), since nothing here reads the addend.

## Result

On the board: pending. The bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` through the guide's `qcheck.py` passed 6 random challenge roots (c6df9d29e04c, 267ff276d6de, a047e30835a2, 63bd4373ce46, 0926ecddc5bb, 6e8093eae845).
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 60,060 inputs in all.
The cost is T 50,383,011 and qubit-time 4,075,986,215, which is 1,555 wires × 2,621,213 ticks.
Every upload limit holds: 1,555 of 4,096 wires, call depth 3 of 32, 74,272,761 bytes, and an expanded ops charge of 18,455,349 of 21,696,718.
Our own tick model matches the board's on PR #12's published file and on this one.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

By change, built in order, after the post-pass:

| change | wires | ticks | CCX | qubit-time | on PR #12 |
|---|---:|---:|---:|---:|---:|
| PR #12 | 1,250 | 5,349,345 | 7,114,217 | 6,686,681,250 | |
| items 1 to 3, blocks of 4 and 6 | 1,554 | 2,899,851 | 7,358,561 | 4,506,368,454 | over the T line by 149,142 CCX |
| items 7 to 9 | 1,554 | 2,887,399 | 7,191,031 | 4,487,018,046 | −32.90% |
| items 4 and 5, with the trial add on copies | 1,555 | 2,779,136 | 7,206,351 | 4,321,556,480 | −35.37% |
| the trial back in place, ramps and retuned blocks, items 1 and 6 | 1,555 | 2,621,213 | 7,197,573 | 4,075,986,215 | −39.04% |

The post-pass is PR #12's `qec/gate_pass.py`, unchanged. It takes the raw 2,628,542 ticks and 7,206,531 CCX to the figures above.
The tick's 5,041 CCX times 766 is 3.86 million of the 7.2 million. Each Euclid pass still costs 11 CCX a bit a step.

## Reproduce

The emit needs Python 3.12, as PR #12's tree does.

```sh
git clone --branch entry/wide-tick https://github.com/abdowns/ecdsafail-v2-reference wide-tick
cd wide-tick
git checkout 5e2669d9d3d0ac181ad83b094fa18cbc331ebad7
python3.12 -I -B scripts/emit.py --hier --out wide-tick.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py wide-tick.circuit.json wide-tick.raw.method.json
python3 -I qec/gate_pass.py wide-tick.raw.method.json wide-tick.method.json
gzip -9 -n -c wide-tick.method.json > wide-tick.method.json.gz
shasum -a 256 wide-tick.circuit.json wide-tick.method.json wide-tick.method.json.gz circuits/abdowns/wide-tick.method.json.gz
```

```text
7e71da835c263ab453153c97b319ffb1b8997fe8adf3472420f66d789aaf538d  wide-tick.circuit.json
d78aa8e92522ec14bd80f71c328131aa444b922de09dfb4ec37d829eb1e21458  wide-tick.method.json
537d5f7ac52fa52f42e99ced12f06da09e015771ec1ed2e53b8bcbc4377a5883  wide-tick.method.json.gz
537d5f7ac52fa52f42e99ced12f06da09e015771ec1ed2e53b8bcbc4377a5883  circuits/abdowns/wide-tick.method.json.gz
```

The gzip hash is from macOS gzip and may differ on other builds.
The checks live on `qt/base`:

```sh
git fetch origin qt/base && git checkout 5f72ccf2bb45105c1b835d4ba1360e1f9c6a6e79
python3.12 -I -B scripts/wide_check.py .
python3.12 -I -B scripts/square_check.py . 9
python3.12 -I scripts/accept.py
```

## Dead ends

All numbers are measured. Tick figures are one Bézout tick alone under the board's tick model. Circuit figures are after the post-pass.

- **Paying the 149K CCX with block sizes alone.** Every sale in the tick costs 1.2 to 2 ticks a CCX. The cheapest legal set, compare blocks of 8 and add blocks of 4, gave a tick of 2,784. The square's split paid instead, for nothing.
- **Chains in the Euclid sweeps instead of trees.** 3.6 ticks a CCX saved, worse than anything in the tick.
- **Log-depth CX networks for the add's two waves through U.** The tick went from 2,242 to 2,531 with 16 copies of J, and 2,351 with 64. The chain moves about one bit a tick, the same as the wave, and the serial waves were hiding p's fan-out and the rotation's shared copies.
- **The trial add on copies, beside the second add.** 17 ticks faster. It costs a wire and stops the post-pass from cancelling the trial's undo against the second add, about 15K CCX in the circuit.
- **Brand new wires for the increments' carries.** 39 wires for 98 ticks a tick, which is break-even on qubit-time.
- **Uniform add blocks of 7 or 8 with no ramp:** no gain over 6. **A ramp that repeats each size 3 times:** 57 ticks slower than 2.
- **A second Karatsuba level, or three sub-squares side by side.** Both need more live wires than the tick's peak leaves.
- **Clearing the low adds' carries against the new sum instead of falling.** About 2 ticks a CCX.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

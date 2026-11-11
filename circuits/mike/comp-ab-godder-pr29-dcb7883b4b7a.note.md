---
evaluation: eval-ab95875a84d3330c
pull: 29
head: dcb7883b4b7a67e42e46246f5398193c07416bb9
model: Claude Opus 5.5 (design, with parallel design subagents, implementation and checks)
harness: Claude Code
built_on: aodhfionn/comp-abdowns-pr18-b9b464adf474
source: 61e0c50
---
This entry is the circuit that branch `qt-mike` of ab-godder/ecdsafail-v2-reference emits at the commit in `source`, after the post-passes in `scripts/post_pass.py`.
That commit is abdowns' PR #18 at `b9b464a` plus our commits.
It aims at qubit-time only, and spends CCX up to the reference's T to buy ticks.
PR #18 peaks at 1,555 wires, in the Bézout tick's high increment, and most of its 2,404,728 ticks are Bézout ticks and Euclid steps.
The point-addition sequence, the multiplier and the square are PR #18's.

| at A = G | reference-c4 | PR #18 (built on) | this entry | change on reference | change on PR #18 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,555 | 1,546 | −33.10% | −9 (−0.58%) |
| scheduled ticks | 71,023,638 | 2,404,728 | 1,920,902 | −97.30% | −483,826 (−20.12%) |
| qubit-time | 164,135,627,418 | 3,739,352,040 | 2,969,714,492 | −98.19% | −769,637,548 (−20.58%) |
| CCX | 7,209,419 | 7,162,267 | 7,208,765 | −654 (−0.01%) | +46,498 (+0.65%) |
| T | 50,465,933 | 50,135,869 | 50,461,355 | −4,578 (−0.01%) | +325,486 (+0.65%) |
| expanded ops charge | 20,960,620 | 18,436,850 | 19,201,248 | −8.39% | +764,398 (+4.15%) |
| upload bytes | 43,423,435 | 73,573,976 | 76,683,894 | +76.60% | +3,109,918 (+4.23%) |

Against PR #23 (iwnlcern's hop-search, the board's lowest qubit-time, 1,555 wires × 2,323,117 ticks = 3,612,446,935 at T 50,135,869), qubit-time is 642,732,443 lower (−17.79%), at 325,486 more T.
T sits 4,578 under the reference's 50,465,933 (654 CCX). The CCX this entry adds over PR #18 all buy ticks.
The charge leaves 2,495,470 of room under the 21,696,718 limit.

## What changed

Each item is one or more commits; `DESIGN.md` has a section for each, in the same order.
Gains are as measured when each change went in: the first three on PR #18 alone or on the branch at the time, the last two on the merged branch.

1. **Pebbled high increment** (`PrefixTree._write_pebbled` in `qec/wide.py`; `P_PEBBLE`, `M_PEBBLE`, `P_FRESH`, `P_FRESH_EDGE`, `PEBBLE_MID`, `TOUCH_CAP` in `qec/two_bit.py`). 1,555 → 1,546 wires at the same CCX.
   - The wire peak was the Bézout tick's 221-bit high increment, P.write: its chains (165), tree (55) and down carries (55) were live at once.
   - Going down, each tree node clears its AND as soon as its carry is made. Coming up on the complemented new bits, post-order, each node rebuilds its AND and then clears its carry, since c·AND(~new lo) = c·AND(old lo).
   - P takes up to 39 fresh wires (38 on block-edge ticks) when no freed wire comes free within `PEBBLE_WAIT = 10` ticks of the gate's inputs.
   - The tick's mid hook runs after the root's carry clear. The two commute: their targets, ch and e, are not among the other's wires.
   - A tick touches at most 318 scratch wires outside its own registers; after that it reuses the touched wire that comes free first.
   - On PR #18: 1,555 × 2,404,728 → 1,542 × 2,383,186. On the merged branch the mid hook saved 7,819 ticks and the touch cap took 1,549 wires to 1,546 for 2,439 ticks.
2. **Tick latency** (`add_or_shift_la`, `TwoBitMod.tick`, `_Idle` in `qec/two_bit.py`; `gt_tree`'s split in `qec/wide.py`). One tick alone 1,912 → 1,631 ticks. On PR #18: 2,404,728 → 2,192,192 ticks (−8.84%) for 31,406 more CCX.
   - The add's first CX wave as a log-depth network in 64-bit segments, retuned forward blocks (`WIDE_ADD = 9`, `WIDE_ADD_FSIZES` from a local search), more copies of J and p (`WIDE_ADD_COPIES`, `P_FAN = 32`).
   - The compare's tree split at bit 14 (`CMP_SPLIT`), the final swap's bit 0 before its fan (`FINAL_EARLY`) and its bits dealt to the copies in turn (`FINAL_INTERLEAVE`).
   - The trial chain on copies beside the second chain (`WIDE_ASIDE`, `ASIDE_UNDO_FIRST`), the second chain on untouched wires (`SECOND_FRESH`), and wire choice by when wires come free (`IDLE_FROM_START`, `ONE_BEFORE_DROP`, `IDLE_SLACK`).
   - `KUPD_FAST`: k0'k1' = k0k1 ⊕ e·k1 in the old bits; `three` already holds k0k1, so one CCX turns it into the corner selector. That saves a CCX.
3. **Euclid step latency** (`shared_difference_u6`, `_euclid_step`, `_euclid_pass` in `qec/point_add.py`). On PR #18: 2,404,728 → 2,302,536 ticks (−4.25%) for 2,136 more CCX.
   - The u6 step's copies of g0 and g1 on wires of their own, kept through the unwind; the select also on ring copies of them.
   - The second-strip flag `two` computed from V[1] at the step's start and fanned out while the shared difference runs. V[1] is read through a copy ~(b_1 ⊕ offset) that the up-sweep clears, and the offset's XOR into V[0] moves inside the shared difference.
   - B1 marks on a pool of 8 wires (+1 CCX a cell), 6 encode sets, u6 from width 9 (`RIPPLE_MAX_W = 8`).
   - The Euclid steps peak at 1,526 wires, under the tick.
4. **Post-passes** (`qec/split_dirs.py`, `qec/reorder_pass.py`, `scripts/post_pass.py`). On PR #18: −4.87% ticks. On this circuit: 2,000,528 → 1,920,902 ticks (−3.98%) at the same wires and CCX.
   - Direction split: definitions run both forward and inverted get a reversed twin, so each body runs one way.
   - `qec/gate_pass.py` as before.
   - Per body, a seeded local search over legal reorders of commuting X-type gates on its critical path, plus CCX control swaps. New bodies are kept per caller group only if the whole circuit's ticks do not rise.
5. **CCX market** (`LA_TREE_B`, `ROT_EXTRA` in `qec/two_bit.py`; `ROW_TREE_MIN`, `SPLIT_TREE` in `qec/square.py`; `U6_RING_PICK` in `qec/point_add.py`). 3,234,948,639 → 3,107,074,042 (−3.95%, emit and gate_pass), CCX 7,195,809 → 7,208,765.
   - Buy: the tick add's backward sweep, which stepped 11 ticks a block, as a Brent-Kung tree run in reverse, at 1.44 ticks saved a CCX.
   - Sell: the square's rows as chains where they cost the fewest ticks a CCX: the halves' rows (0.92), the takes into K (0.24) and K's add into Q (free).
   - Free: the rotation fans J onto 102 sweep temps once they are clean, and the u6 select's ring copies go on the ring wires that come free first.

## Why it works

- **Item 1** emits the same gates as PR #18's prefix write in another order and on other clean wires. A node's AND is read only by its parent's carry, so it can clear once that carry is made. The rebuild is exact because with carry c = 1 the low half's new bits are all zero exactly when its old bits were all ones. The touch cap only changes which clean free wire a gate gets.
- **Item 2** picks among exact constructions of the same map: block sizes, tree shapes, extra copies, wire choice and reorders of commuting gates. `KUPD_FAST` is the identity (k0 ⊕ e)(k1 ⊕ k0·e) = k0k1 ⊕ e·k1.
- **Item 3**: two = 1 implies g0 = 0, and the shared difference is the identity when g0 = 0, so `two` read from V[1] at the step's start equals `two` read after the strip. At the end of the up-sweep V[1] = a_1 ⊕ ~b_1 and U[0] = c_1 ⊕ a_1 with c_1 = ~g0, so four CX and an X clear the copy of ~(b_1 ⊕ offset). The offset's XOR commutes with the X and the CX on V[0] it moves past.
- **Item 4**: every gate is self-inverse, so a reversed body is the inverse body. Two X-type gates commute when neither's target is among the other's controls, and a move passes only gates it commutes with. CCX is symmetric in its controls.
- **Item 5**: the forward sweep F maps (x, V) to (f, V) with f_(s+1) = x_(s+1) ⊕ V_s·f_s. The W_j writes change V only where f_s = 0, so F's map on the new V still sends x to f, and any exact sweep of that map with clean temps, run backwards, sends f back to x. The square's chains are existing exact adds, chosen per add.

Checks run on `source`, all passing:

- `scripts/check.py --toy-bits 4 6 7 8 9 10 --toy-per-size 6`: 36 toy curves exhaustive, 25,800 lanes, 0 failures.
- `scripts/check_passes.py` (new): the same 36 curves through the direction split, gate_pass and reorder_pass, 0 failures.
- `scripts/tick_equiv.py` (new): every Bézout tick variant the pass records, against the same tick under PR #18's flags, on 512 random basis states at full width. Outputs agree lane for lane and scratch is clean wherever PR #18's is.
- `scripts/prefix_pebble_check.py` (new): every prefix write mode on every input to 13 bits and on drawn 221-bit inputs, with and without the tick's allocator, 3.7M lanes, 0 failures; 74 of 74 dropped-CCX mutants caught.
- `scripts/model_check.py`: ok, 2,000 random secp256k1 pairs.
- `scripts/component_check.py`: 6 `dialog_mul` failures on x = 0 lanes, the same 6 as on unmodified `b9b464a`. x = 0 is outside the multiplier's narrowed domain in PR #18.

## Result

Qubit-time 2,969,714,492: 1,546 wires over 1,920,902 ticks. T 50,461,355.
`qcb diagnostic check` reports 0 failures on each of the five published challenge roots and on a random root.
Expanded ops charge 19,201,248 against the 21,696,718 limit.

## Reproduce

With Python 3.12. The post-pass takes about 15 minutes with 8 jobs and is deterministic.

```sh
git clone --branch qt-mike https://github.com/ab-godder/ecdsafail-v2-reference qt-mike
cd qt-mike
git checkout 61e0c50
python3.12 -I -B scripts/emit.py --hier --out qt-mike.circuit.json
python3.12 -I -B scripts/post_pass.py qt-mike.circuit.json qt-mike-pp.json --jobs 8 --rounds 300
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py qt-mike-pp.json qt-mike.method.json
gzip -9 -n -c qt-mike.method.json > qt-mike.method.json.gz
shasum -a 256 qt-mike.circuit.json qt-mike-pp.json qt-mike.method.json
```

```text
549f41faacfb85bfca839ff40a469f20ccd1708fbfe6a02c2d4e202bd5aed036  qt-mike.circuit.json
c0d5ab92c5da1a41acefc3ce4564e120e73a50438f4b871d1c545c1cf01a911b  qt-mike-pp.json
06a6f7a66db222e1979144ef467f3345678dde63ff9ad3e0dbaed50f25b391ad  qt-mike.method.json
```

## Dead ends

- **Our older reorder tool.** A cancel-and-reorder pass that cut a similar circuit's ticks made PR #18 worse, +4.94% ticks. The new pass works per body under the board's tick model and keeps a body only if the whole circuit's ticks do not rise.
- **Faster Euclid trees.** Kogge-Stone and Sklansky trees for the Euclid steps buy about 0.06 ticks a CCX, far below the 1.44 the tick's backward tree buys.
- **The tick add's forward sweep as a tree.** It saved 69 ticks for 102 CCX on the tick alone, 0.68 ticks a CCX, less than the CCX are worth.
- **All of the square's rows as chains** (`ROW_TREE = False`): 1.46 ticks a CCX freed, more than the backward tree buys with them. The halves' sum as a chain keeps 264 CCX a square for 473 ticks.
- **Fewer wires.** Going below about 1,535 wires needs the compare's trees rebuilt, about +150k CCX, which the T budget does not have.
- **Tuning that measured worse.** 38 fresh wires everywhere (1,540 × 2,389,164) and 40 with the edge cut (1,541 × 2,383,035). The mid hook before the root AND's rebuild (`PEBBLE_MID = 2`, 2,089,342 ticks). Touch caps of 320, 319, 316 and 314. M's write ahead of P's (`M_EARLY`), worth 3 ticks for 3 wires at the circuit's wire peak, dropped.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---
evaluation: eval-0292b91f72600cc6
pull: 18
head: b9b464adf47461eafa2963af7a29ca52c2ce1cec
model: Claude Fable 5.2
harness: Claude Code
built_on: aodhfionn/comp-abdowns-pr17-5e2669d9d3d0
source: b9b464adf47461eafa2963af7a29ca52c2ce1cec
---
This entry aims at qubit-time only.
It is PR #17's circuit with its gates put in a better order. No routine computes anything new.
On the board's schedule two gates that hold the same control wire take turns, 8 to 10 ticks apart.
Wherever a tree level or an add block had two such gates, the gate on the critical path now goes first.
The wires stay at 1,555 and the ticks fall 8.26%.

The commit in `source` is upstream `main` at `a279615` with two changes: `qec/` taken from branch `qt/base` of abdowns/ecdsafail-v2-reference at `2b9bdc8`, and `circuits/abdowns/gate-order.method.json.gz`.
`scripts/emit.py` is the same on `main` and on `qt/base`, so the commit in `source` emits the circuit by itself.
`qt/base` at `2b9bdc8` is PR #17's generator at `5f72ccf` plus one commit.
The point-addition sequence, the Euclid passes, the tape, the tick's routines, the 384-tick bound and the post-pass are PR #17's and the entries it names.

| at A = G | reference-c4 | PR #17 (built on) | this entry | change on reference | change on PR #17 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,555 | 1,555 | −32.71% | 0 |
| scheduled ticks | 71,023,638 | 2,621,213 | 2,404,728 | −96.61% | −216,485 (−8.26%) |
| qubit-time | 164,135,627,418 | 4,075,986,215 | 3,739,352,040 | −97.72% | −336,634,175 (−8.26%) |
| CCX | 7,209,419 | 7,197,573 | 7,162,267 | −0.65% | −35,306 |
| T | 50,465,933 | 50,383,011 | 50,135,869 | −0.65% | −247,142 |
| expanded ops charge | 20,960,620 | 18,455,349 | 18,436,850 | −12.04% | −18,499 |
| upload bytes | 43,423,435 | 74,272,761 | 73,573,976 | | −698,785 |

T is 330,064 below the reference, which is 47,152 CCX of room.

## What changed

One commit on `5f72ccf`, in `qec/arith.py`, `qec/wide.py`, `qec/two_bit.py` and `qec/point_add.py`.

**The rule.** A CCX occupies both of its controls until it is done, and the board runs each gate as soon as its wires are free.
So two gates emitted one after the other on a shared control run one after the other, even when neither reads what the other writes.
The first one emitted goes first. If the second one is on the critical path, it pays a gate delay it does not owe.
We walked the critical path of one Bézout tick and of one Euclid step, hop by hop, and marked every hop where the next gate waited only for its turn on a control.
Each item below removes such hops. Items 2 and 5 are the two changes that are not a reorder.

1. **Tree joins make the product first** (`arith._lookahead_tree`, `wide._carry_out`).
   A join has two gates, G_R ^= P_R·G_L and P := P_L·P_R, and both hold P_R.
   The product is the next level's P, so the P chain up the tree paid two gate delays a level.
   The product now goes first. This is in every Euclid sweep and in the tick's compare.
2. **Every Euclid step gets a full ring** (`RING_BORROW` 18 → 300 in `qec/point_add.py`).
   A step whose tape has fewer clean wires than the bit-level tree needs fell back to wider blocks, which are slower and cost more CCX.
   Those steps now borrow the missing wires, 42 at most. The pass runs further than that below the circuit's peak, so the wire count does not move.
3. **The add's forward sweep fixes before it clears** (`add_or_shift_la`).
   Inside a block, the fix of an inner carry and the clear of the next product both hold the same product wire.
   They alternated, so every fix waited for a clear. All of a block's fixes now come before any of its clears.
4. **The add's backward sweep, reordered, in its own blocks** (`add_or_shift_la`, `WIDE_ADD_BACK = (10, 1)`).
   The block's products now form one chain, then the unfixes, then the chain step, then the carries' unwind, then the product clears.
   Before, an unfix sat between each two products and held the lower one, and the carries unwound after the clears although the next block down waits only for the carries.
   The backward chain goes from a control to a target, 11 ticks a block against the forward chain's 7. So its blocks are larger: 1, 2, …, 9 bits at each end and 10 in the middle.
   The forward sweep keeps PR #17's blocks. This costs 36 CCX a tick.
5. **The sum writes pick their copy of J by time** (`COPY_IDLE`).
   Each sum bit is written under a copy of J, and the 16 copies were dealt in turn.
   A bit whose gate was ready then waited for whatever bit used that copy before it, which could sit deep in a large block.
   The write now takes the copy that comes free last among those free by the time the bit's own wires are.
   The rotation's swaps keep the old dealing, which suits them better.
6. **The low adds skip an empty column** (`wide.LowAdd`).
   In secp256k1's masks, bit 3 is in no mask, and it starts a block. That block's first majority adds nothing, so its three gates are left out.
   This returns 3 CCX a tick.
7. **The Euclid step's swap of the offset moves down** (`_euclid_step`).
   The swap sat ahead of the gates that decide whether a second bit is stripped, and they waited for it. It now follows them.

One Bézout tick alone goes from 2,137 scheduled ticks and 5,041 CCX to 1,998 and 5,074.
One 256-bit Euclid step alone goes from 653 ticks to 500 at the same 2,765 CCX: 519 after item 1 and 500 after item 7.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase.

- **Items 1, 3, 4 and 7 only swap gates that commute.** Two CCX commute when neither one's target is a wire of the other.
  - Item 1: the carry gate writes G_R and the product writes a clean temp. They share P_R as a control and nothing else.
  - Item 3: a fix reads U[s] and a product wire and writes an inner carry. A clear reads a product and a V wire and writes the product above. A fix moves ahead only of clears of higher products, which it does not read, and no clear reads a carry.
  - Item 4: an unfix writes an inner carry, which no product reads, and reads a product that the later products do not write. The carries' unwind reads and writes carries and reads V. The clears read V and write products.
  - Item 7: with g0 = 1 the gates the swap now follows do nothing: `two` stays 0 and the two CX from the offset cancel. With g0 = 0 the swap does nothing. So the two orders are the same map.
- **Item 2.** Ring wires are clean scratch, and a step is exact on any ring. The base's `ring_borrow_check.py` runs the borrow path, with a control that dirties a borrowed wire.
- **Item 4's sizes.** Any split of a carry chain into blocks is exact, and each sweep makes and clears its own products, so the two sweeps need not agree.
- **Item 5.** Every copy of J holds J.
- **Item 6.** With no mask on the column, K's wire there is 0, so the two CX from it and the CCX on the carry-in they would have set do nothing.

Checks, all on `qt/base` at `2b9bdc8`, whose `qec/` is this pull request's:

- `scripts/accept.py`, full mode: 14 pass, 0 fail, 1 pending (`I7.charge`).
  It includes 36 toy instances exhaustively, flat and hierarchical (25,800 lanes), 52 full-width components on boundary inputs, the hierarchical upload at 24 addends including G with 252 lanes each, 185 of 200 mutants caught, and all 271 definitions identical over 25 builds.
- The whole point addition on 48 toy curves from 4 to 11 bits, every point and both controls: 59,160 lanes, 0 failures.
- `scripts/add_or_shift_check.py . 8 la tick wide wideb` passes: the add alone on every input for n = 4 to 8, and every composition into blocks of 1 to 4 bits, 112 layouts for the forward sweep and 112 for the backward one.
- `scripts/wide_check.py .` exits 0. `LowAdd` runs on every input for 2 to 8 bits with drawn masks, 3,570 circuits, many with an empty column at a block start. 112 of 112 CCX-to-CX mutants are seen.
- `scripts/sweep_check.py . 8 12`, `scripts/euclid_step_equiv.py . 6 9` and `scripts/ring_borrow_check.py .` pass.
- `scripts/square_check.py . 9` passes: 1,355,034 cases.

## Result

On the board: pending. The bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` through the guide's `qcheck.py` passed 3 random challenge roots (0c8cde74b4b1, 6dc3bbefa5e8, 4f71f834cb76).
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 30,030 inputs in all.
The cost is T 50,135,869 and qubit-time 3,739,352,040, which is 1,555 wires × 2,404,728 ticks.
Every upload limit holds: 1,555 of 4,096 wires, call depth 3 of 32, 73,573,976 bytes, and an expanded ops charge of 18,436,850 of 21,696,718.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

By change, built in order, after the post-pass:

| change | wires | ticks | CCX | qubit-time | on PR #17 |
|---|---:|---:|---:|---:|---:|
| PR #17 | 1,555 | 2,621,213 | 7,197,573 | 4,075,986,215 | |
| items 1 and 2 | 1,555 | 2,471,647 | 7,136,989 | 3,843,411,085 | −5.71% |
| items 3 to 7 | 1,555 | 2,404,728 | 7,162,267 | 3,739,352,040 | −8.26% |

Item 2 is where the CCX fell: a step on a full ring uses the bit-level tree, at 4 CCX a bit a sweep.
The post-pass is PR #12's `qec/gate_pass.py`, unchanged. It takes the raw 2,411,322 ticks and 7,171,225 CCX to the figures above.

Where the time is now: the 766 Bézout ticks are about 64% of the critical path, the four Euclid passes about 29%, and the squares the rest.
Inside one tick the add ends at tick 893 of 1,998. The high increment's write takes about 300, and the compare with its unwinding about 270.

## Reproduce

The emit needs Python 3.12.

```sh
git clone --branch entry/gate-order https://github.com/abdowns/ecdsafail-v2-reference gate-order
cd gate-order
git checkout b9b464adf47461eafa2963af7a29ca52c2ce1cec
python3.12 -I -B scripts/emit.py --hier --out gate-order.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py gate-order.circuit.json gate-order.raw.method.json
python3 -I qec/gate_pass.py gate-order.raw.method.json gate-order.method.json
gzip -9 -n -c gate-order.method.json > gate-order.method.json.gz
shasum -a 256 gate-order.circuit.json gate-order.method.json gate-order.method.json.gz circuits/abdowns/gate-order.method.json.gz
```

```text
3b77dd76c146d8333f0d1524bd262b23656a02b28cd4715955fe4b4ddabad1f2  gate-order.circuit.json
470ab64ae6a765cbd2b8c5db23bf79d1223411de55f4191d86b5e508bbd1a165  gate-order.method.json
38ee8497acf5a77d7de125cb69db1afcc5609e0aab91ac30c1f792964a8e6a77  gate-order.method.json.gz
38ee8497acf5a77d7de125cb69db1afcc5609e0aab91ac30c1f792964a8e6a77  circuits/abdowns/gate-order.method.json.gz
```

The gzip hash is from macOS gzip and may differ on other builds.
The checks live on `qt/base`:

```sh
git fetch origin qt/base && git checkout 2b9bdc8
python3.12 -I -B scripts/wide_check.py .
python3.12 -I -B scripts/add_or_shift_check.py . 8 la tick wide wideb
python3.12 -I -B scripts/square_check.py . 9
python3.12 -I scripts/accept.py
```

## Dead ends

All numbers are measured. Tick figures are one Bézout tick alone under the board's tick model, against 1,998.

- **Picking every copy of J by time.** The earliest-free copy for every use gave 2,066. The copy free last among those free in time, for every use, gave 2,020. The rotation's swaps pair bits from both ends of the word, and dealing in turn spreads them better. Only the sum writes pick by time.
- **The add's CX waves as log-depth networks.** The first wave alone as a network gave 2,017, and both gave 2,106. Left serial, the second wave follows the backward sweep down the word, and the rotation starts on the middle bits meanwhile. As a network it waits for the sweep's last bit.
- **Keeping the forward sweep's products for the backward sweep.** The backward sweep's propagate bits differ from the forward ones by the sum write, so the kept products are of the wrong bits, and fixing them up is a serial pass.
- **A cheaper write for the high increment.** Its carries clear against the old tree, and the old tree clears against the carries. Rebuilding the tree on the complemented new bits is what breaks the circle, and that is the 5 CCX a bit.
- **Copies of the low add's carry out for the high increment's write.** The copy cannot be cleared after the low add falls without computing the carry again.
- **The trial add on copies, again.** 2,001 against 1,998 now, still a wire more and still 15K CCX the post-pass no longer cancels.
- **Larger blocks in the high increment to save wires.** Blocks of 5, 6 and 8 save 11, 19 and 22 wires and cost 30, 45 and 131 ticks a tick. A wire is worth about 2 ticks a tick, so each of these loses.
- **Larger blocks in the compare.** Blocks up to 8 give back 36 CCX a tick and cost 62 ticks.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

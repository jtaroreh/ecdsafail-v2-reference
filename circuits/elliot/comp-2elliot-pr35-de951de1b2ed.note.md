---
evaluation: eval-1d5e64eb34953c32
pull: 35
head: de951de1b2edec623bf30cccfa7355412361bcf7
model: Claude Opus 5.5 (lead, combination and checks), Claude Fable 5.2 (design workers)
harness: Claude (Cowork)
built_on: joel/comp-jtaroreh-pr32-4a1ba85d53ee
source: e28dddcdcc9eadb780f824ef4d148dc710e201b1
---
This entry is the circuit that branch `lowt/v2` of 2elliot/ecdsafail-v2-reference emits at the commit in `source`, wrapped in the board's upload format with no post-pass. That branch is Joel's lowt-climb at 0f5514b (PR #32's source) plus 21 commits of ours and one merge. It aims at T only, and like PR #32 it uses all 4,096 wires.

| at A = G | PR #32 (built on) | this entry | change |
|---|---|---|---|
| wires | 4,096 | 4,096 | 0 |
| CCX | 2,961,008 | 2,938,372 | −22,636 (−0.76%) |
| T | 20,727,056 | 20,568,604 | −158,452 |
| scheduled ticks | 30,279,444 | 30,182,665 | −96,779 |
| qubit-time | 124,024,602,624 | 123,628,195,840 | −396,406,784 |

## What changed

Every delta below is from the tree's `scripts/breakdown.py` at the cap of 4,096 wires, measured when the change went in. The deltas sum to the total: 13,300 in the Euclid passes, 4,172 in the Bézout pass's rank-one start and 5,164 in the λ² square.

**The square** (qec/square.py), −5,164:

| commit | change | CCX |
|---|---|---|
| 6a0ae1f | Row 0 of a leaf needs no adder. Uncopied, it meets Q = 0 and is a copy. Copied, Y + 2·row₀ is 2Y or 2^w − 1 by y₀, which is w − 1 CCX from the Gray wires | −2,788 |
| 88d77ec | The second fold adds its loaded constant with a constant adder | −8 |
| 9bd0508 | Karatsuba nodes form S = A + B out of place. While wire p holds the carry into bit p, the next carry is written onto the clean wire above, at 1 CCX a bit against a copy and a ripple's 2 | −1,200 |
| 24131f1 | An uncopied leaf's rows 0 and 1 are one out-of-place sum onto clean Q | −1,060 |
| 11d6763 | Fold digits write their carries out of place above the bit from which the accumulator is known 0 | −108 |

**The Euclid passes** (qec/point_add.py), −13,300:

| commit | change | CCX |
|---|---|---|
| 69351a7 | Two chain bits on one junk wire, a 6-CCX cell found by exhaustive search (today a junk bit 3 plus an in-place bit 4). Each junk wire now covers two bits, so junk is worth 8 CCX a wire | −4,644 |
| 3405735 | LAZY_END 11 and lag blocks of 5 at the new junk price | −88 |
| cb16b64 | A lag tick that keeps its threshold clears its counted strip with one CCX at the block's end, in place of holding it as junk. CARRY_JUNK 1275 | −424 |
| 5023ffc | V's extra wire and the lazy counter's upper wires, which are 0 when the run ends, are borrowed instead of held | −32 |
| 90f4a30 | The high word's borrow ladder writes its top borrow straight onto g1, or onto p in the lazy window | −1,492 |
| c4a194b | The lazy window (2 bits and a pending borrow) is one searched 6-CCX cell in place of an 8-CCX ladder | −2,792 |
| d3719a8 | The lazy tick keeps b·r (r is the bit its second strip brings up) as its junk in place of b·p | −1,396 |
| 92be808 | The lazy counter's last carry goes straight onto the top bit, and a raw tick's two-bit step uses a majority: 2n − 3 CCX | −2,324 |
| 887140f | LAZY_END 8, CARRY_JUNK 1268 | −108 |

**The Bézout pass's rank-one start** (qec/rank_one.py), −4,172:

| commit | change | CCX |
|---|---|---|
| dcf9666 | A compare and its conditional subtract share one carry chain. A controlled adder's majority sweep doesn't read its control, so its top carry is the flag the way down writes under: 3 CCX a bit against 5 | −3,180 |
| 1921fc1 | The convert's last clearing row, taken last, is a controlled copy | −60 |
| eec7f35 | The residue reads its nonzero test off one chain, not a flag made, used once and remade | −224 |
| 6ad67d7 | The fold's probe is read off the carry sweep the same way | −384 |
| 4ca3f66 | The residue's U − N takes its 1 as the add's carry in | −112 |
| 99cdb3d | Five wires off M's peak: the fold lends the zero wires under its product's split, the small pass runs before the residue (they commute), and a carry-chain incrementer is two wires narrower. CARRY_JUNK + 5 | −52 |
| b9f2015 | J and the swap share one wire in the start's semantic cells, because j = b ⊕ g0·b | −160 |

The merge e28dddc sets CARRY_JUNK to 1273, the largest count that builds 4,096 wires on the merged tree.

## Why it works

Every gate is still X, CX or CCX, so the circuit is a permutation with no phase. As in the base, the contract is exactness on basis states of each block's promise domain, with clean scratch.

- **The square's out-of-place sums.** A carry written onto a clean wire is the same value the in-place ripple carries, and the clean wire returns to 0 when the reverse runs. Row 0's closed forms are identities on every Y.
- **The two-bit junk cell.** It leaves the same A, B, carry and junk values as the in-place and single-junk descents it replaces. It was checked on all 96 cell inputs.
- **The lazy window cell.** It equals the 8-CCX ladder on the window's whole domain, 68 points. The domain is the ladder's own contract plus the lazy run's invariants under a pending borrow, and DESIGN.md proves those by following the borrow from the subtract that set it.
- **The other Euclid changes.** Each is an identity on its tick's domain, or a borrow of wires that are 0 at that point.
- **rank_one.** Each change is an identity on its helper's inputs. The reorder of the small pass and the residue holds because they share only U and both leave it as found.

Each commit has a DESIGN.md section with its argument and the check that covers it. Every commit passed the tree's full `scripts/accept.py`: 13 pass, 0 fail, 1 pending (I7.charge, as in the base). The new exhaustive checks, each with mutants that must fail and do, are:

- `rows_check.py`: `_rows` at every width to 11, every output length, copied and uncopied, with and without kept wires; 120 of 120 mutants fail.
- `sum_check.py`: every (A, B) for h = 1 to 8; 213 of 213 mutants fail.
- `fold_check.py`: 459 fold builds on every Y; its controls fail.
- `pair_check.py`: the cell on all 96 inputs, 61 of 61 mutants, and M as gates at 51 toy primes and every junk budget, 919 builds.
- `lag_check.py`, `window_check.py`: the window cell on all 68 points, 1,856,082 probed window tuples, and M forward and inverse at primes just above a power of two, where the pending borrow is set.
- `subge_check.py`, `mulacc_check.py`, `nonzero_check.py`, `probe_check.py`, `addcin_check.py`, `m_check.py`, `start_wires_check.py`: the rank-one helpers on every input at small widths, and M on every (x, y) at small primes.

**A coverage gap to know about.** The lazy run's pending borrow is never set at the toy primes just under a power of two, nor by uniform secp256k1 inputs or the 2,313 hard inputs used before. It is set by about 0.4% of Euclid inputs of the form q − 2^j − r with small r. We built 246 point-add lanes from 159 such inputs, and the final circuit passes them all.

## Result

On the board: pending; the bot's comment will give it.

Locally, on the final upload file:
- A bit-sliced simulation on three seeds: 10 boundary cases, 18 adversarial points and 10,000 random points each, then a reverse run back to the input. 0 failures.
- The 2,313 hard inputs and 2,000 uniform draws, with the reverse run: 0 failures.
- The 246 pending-borrow lanes and 500 uniform draws, with the reverse run: 0 failures.
- The tree's toy curves, exhaustive on their promise domains: 0 failures.

The board's own check, `qcheck.py` with `qcb` on Windows, passed 3 random roots (7f0763000e24…, c6da6a9b6f74…, 3e25450bcfec…), each with 0 of 10 boundary cases and 0 of 10,000 seeded inputs failing. It reports lowered_t 20,568,604 and qubit-time 123,628,195,840 = 4,096 wires × 30,182,665 ticks, and says the file meets the bar. Those costs are exactly the local measurement's.

The expanded ops charge is 11,977,551 against the 21,696,718 limit. The upload is 61,788,101 bytes plain, against the 104,857,600 cap.

## Reproduce

With Python 3.14 (the generator is stdlib only):

```sh
git clone https://github.com/2elliot/ecdsafail-v2-reference lowt && cd lowt
git checkout e28dddcdcc9eadb780f824ef4d148dc710e201b1
python3 -I -B scripts/emit.py --hier --out lowt-stack.circuit.json
# c06cdbd7543648a9d810ad8e3162475a5c8d30e231985a4d2355e7010b632d21  lowt-stack.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py lowt-stack.circuit.json lowt-stack.method.json
# 1bbb1ca88de20fc851aa24e31babc286b583750fd1ab3f0a9c18ff39d6aa1dec  lowt-stack.method.json
gzip -9 -n -c lowt-stack.method.json > lowt-stack.method.json.gz
python3 -I -B scripts/breakdown.py
python3 -I -B scripts/accept.py
```

## Dead ends

Joel's PR #32 note lists his. These are ours, measured or proven unless marked as an estimate.

- **No 7-CCX two-bit Euclid cell in place.** The 2+5 and 3+4 splits, with or without one clean borrowed wire, have none by exhaustive search.
- **A merged strip-meeting and window cell.** Its lower bound is 6 CCX against the 8 used, about 2,800 CCX in all, but the 12- and 13-dimension searches didn't finish.
- **The whole high word in 2R + 1.** Cells exist at R = 2 (5 CCX) and R = 3 (7 CCX), but neither has a ladder shape. The R = 3 cell saves only 36 CCX at G and isn't in this entry. A general construction would be worth about 400 to 700 CCX, an estimate.
- **Copied square leaves' rows 0 and 1 at 2 a place.** Y + 2·r₀ isn't linear in Y, and every form found needs three products a place.
- **The sampled inert-CCX leads in the Bézout pass's first cells.** They are padding on sampled inputs, but proving them needs a tick bound below T, which PR #32 lists as closed.
- **Mike's secp256k1 width cut (PR #28).** It measures 0 here, because the high word sizes from the joint bound, not from the tick widths.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

https://claude.ai/code/session_01WBnjwApiKZrZbiFEd1azjq

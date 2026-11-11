---
evaluation: eval-5100e4daff5edc3e
pull: 22
head: 8fd153c376913ff43b3058ed8836381d8c3b96c6
model: Claude Opus 5.5 (design and pricing), Claude Fable 5.2 (implementation, checks and review) and GPT in Codex (the first draft of the Karatsuba square-subtract, on our `v2/lean`)
harness: Claude Code and Codex CLI
built_on: elliot/comp-2elliot-pr12-b2284da7c963
source: 0cb7bc6a3b195866153b2e24a1823b776815f915
---
This entry is the circuit that branch `v2/karatsuba-trees` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`, after PR #12's own post-pass, `qec/gate_pass.py`.
That commit is 2elliot's `depth/v5` at `435d287`, the source of `cold-trees` (PR #12), plus eight commits of ours: six that change the circuit, one that widens a check and one that corrects text.
PR #12 is built on ab-godder's PR #8, which is built on our PR #7. Everything PR #12 and PR #8 made is kept here, and is their work.

**This entry is for the Pareto frontier, not for an award.** It is lower than PR #12 in qubit-time, by 6.37%, and lower than PR #18 in T, by 234,297, so no entry on the board is lower on both costs.
PR #18 holds best qubit-time, 40.27% below this entry's.

| at A = G | reference-c4 | PR #12 (built on) | this entry | change on reference | change on PR #12 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,250 | 1,258 | −45.56% | +8 |
| scheduled ticks | 71,023,638 | 5,349,345 | 4,976,763 | −92.99% | −372,582 (−6.965%) |
| CCX | 7,209,419 | 7,114,217 | 7,128,796 | −80,623 (−1.12%) | +14,579 (+0.205%) |
| T | 50,465,933 | 49,799,519 | 49,901,572 | −564,361 (−1.12%) | +102,053 (+0.205%) |
| qubit-time | 164,135,627,418 | 6,686,681,250 | 6,260,767,854 | −96.19% | −425,913,396 (−6.370%) |
| expanded ops charge | 20,960,620 | 19,109,053 | 18,965,063 | −9.52% | −0.754% |
| upload bytes | 43,423,435 | 79,727,065 | 77,273,181 | +77.95% | −3.078% |

Every figure for PR #12 and this entry is for the post-passed method, the one the board scores.
T is under the reference both before and after the post-pass: the raw emission has 7,130,322 CCX and T 49,912,254, 553,679 under, and the post-pass, 2elliot's, takes it to 49,901,572.

## What changed

Our commits, in order, with their effect at G after the post-pass:

| # | commit | change | wires | scheduled ticks | CCX | T |
|---:|---|---|---:|---:|---:|---:|
| | `435d287` | PR #12 | 1,250 | 5,349,345 | 7,114,217 | 49,799,519 |
| 1 | `da98fc5` | the tick's probe as a carry-out tree | 1,250 | 5,246,467 | 7,144,857 | 50,013,999 |
| 2 | `e5eeec4` | the Euclid difference's ring capped at 217 | 1,250 | 5,244,059 | 7,144,857 | 50,013,999 |
| 3 | `3634b0d` | PR #12's add at 63 and 62 blocks of 3, on 14 temps | 1,256 | 5,055,456 | 7,209,201 | 50,464,407 |
| 4 | `b1d0e53` | the post-pass check's control searches 64 candidates | 1,256 | 5,055,456 | 7,209,201 | 50,464,407 |
| 5 | `d788aae` | our Karatsuba square-subtract, on PR #12's tree adders | 1,256 | 5,112,574 | 7,080,538 | 49,563,766 |
| 6 | `32c18e8` | PR #12's add at 84 and 84 blocks of 3, the layout's cap | 1,256 | 5,000,829 | 7,128,796 | 49,901,572 |
| 7 | `6a13bcb` | a corrected check description and text | 1,256 | 5,000,829 | 7,128,796 | 49,901,572 |
| 8 | `0cb7bc6` | PR #12's add with 8 copies of J | 1,258 | 4,976,763 | 7,128,796 | 49,901,572 |

1. **The tick's probe as a carry-out tree** (`_probe_tree` in `qec/two_bit.py`). Ours.
   It is the same change as commit `976a302` on our public `v2/lookahead` branch, ported onto PR #12's tick.
   The tick's first chain only computes e = P.top · (the carry out of L plus the selected constant), and it rippled 15 `_selected_carry` steps up and down.
   For each bit the tree puts the XOR of the active selectors on a temp k, forms g = L·k and sets L ^= k, so L holds p, and g and p are never both 1.
   The bits below the quiet run merge in a balanced tree; M's top carries the result across the quiet run, the top bits ripple, e takes its CCX, and the recorded gates run backwards.
2. **The Euclid difference's ring capped at 217** (`U6_RING` in `qec/point_add.py`, from 193).
   `U6_RING` is ours, from `d9ecbbe` in PR #7's line. `WIDE_RING` and `RING_BORROW`, which decide when a step takes a wider ring, are 2elliot's.
   CCX, wires and call sites are unchanged, and the post-pass ticks fall by 2,408. 217 is the best of the caps we ran, from 193 to 289, and not a proven optimum.
3. **PR #12's add at 63 and 62 blocks of 3, on 14 temps** (`LA_F_WIDE`, `LA_B_WIDE` and `LA_NTEMPS` in `qec/two_bit.py`).
   The add, its blocks of 3, `_la_wide` and the form of the counts and temps are 2elliot's. We changed their values, from 34 and 35 blocks of 3 and 8 temps.
   `_la_wide` saves a chain step only when a count goes from even to odd, at 3 CCX a call a step.
4. **The post-pass check's control searches 64 candidates** (`TRIES` in `scripts/post_pass_check.py`, from 20).
   Commit 3's temps reordered the control's candidates, and on one instance all of the first 20 were harmless. The floor stays at 16 instances, and the search was widened instead.
5. **Our Karatsuba square-subtract, on PR #12's tree adders** (`KaratsubaMulsub` in `qec/square.py`).
   The class is our one-level Karatsuba square-subtract from our `v2-lean` entry (PR #11), whose starting point we credit to ab-godder and 2elliot.
   It splits Y = L + βH with β = 2^h, h = n/2 and β² = 2^n = f mod q. Group AC builds A = L² and C = (L + H)², and group B builds B = H². Each group is a recorded block whose kept wires form its own junk register.
   One controlled modadd subtracts each group's residue from X, and the block's inverse then restores Y and clears the junk.
   The half-square offsets collect into one constant, which joins the kernel's constant add of 3a under gen.
   Building, applying and unbuilding half squares into X with β² = f mod q follows arXiv:2609.09582, section 5.3.4, and the Karatsuba modular square is arXiv:2609.05625, section VII.2.
   On this branch the class's adds run on 2elliot's adders, each with the same carry in and carry out as the add it replaces: `add_lookahead` trees for the rows and the folds, and 2elliot's tree modadd.
   Taken as it was, the class would peak at 1,315 wires here. At 44 temps and 7 copies the AC modadd brings it back to 1,256.
   This commit saves 128,663 CCX for 57,118 ticks.
6. **PR #12's add at 84 and 84 blocks of 3, the layout's cap** (`LA_F_WIDE` and `LA_B_WIDE`).
   Commit 5's CCX pay for the counts' cap at n = 256: each sweep is 84 blocks of 3 and 3 ripple steps, 87 chain steps against 97 and 98. That is 21 steps, at 2,298 CCX each, 48,258 CCX, for 111,745 ticks. The temps stay 14.
7. **Text.** INVARIANTS.md now says what `probe_check.py` requires of a borrowed wire: it reads 0 right after its X wherever AND(H) = 1.
8. **PR #12's add with 8 copies of J** (`LA_NCOPIES` in `qec/two_bit.py`, from 6).
   The copies of J and their rotation are 2elliot's, and we changed the count. Two more wires take 24,066 ticks off at the same CCX, which is −0.323% in qubit-time. Ten copies give 4,970,532 ticks at 1,260 wires, a higher qubit-time than 8. We have not shown why more copies shorten the path.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase. Exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in the base's `DESIGN.md`.

- **Probe tree.** The tree computes the same carry out as the ripple. Its temps come from the same `take()` the ripple used, which PR #12 made idle-first. That `take()` borrows one of H's wires with an X and returns it with an X. This is sound for any H wire, because e can be 1 only where every H wire reads 1, so a borrowed wire is clean exactly where the result matters. P's and M's tops come from the allocator's clean wires, never from H.
- **Ring cap.** The step's map is the same at every ring size; the ring only holds the sweep's temps and control copies, and every one is cleared before the ring is handed back.
- **Block counts.** `_la_wide` makes only layouts that PR #12's add already supports. PR #12's `add_or_shift_check.py` runs the add with every composition of each sweep into blocks of 1 to 4 bits, on every input, at small widths.
- **Karatsuba square-subtract.** Each add it calls has the carry in and carry out of the add it replaces, each group's block is applied and then inverted with nothing in between changing a wire it read, and the identity is the one above.

### The post-pass

The board scores the method after `qec/gate_pass.py`, so we checked the pass on our code as well as the raw circuit.

- **The tree.** At commit 1, at G, the pass cancels 21,344 pairs. 296 of them include a gate emitted while the tree runs: 8 pair two of the tree's own gates, and the other 288 are X gates of PR #12's `take()` and `drop()`.
  `I2.post-pass` builds 40 small instances (the 36 toy instances and four 14- to 17-bit primes on the tick's merged path, 514,700 lanes) and checks each exhaustively, as written and after the pass. Its control makes one wrong cancel in our gates at a time. At commit 1 it is caught on all 22 instances that run the tick, and on this head on 21 of them, against a floor of 16.
- **The ring cap.** `I2.ring` checks the Euclid step at every ring size from w − 2 to 2w at widths 9 to 12, as built and after the pass.
- **The add.** At 256 bits the raised counts change the layout from bit 3 up, and below width 108 the layouts equal PR #12's. So `I2.add256` checks the add alone at 256 bits with this head's layout and temps. It runs every carry run from each bit i to each j at or above it, plus all-zero and all-one words and random lanes, on both J, against integer arithmetic, as built and after the pass.
- **The square-subtract.** No toy row reaches the class, so `I2.karatsuba` runs every Y and every X at each canonical tail at q = 65,521, 262,121 and 262,139, its pieces at full width, twelve controls and 200 mutants. `I2.karatsuba-post-pass` checks the class as built, as `qec/emit.py` writes it and after `qec/gate_pass.py`, at those primes and at both full widths, with a wrong cancel that must be caught.
- At 256 bits the pass's cancellations across these blocks' boundaries are reached only by `qcb` and the fresh roots below.

### Checks on the head

- `scripts/accept.py` reports 24 pass, 0 fail and 1 pending (`I7.charge`) at seeds 2041 and 7331 on `0cb7bc6`. Its mutation check catches 189 of 200 CCX-to-CX mutants, against a floor of 163.
- Each circuit commit's tests fail on its parent and pass on its own.
- Two limits are named in the tree's README and INVARIANTS.md: `I2.model`'s zero-caller row is narrower than the reference's gate, and the standalone multiplier's contract is narrowed to x in [1, q), with x nonzero at its entries resting on the kernel's flag code.

## Result

On the board: pending; the bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` passed on the post-passed method at the challenge root of published round 5 and at two fixed roots, and the guide's `qcheck.py` passed at 106 random roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 1,091,090 inputs in all.
The cost is T 49,901,572 and qubit-time 6,260,767,854, which is 1,258 wires × 4,976,763 ticks.
Every upload limit holds: 1,258 of 4,096 wires, call depth 3 of 32, 77,273,181 bytes, and an expanded ops charge of 18,965,063 of 21,696,718.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

The emit needs Python 3.12, as PR #12's does:

```sh
git clone --branch v2/karatsuba-trees https://github.com/iwnlcern/ecdsafail-v2-reference karatsuba-trees
cd karatsuba-trees
git checkout 0cb7bc6a3b195866153b2e24a1823b776815f915
python3.12 -I -B scripts/emit.py --hier --out karatsuba-trees.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py karatsuba-trees.circuit.json karatsuba-trees.raw.method.json
python3 -I qec/gate_pass.py karatsuba-trees.raw.method.json karatsuba-trees.method.json
gzip -9 -n -c karatsuba-trees.method.json > karatsuba-trees.method.json.gz
shasum -a 256 karatsuba-trees.circuit.json karatsuba-trees.method.json karatsuba-trees.method.json.gz
```

```text
f07a2e0b7563a2b7c0561ddb54de4bb402be7879760c894456e36c1bfa762add  karatsuba-trees.circuit.json
1009a280394132ab9c4cc9ed7c0be5d3b412a1c1bbcdcf2394d93ac26b92dd84  karatsuba-trees.method.json
d8fd2bdf5afe44b3d6929406bbe8ad0a8b7a87a7c618327b06c1d35d24e9aa2a  karatsuba-trees.method.json.gz
```

The gzip hash is from macOS gzip and may differ on other builds.

## Dead ends

All numbers are measured, after the post-pass.

- **Our own lookahead add in the tick**, in 10 forms, priced against the tree alone, before commits 2 to 6. None beats the tree alone. The best, all blocks of 3 on 19 temps, is 16,668 ticks faster, but its 5 more wires make qubit-time 0.081% higher, and it is 271,936 T over the reference. As we read it, PR #12's add already has blocks of 3, temps in turn and a cheaper backward step.
- **Rotating the Euclid encodes over more ancillas.** PR #12's encodes are off the critical path: 4, 6 or 8 sets give the same ticks as its 3. Our pool saves 2,089 ticks, but for 5 or 11 wires, +0.361% or +0.841% in qubit-time.
- **A 257-wire ring**, our earlier cap: +392 ticks on PR #12. As we read the code, PR #12's wide ring and borrow already give the widest steps the bit-level tree.
- **9 temps at the raised counts of commit 3.** 1,251 × 5,086,325 (−4.841% on PR #12) at F 63 and B 61. 13 to 16 temps all beat it, and 14 beats it at every split we ran; 17 and 18 lose to 14 on qubit-time.
- **The Karatsuba class with its own ripple adds**, as on our lean branch. Taken onto commit 3 that way, it saves 426,731 CCX but costs 689,003 ticks (+13.6%), because PR #12's square runs trees where those adds ripple.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---
evaluation: eval-18e30ae6181f7877
pull: 31
head: 8008ba6fcd11732f8195716e4dde9863fc10bbc4
model: Claude Opus 5.5 (design, with parallel design subagents, implementation and checks)
harness: Claude Code
built_on: mike/comp-ab-godder-pr28-9ffef68b2198
source: 96255f2
---
This entry is the circuit that branch `bal-24` of ab-godder/ecdsafail-v2-reference emits at the commit in `source`, in the form the board scores, after the post-passes in `scripts/post_pass.py`.
That commit is our PR #28's source, `lowt-depth` at `5ca6779`, plus two lines of our commits:
- `bal-wires` gives back PR #28's wire spends, which leaves the multiplier's own peak of 1,260 wires. Its source is also entered alone.
- `bal-tick` spends CCX on the Bézout tick's depth.

It is a point between the board's low-T entries and its low-qubit-time ones.
Two of its ideas are iwnlcern's, each reimplemented here from its description for our tick and square:
- the split coefficient add and the cap on the tick's prefix blocks, from PR #24 (`v2/split-cap`, `dab249c` and `6b5d78b`);
- the one-level Karatsuba square-subtract, from PR #11.

| at A = G | reference-c4 | PR #28 (built on) | PR #24 | this entry | change on reference | change on PR #24 |
|---|---:|---:|---:|---:|---:|---:|
| wires | 2,311 | 4,096 | 1,249 | 1,263 | −45.35% | +14 (+1.12%) |
| scheduled ticks | 71,023,638 | 18,475,761 | 17,016,612 | 15,935,010 | −77.56% | −1,081,602 (−6.36%) |
| CCX | 7,209,419 | 3,185,039 | 3,472,890 | 3,446,208 | −52.20% | −26,682 (−0.77%) |
| T | 50,465,933 | 22,295,273 | 24,310,230 | 24,123,456 | −52.20% | −186,774 (−0.77%) |
| qubit-time | 164,135,627,418 | 75,676,717,056 | 21,253,748,388 | 20,125,917,630 | −87.74% | −1,127,830,758 (−5.31%) |
| expanded ops charge | 20,960,620 | 13,411,577 | 13,766,823 | 13,369,496 | −36.22% | −397,327 (−2.89%) |
| upload bytes | 43,423,435 | 39,166,573 | 38,797,998 | 42,053,772 | −3.15% | +3,255,774 (+8.39%) |

This entry is lower than PR #24 in both T and qubit-time.
Against our low-wire build alone (1,260 wires × 18,673,926 ticks, T 22,750,784), it spends 196,096 CCX and 3 wires for 2,738,916 fewer ticks.

## What changed

`DESIGN.md` gives each change in its own section: "A low-wire build", "The split add-or-shift" and "A cap on the tick's high Prefix".

### The low-wire build (`qec/point_add.py`, `qec/square.py`)

1. **No held offsets** (`DialogMultiplier.HELD = False`). +3,056 CCX, −384 wires.
   - The Euclid step runs on its two-wire cell with the offset sweep at the end of the pass.
   - The strip and the clear of `two` go before the subtract's cswap(g0, E, b). `two` implies NOT g0, so both orders read the same E and b.
   - The step's control copies sit on the pass's unwritten tape wires. At G the ticks are the same as with held.
2. **No kept carries, and a coded tape** (`KEEP = 0`, `RAW_TAPE = False`). −55 wires off the Bézout peak, +13,768 CCX.
3. **Half squares taken mod q** (`HalfSquares`). With β = 2^(n/2), Y = L + βH and S = L + H, Y² ≡ L² + β(S² − L²) − (β − f)H² (mod q), in two groups.
   - Each group is a recorded block that the gen-controlled modadd takes into X, and then its inverse clears it.
   - The groups' offset K joins the kernel's 3a add.
   - The square peaks at 1,241 wires.

### The tick (`qec/two_bit.py`, `_bezout_tick` and `_bezout_pass` in `qec/point_add.py`)

4. **The split add-or-shift** (`SPLIT_ADD`, `_add_then_rotate`). +196,096 CCX, no wires, −2.25M ticks at G, 11.49 ticks a CCX.
   - The old add-or-shift gave each bit one CCX for both the add's sum and J's shift, through a rolling XOR D that every bit of the down pass reads and writes. So the down pass ran at about 23 ticks a bit.
   - The split form keeps the two maps apart.
     - **The add:** a ripple add of a >> 1 with each sum bit made conditional on NOT J in its up pass.
     - **The rotation:** on J, a left rotation of the word as two layers of disjoint controlled swaps. Each swap goes out as soon as the down pass has finished both its bits.
   - The add acts only off J and the rotation only on J, so on every lane one of them is the identity.
   - **Gate order, which is new here.** In the up pass each carry line's c ^= a goes before the CCX that sets the carry, so the carry is only ever a CCX control, at 7 ticks a bit. In the down pass the restoring c ^= a goes after the next lower CCX, so the line is only ever a CCX target, at 11 ticks a bit.
   - The standalone add drops from 8,478 to 4,673 ticks each way. It is 4n + 1 CCX a tick against 3n + 2.
5. **A cap on the tick's high Prefix** (`TwoBitMod.P_CAP = 37`). +3 wires, −493,355 emitted ticks.
   - The tick's high-word prefix P uses blocks of at most 37 bits instead of doubling blocks.
   - Caps 31, 44 and 48 gave 1,264, 1,262 and 1,262 wires at higher qubit-time.

Kept from PR #28:
- the handed-over held bits;
- the shared bottom carry;
- the B1 sweep;
- the secp256k1 width cut at Euclid ticks 129 to 168;
- every depth change, through the post-emit passes.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase. Exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in the base's `DESIGN.md`.

- **Items 1 and 2** restore constructions the tree already had. The reorder in item 1 is the identity on every state.
- **Item 3** is the identity above. `HalfSquares.bounds` checks every inequality its folds need, on the integers.
- **Item 4:**
  - Right after bit i's carry CCX, with x = c_i ⊕ a_i on the carry line, u ^= NOT(J)·NOT(x). Since x·NOT(x) = 0, the down pass reads the same x·u, so it is the plain undo.
  - Each moved c ^= a shares only a target wire with the CCX it passes, so the two commute.
  - On J, the word's top two bits are 0, because a J cell never follows a B1. So doubling the word is a rotation.
- **Item 5** only changes the block sizes of an exact prefix.

Checks run on `source`:

- `scripts/accept.py`: gate PASS, 19 pass, 0 fail, 1 pending (`I7.charge`). This includes the new `I2.add` row.
- `scripts/check.py --toy-bits 4 6 7 8 9 10 --toy-per-size 6`: 36 toy curves exhaustive, 0 failures. The toy curves run the split add. `scripts/check_passes.py` checks the post-passed toys: 0 failures.
- `scripts/add_check.py` (new) runs every variant of the split add at n = 3 to 7 on every (J, a, word):
  - with held lanes on and off;
  - with 1, 2, 4 and 7 copies of J.

  50 cases, 0 failures, and its controls are caught 48 of 48.
- `scripts/prefix_check.py`: 122 cases, 0 mismatched, controls caught 248 of 248. It runs caps 1, 2, 3 and 5 exhaustively, and the tick's lengths at caps 23 and 37.
- `scripts/halves_check.py`: 158 rows, 0 failures, controls caught 18 of 19. The 19th is an equivalent circuit on its whole domain.
- `scripts/step_check.py`: 1,068 variants, 0 mismatched, controls caught 828 of 828.
- `scripts/keep_check.py`: 108 primes, 0 bad lanes. `scripts/model_check.py`: all rows ok. `scripts/check_hier.py`: hier == flat.
- `scripts/check_file.py` on the submitted circuit at G: 252 lanes, 0 failures.

## Result

T 24,123,456 and qubit-time 20,125,917,630: 1,263 wires over 15,935,010 ticks.
`qcb diagnostic check` reports 0 failures in 10 boundary and 10,000 random cases on each of the five published challenge roots and on a random root.
Expanded ops charge 13,369,496 against the 21,696,718 limit.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

With Python 3.12:

```sh
git clone --branch bal-24 https://github.com/ab-godder/ecdsafail-v2-reference lowwire-split
cd lowwire-split
git checkout 96255f2
python3.12 -I -B scripts/emit.py --hier --out lowwire-split.emit.json
python3.12 -I -B scripts/post_pass.py lowwire-split.emit.json lowwire-split.circuit.json --jobs 8 --rounds 2
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py lowwire-split.circuit.json lowwire-split.method.json
shasum -a 256 lowwire-split.emit.json lowwire-split.circuit.json lowwire-split.method.json
```

```text
73ca4888cbc583901d9bd2aab55d39608ed44eaf16548ca4aff9366b02ca9dd8  lowwire-split.emit.json
670e5099256d42b463532fbd972324cac91c24bab7d327c6356a388c145630e9  lowwire-split.circuit.json
17424ea61d7b15539622dd4348df57464ba51dbccd8b0e0de16b0770d522d4bd  lowwire-split.method.json
```

## Dead ends

- **PR #25's unwind and PR #26's carry-select compare.** They buy about 3.6 and 3.7 ticks a CCX, against the split add's 11.5. The split add already uses most of the T room under PR #24.
- **Our own qubit-time entry's tick pieces** (PR #29). Its tree backward sweep buys 1.44 ticks a CCX, and its pebbled prefix write takes 39 fresh wires. Neither fits a 1,260-wire base.
- **The compare in the split add's carry order, with a depth-pass tie-break.** It gave −471,000 emitted ticks, but after the post-passes it was 3,862 ticks worse than the version without it, because `reorder_pass` already finds those orders.
- **A split add with no extra CCX**, folding the shift back into one CCX a bit with blocked D wires. Each carry line holds its value only until the next lower carry CCX overwrites it, so every variant we worked through serializes again or needs a growing buffer of wires.
- **`SPLIT_SKIP`.** It keeps the one-pass add on chosen ticks and gives back 510 CCX a skipped tick for about 5,700 ticks, linearly. It is 0 here.
- **The tick's floor.** The add, the compare and the Euclid steps now each run at about 18 ticks a bit per up-and-down pass. Below that, each needs lookahead CCX or wires.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

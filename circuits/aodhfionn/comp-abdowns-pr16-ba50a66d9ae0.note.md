---
evaluation: eval-9dbc38b0f2f7af96
pull: 16
head: ba50a66d9ae061bd0779814211a4ea7bb040e2db
model: Claude Fable 5.2
harness: Claude Code
built_on: jack/comp-iwnlcern-pr10-0e70e4252785
source: ba50a66d9ae061bd0779814211a4ea7bb040e2db
---
This entry puts PR #14's circuit on `main` and brings the generator that emits it into `qec/`.
PR #14 is on `best-t` as `aodhfionn/comp-abdowns-pr14-87074c1cc523`, and this upload is byte-identical to it.
The commit in `source` is upstream `main` at `1430a30` with two changes: `qec/` taken from branch `lowt-junk` of abdowns/ecdsafail-v2-reference at `171738f` (PR #14's source), and `circuits/abdowns/lowt-junk.method.json.gz`.
`scripts/emit.py` is the same on `main` and at `171738f`, so the commit in `source` emits the circuit by itself.
`171738f` is iwnlcern's `v2/lowt-c1` at `a4d4c28`, the source of `lowt-c1` (PR #10), plus two commits of ours.
It aims at T only.
PR #10 peaks at 3,135 wires, in the square, and its multiplier runs at 1,312.
The cap is 4,096, so this entry fills the wires up to the cap with carries that would otherwise be cleared and rebuilt.
The point-addition sequence, the Euclid step's logic, the Bézout tick and the Karatsuba square are PR #10's and PR #9's.

| at A = G | reference-c4 | PR #10 (built on) | this entry | change on reference | change on PR #10 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 3,135 | 4,096 | +77.24% | +961 |
| scheduled ticks | 71,023,638 | 36,228,738 | 35,763,926 | −49.65% | −464,812 (−1.28%) |
| CCX | 7,209,419 | 3,206,027 | 3,192,795 | −55.71% | −13,232 (−0.41%) |
| T | 50,465,933 | 22,442,189 | 22,349,565 | −55.71% | −92,624 (−0.41%) |
| qubit-time | 164,135,627,418 | 113,577,093,630 | 146,489,040,896 | −10.75% | +28.98% |
| expanded ops charge | 20,960,620 | 12,679,056 | 12,727,517 | −39.28% | +48,461 |
| upload bytes | 43,423,435 | 35,112,462 | 35,606,310 | −18.00% | +493,848 |

Qubit-time rises because the wire count does.
It stays 17,646,586,522 below the reference's, so the entry is lower than the reference on both costs.

## What changed

Two commits on `a4d4c28`: `c603823` changes the generator and adds a check, and `171738f` adds a `DESIGN.md` section, README figures and the baseline pin.
Entries may change only `circuits/` and `qec/`, so this pull request carries `qec/` as it is at `171738f`, nine files against `main`.
The check `scripts/junk_check.py`, the proofs under `docs/`, the receipts and the `DESIGN.md` section stay on `lowt-junk` at `171738f`.

**The rule both changes use.** A block that is applied forward and later inverted, with nothing in between changing a wire the block read, is a compute and uncompute pair.
Inside such a block, a temporary that the block clears with a CCX can stay set on a wire of its own.
The forward block loses the CCX that cleared it, and the inverse loses the CCX that rebuilt it.
The price is one wire, held from the forward block to its inverse.
PR #9's note names both uses below as unbuilt or switched off.

1. **Euclid carries kept across the Bézout pass.** `_shared_difference`, `_euclid_step`, `step_carries`, `junk_capacity`, `_euclid_pass` and `_body` in `qec/point_add.py`. −11,136 CCX on 2,784 wires.
   - M is the forward Euclid pass, the Bézout pass, then the inverse of the same recorded pass.
   - A step's shared difference builds its carries in place on U, then spends one CCX a bit restoring a_i.
     With `junk` wires, the carry out of each bit 1..k goes onto a junk wire as c ⊕ (a ⊕ c)(b ⊕ c), with a_i and b_i put back by CX.
     Those bits need no restore, so they cost 3 CCX, not 4.
   - Bits above k run the in-place chain as before. Their carry in is the last junk wire, which their unwind restores.
     `_top_bit` and the top `g1` gate read the carry from whichever wire holds it.
   - The step's second-strip flag `two` can also stay on a junk wire, which drops the CCX that cleared it.
   - `_euclid_pass` hands junk wires to its steps in order, each step's carries first and then its flag.
     It skips the first step, which may be `_euclid_tick0`, and the last.
     At secp256k1 the first ten steps after tick 0 keep every carry and their flag, and the eleventh keeps 234 of its 254 carries.
   - `_body` allocates the junk before the forward pass and frees it after the inverse pass.
     `DialogMultiplier` takes the budget as a constructor argument, 0 by default, so every existing caller emits as before.
   - A wire saves one CCX in the pass and one in its inverse, and M runs twice, so each wire is worth 4 CCX.
2. **Square row carries.** `_keep_add`, `_rows`, `_Lend` and `SquareFold` in `qec/square.py`. −2,096 CCX on 1,048 wires.
   - ab-godder's `SquareFold.KEEP` already keeps the leaf rows' carries and was left at 0. It is now a constructor argument.
   - A row that the remaining junk cannot cover keeps the carries it can and finishes as a plain add from the last kept carry. Before, such a row kept none.
   - `_Lend` lends the fold's wires E, T, t, c and k, which are clean until the fold, to the allocator during the Karatsuba recursion and the first `_multiple`.
     The Karatsuba adds take their pads and constant-add carries from those wires.
     Compute's own scratch falls from 128 wires to 32, under the modadd's 41, which frees 87 more wires for rows.
   - Compute runs once and its inverse once, so each wire is worth 2 CCX.
3. **The budget.** `ControlledPointAdd._junk_budgets` and `WIRES_PER_BIT` in `qec/point_add.py`.
   Each stage is built alone with no junk to find its peak. The room left under 16n wires, 4,096 at n = 256, is that stage's budget.
   At secp256k1 a Euclid pass could use 65,279 junk wires, so the cap binds, not the idea.
4. **A check.** `scripts/junk_check.py`, described below.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase, and exactness on basis states with clean scratch is the whole contract, as in the base's `DESIGN.md`.

- **The junk is a function of the block's input.** A kept carry is MAJ of bits the step read, and `two` is a function of the step's g0 and V.
- **Nothing between the pass and its inverse changes what the pass read.** The Bézout pass borrows U's and V's zero wires and returns them at zero, and it restores every tape cell. PR #10 relies on the same fact when it replays the recorded pass in reverse.
  The junk wires are in the pass's register layout, and no other block is given them.
  So the inverse pass meets each junk wire holding the value its reversed gates expect, and returns it to 0.
- **The kept bits compute the same outputs.** On a kept bit the select is B ⊕= g0·(a ⊕ c) and the swap select is A ⊕= g1·(B ⊕ c ⊕ 1), on the same a, b and c as before. Only where c sits and whether a was overwritten differ.
- **The square.** `_keep_add` is ab-godder's. The partial row is its first k bits followed by `add` with a carry-in wire, which `add` restores.
  `_Lend` only changes which clean wires an add pads with. It takes back exactly the wires it lent, and every allocation in between is freed clean.

Checks on `source`:

- `scripts/junk_check.py 600 16384` exits 0.
  - M on every x in [1, q) with a random y, forward and inverse, at all 107 odd primes from 5 to 599, for up to eleven junk budgets each: none, 1, 2, 3, 5, 8, a third, half, one short of full, full, and over-full. 1,162 rows, 0 failures.
  - In every row the CCX saved is exactly 2 per junk wire used.
  - The square's compute on every n-bit Y, and its inverse back to clean junk, at all 50 primes below 16,384 where the fold applies, for up to eight keep budgets. 367 rows, 0 failures, and exactly 1 CCX saved per kept wire.
  - Control: 60 of 60 CCX-to-CX mutants of M with full junk are seen.
- `scripts/accept.py`, full mode: 13 pass, 0 fail, 1 pending (`I7.charge`), the same as the base.
  It includes 36 toy instances exhaustively, flat and hierarchical (25,800 lanes), 52 full-width components, the hierarchical upload at 24 addends including G, and 191 of 200 mutants caught.
- Before the budget became automatic, the whole point addition was checked exhaustively on 48 toy instances from 4 to 11 bits (59,160 lanes) at each of eleven junk budgets from 0 to 1,000. 0 failures, and 4 CCX saved per wire.
- All 266 definitions are identical over 25 addends (`I5`), since the budgets depend on q alone.

## Result

On the board: the same circuit passed as PR #14, the lowest-T entry at layer 1, with T 22,349,565 and qubit-time 146,489,040,896.
This upload is pending; the bot's comment will give it.

Before this pull request, the method emitted from the commit in `source` passed `qcheck.py` on 3 new random roots, 0 of 10 boundary and 0 of 10,000 seeded inputs failing on each.

For PR #14, `qcb` 8376098 `diagnostic check` ran through the guide's `qcheck.py` on 43 random challenge roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 430,430 inputs in all.
The cost is T 22,349,565 and qubit-time 146,489,040,896, which is 4,096 wires × 35,763,926 ticks.
Every upload limit holds: 4,096 of 4,096 wires, call depth 3 of 32, 35,606,310 bytes, and an expanded ops charge of 12,727,517 of 21,696,718.
The same tools reproduce PR #10's published circuit and method hashes byte for byte from `a4d4c28`.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

By change, built in order:

| change | CCX | saved | wires |
|---|---:|---:|---:|
| PR #10 | 3,206,027 | | 3,135 |
| square row carries at the old scratch, 961 wires | 3,204,105 | 1,922 | 4,096 |
| Euclid junk, 2,784 wires a pass | 3,192,969 | 11,136 | 4,096 |
| lent pads and partial rows, 87 more row wires | 3,192,795 | 174 | 4,096 |

## Reproduce

With Python 3.12 or 3.14, which emit the same bytes:

```sh
git clone --branch entry/lowt-junk-main https://github.com/abdowns/ecdsafail-v2-reference lowt-junk
cd lowt-junk
git checkout ba50a66d9ae061bd0779814211a4ea7bb040e2db
python3 -I -B scripts/emit.py --hier --out lowt-junk.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py lowt-junk.circuit.json lowt-junk.method.json
gzip -9 -n -c lowt-junk.method.json > lowt-junk.method.json.gz
shasum -a 256 lowt-junk.circuit.json lowt-junk.method.json lowt-junk.method.json.gz circuits/abdowns/lowt-junk.method.json.gz
```

The junk check lives on `lowt-junk`:

```sh
git fetch origin lowt-junk && git checkout 171738fdc528276022da469151048843141f07a4
python3 -I -B scripts/junk_check.py 600 16384
```

```text
0ffeb104bfd7e9f9830bcfe14186aa68697c1f6a145e8f1e7ac78cdc198c3548  lowt-junk.circuit.json
29ac0b13e88f162059e5e441f99cb5056caf1521204f11d5c616bed8b6e449fd  lowt-junk.method.json
d71f1279a9d402c2607d22bf82f6143774c59e38d0fa2d54a12748fce10a6224  lowt-junk.method.json.gz
d71f1279a9d402c2607d22bf82f6143774c59e38d0fa2d54a12748fce10a6224  circuits/abdowns/lowt-junk.method.json.gz
```

The gzip hash is from macOS gzip and may differ on other builds.

## Dead ends

Measured numbers are marked. The rest is argument, and says so.

- **A different Karatsuba depth, with the wires going to rows instead.** Measured on PR #10's tree by leaf threshold.
  Thresholds 16 to 28 need 4,554 to 5,904 wires, over the cap.
  Threshold 32 costs +1,012 CCX at 3,891 wires.
  Threshold 33 saves 32 CCX for 216 wires, where rows give 432.
  Threshold 64, two levels, costs +4,660 CCX and frees 495 wires, which rows turn into 990.
  So the base's 42 stays.
- **Flag first or carries first.** Measured on the 48 toy instances: the totals are the same at every budget, since each wire saves one CCX either way.
- **Coding the tape again to free wires for junk.** By count from PR #2's code: a seven-tick block returns one wire for 30 CCX each way, four times per M. A junk wire is worth 4 CCX in all.
- **Junk in the Bézout pass.** Argument only. The tick's compare and prefix are compute and uncompute pairs, but they contain no in-place restore to drop. Keeping the quotient bit k1 instead of comparing would save 511 CCX a tick, but only the inverse of the Bézout pass could clear it.
- **Junk on a second wire per Euclid bit.** Argument only. With the carry kept, a bit costs one CCX for the carry, one for the select and one for the swap select, and the last two write outputs, not temporaries.
- **Not built.** The square's modadd holds 41 scratch wires at the square's peak, and the Bézout tick 29 at M's peak. Each wire trimmed there is worth 2 or 4 CCX. `_euclid_tick0`'s constant add could keep its carries at the same 4 CCX a wire, which changes nothing while the cap binds.

🤖 Generated with [Claude Code](https://claude.com/claude-code)


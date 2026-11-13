---
evaluation: eval-b72fbc1b8f653457
pull: 37
head: a667d3704df87dd9322943e6ac6080d01dd1365a
model: Claude Fable 5.2
harness: Claude Code
built_on: elliot/comp-2elliot-pr35-de951de1b2ed
source: a667d3704df87dd9322943e6ac6080d01dd1365a
---
This entry aims at T only.
It is PR #35's circuit (`lowt-stack`, 2elliot) with a different rank-one start in the Bézout pass.

PR #35, like PR #32 under it, starts each Bézout pass with fourteen cells on unreduced integers and then pays for a conversion that brings both registers back under q. PR #35 trimmed that conversion by 4,172 CCX.
Here the conversion is another construction. It does no arithmetic modulo a cofactor. It clears the fold's two quotients by walking the cells back on small integers, two digits a cell, and it reads the cofactors from copies the Euclid pass leaves behind.
The cheaper conversion moves the best start from fourteen cells to twenty-seven.
The Euclid passes and the λ² square are PR #35's code, unchanged. Three settings are retuned for the new start.

The rank-one work was built on PR #32's tree before PR #35 opened, and it was never entered. PR #35's branch was then merged into it. So this note describes that work in full, and it says which of its numbers were measured on PR #32's tree.

The commit in `source` is upstream `best-t` at `a6cdcdd` with two changes: `qec/` taken from branch `lowt/descent-stack` of abdowns/ecdsafail-v2-reference at `c1693ce`, and `circuits/abdowns/descent-stack.method.json.gz`.
That branch merges 2elliot's `lowt/v2` at `e28dddc`, which is PR #35's source, into five commits of ours on jtaroreh/lowt-climb at `0f5514b`. Four commits follow the merge: the retune, the removal of six checks of PR #35's rank-one helpers, the sweep check without swap wires, and the pinned flat baseline. It holds the scripts, the checks and the design notes.

| at A = G | reference-c4 | PR #35 (built on, lowest T) | this entry | change on reference | change on PR #35 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 4,096 | 4,096 | +1,785 (+77.24%) | 0 |
| scheduled ticks | | 30,182,665 | 30,123,239 | | -59,426 (-0.20%) |
| CCX | 7,209,419 | 2,938,372 | 2,926,752 | -4,282,667 (-59.40%) | -11,620 (-0.40%) |
| T | 50,465,933 | 20,568,604 | 20,487,264 | -29,978,669 (-59.40%) | -81,340 (-0.40%) |
| qubit-time | 164,135,627,418 | 123,628,195,840 | 123,384,786,944 | -40,750,840,474 (-24.83%) | -243,408,896 (-0.20%) |

Every figure was measured with the full build at 4,096 wires.

## What changed

All of this section is against PR #35.

| file | in this entry |
|---|---|
| `qec/rank_one.py` | Ours, byte for byte the file of our branch before the merge. PR #35's seven rank-one commits are not in it. |
| `qec/point_add.py` | Both sides. PR #35's Euclid pass is unchanged: the two-bit junk cell, the cleared lag strips, the borrowed lazy wires, the direct top borrow, the lazy window cell, the strip-bit junk and the short counter step. Ours adds the kept copies, the cap on the start, the fold's row order and the inline sweep. The two sides met in one hunk, a setting, and in one local name. |
| `qec/two_bit.py` | Ours. The Bézout tick carries the sweep's toggle. PR #35 does not change this file. |
| `qec/square.py` | PR #35's, unchanged. |
| every other file under `qec/` | The same in both. |

Of PR #35's seven rank-one commits, three edit `residue`, `_mod_double` and `_mod_add`, which the digit descent deletes. They were worth 3,516 CCX there. The other four were worth 656 together and are not ported. They are listed under the dead ends as open.

The settings that differ from PR #35's:

| setting | PR #35 | this entry | why |
|---|---:|---:|---|
| `RANK_J` | 14 | 27 | the cheaper conversion |
| `LAZY_END` | 8 | 10 | the start must stay above the last lazy Euclid tick |
| `INLINE_SWEEP` | absent | on | the B1 sweep rides the Bézout ticks |
| `CARRY_JUNK` | 1,273 | 779 | the largest count that builds 4,096 wires |

Where the CCX sit, from `scripts/breakdown.py` on both trees:

| part | PR #35 | this entry | runs | change in total |
|---|---:|---:|---:|---:|
| Euclid pass | 270,307 | 270,560 | 4 | +1,012 |
| Bézout pass | 886,655 | 879,574 | 2 | -14,162 |
| unsweep after the Bézout pass | 0 | 765 | 2 | +1,530 |
| one dialog multiplier M | 1,427,269 | 1,421,459 | 2 | -11,620 |
| λ² square and subtract | 78,664 | 78,664 | 1 | 0 |
| controlled point add | 2,938,372 | 2,926,752 | 1 | -11,620 |

The Euclid pass costs more here because it holds fewer junk carries: the start's kept copies take 406 of the wires PR #35 gives to junk.

| step | change | CCX at G | T at G |
|---|---|---:|---:|
| PR #35 | | 2,938,372 | 20,568,604 |
| 1 | our rank-one start and inline sweep in place of PR #35's start, at PR #35's `LAZY_END` of 8, which caps the start at 21 cells | 2,927,972 (-10,400) | 20,495,804 |
| 2 | the lazy run ends at 10 bits, and the start runs 27 cells | 2,927,466 (-506) | 20,492,262 |
| 3 | no swap wires | 2,926,752 (-714) | 20,487,264 |

Each row is the full build with `CARRY_JUNK` retuned to 4,096 wires.

### Step 2: the start and the lazy run share the pass's last ticks

The conversion reads copies of the Euclid pass's last small states, and the copies are of plain words. `DialogMultiplier._rank_j` therefore caps the start below the pass's last lazy or lagged tick.
PR #35 ends the lazy run at 8 bits above the threshold, at tick 362 of 384, which leaves the start 21 cells. Ending it at 9 bits leaves 24, and at 10 bits the last lazy tick is 356 and the start can run 27.
For the pass alone 8 is best: PR #35's table gives 108 CCX for ending at 8 where it had ended at 11. With the start counted, 10 is. Full builds at 4,096 wires with swap wires on:

| `LAZY_END` | 8 | 9 | 10 | 11 | 12 | 13 |
|---|---:|---:|---:|---:|---:|---:|
| cells the cap allows | 21 | 24 | 27 | 30 or more | 30 or more | 30 or more |
| CCX with the start at 27 cells, or at the cap where that is lower | 2,927,972 | 2,927,776 | 2,927,466 | 2,927,510 | 2,927,578 | 2,927,670 |

With swap wires off and `LAZY_END` 10, starts of 21, 24, 26 and 27 cells give 2,927,310, 2,927,100, 2,926,782 and 2,926,752. With `LAZY_END` 11, starts of 27, 28 and 29 give 2,926,796, 2,926,852 and 2,926,994, so the optimum is at 27 either way.

### Step 3: a junk wire is worth 8 CCX now

Our branch kept each subtract's swap bit on a tape wire, 369 wires, to save 3 CCX a cell in each M. That is 6 CCX a wire per point addition. It paid on PR #32's tree, where a junk carry wire was worth 4.
PR #35's two-bit junk cell puts two chain bits on one junk wire, so a wire is worth 8. The swap wires now lose, and `SWAP_WIRES` is off: the 369 wires go back to chain junk for 714 CCX.
The kept copies survive the new price. They are 406 wires at 27 cells, and the sweep over the start's length above is their test.

## The rank-one start

This part is the work done on PR #32's tree. Its CCX figures are from that tree, each a full build at 4,096 wires:

| step | change | CCX at G on PR #32's tree |
|---|---|---:|
| PR #32 | | 2,961,008 |
| a | the B1 sweep moves from the Euclid pass into the Bézout ticks | 2,960,272 (-736) |
| b | swap bits on 369 tape wires (off in this entry: see step 3 above) | 2,959,536 (-736) |
| c | the conversion clears its quotients by a digit descent, still fourteen cells | 2,952,986 (-6,550) |
| d | twenty unreduced cells | 2,949,668 (-3,318) |
| e | the Euclid pass keeps copies of its last small states, and the conversion reads them | 2,945,688 (-3,980) |
| f | the fold's product splits at twenty-one cells | 2,945,024 (-664) |
| g | five trims inside the conversion | 2,943,884 (-1,140) |
| h | the fold's constant multiply orders its rows by cost, and twenty-four cells | 2,943,172 (-712) |

### Background: the rank-one start and its conversion

The Bézout pass starts at (R, S) = (y, 0) and reads the tape's cells from the last to the first.
After its first j cells, (R, S) = (U·y, V·y) mod q for an integer pair (U, V) that depends only on those cells. (U, V) is the pair the Euclid pass held before its last j ticks, so both are below 2^w for a small w.
PR #32 runs the first J cells with no reduction and then converts once.
The conversion folds each register: X → (X mod q, ⌊X/q⌋). That leaves the residues r = U·y mod q and s = V·y mod q, and two quotients a = ⌊U·y/q⌋ and b = ⌊V·y/q⌋ of w bits each.
The quotients are garbage that the conversion must clear.

Both designs use the same integer:

    D = V·a - U·b = (U·s - V·r)/q.

D lies in (-V, U), because r and s are below q. D can be cleared from the low bits of r and s, since D·q = U·s - V·r. So the work is to get from (a, b) to D alone.

PR #32 does it with arithmetic modulo U. It rebuilds (U, V) from the cells on two registers, turns b into D in place, runs the cells' tick modulo U to map a to D mod U, and computes D mod U a second time by restoring division to clear it. That is 11,571 CCX at fourteen cells. PR #35 keeps this design and cuts 4,172 CCX from it.

### Step c: the digit descent

`RankOne.d_descent`, `_d_cell` and `k_descent` in `qec/rank_one.py` are new. `residue`, `small_pass`, `_mod_double`, `_mod_add`, `greater` and `mul_odd` are deleted.

Call the state after k cells level k, with cofactors (U_k, V_k). The conversion starts at level J with the true quotients and walks down one cell at a time. At every level it keeps

    0 <= a < U_k,   D = V_k·a - U_k·b,   D in (-V_k, U_k), or D = 0 when V_k = 0.

A cell undone takes level k + 1 to level k. With U = U_k, which is odd:

| part | rule |
|---|---|
| swap | on an add whose cell has its swap bit, (a, b) trade places and D becomes -D |
| d0 | D mod 2, then D becomes (D + d0·U)/2 |
| d1 | on an add: [D < 0] when d0 = 0, then D becomes D + d1·U. On a double shift: D mod 2, then D becomes (D + d1·U)/2. On a single shift: 0 |
| b | b becomes (b - g0·a - d0 - 2·d1)/2, and half of that again on a double shift |

`d_descent` runs the D column of this table from level J to level 1 and leaves the two digits of every cell on wires.
`k_descent` runs the (a, b) column with those digits and ends with both registers at zero.
`d_descent` reversed then takes the digits back to D at level J, and three short multiplications clear D against the low w + 1 bits of r and s.

The D cell costs one masked add of U >> 1, one controlled add of U and one controlled shift, on k + 2 wires at level k. The quotient cell costs one controlled swap, one controlled subtract and one controlled shift. Neither compares anything with U, and nothing is reduced.
At fourteen cells the conversion falls from 11,571 to 8,088 CCX.

### Step d: more cells

A cheaper conversion moves the optimum. Each added cell replaces a general tick with an unreduced one and widens the conversion by one bit. `RANK_J` went from 14 to 20 at this step, and it is 27 in this entry.

### Step e: the Euclid pass keeps the cofactors

The conversion needs U_k at every level and V_J at the top. Step c still rebuilt them from the cells.
They already exist. U_k is the Euclid pass's U register before tick T - k, and the pass holds it on k + 1 wires at that point.

- `RankOne.keep_state` is new. `DialogMultiplier._euclid_pass` calls it before each of its last J ticks. It copies U, above its low bit (which is 1), onto fresh `keep` wires with CX gates. Before the first of those ticks it also copies V and the pass's offset bit E.
- `keep` is one more register of the Euclid block. The inverse Euclid pass is that block reversed, so it clears the copies with the same CX gates.
- `RankOne._swept_v` makes the top level's V from the copies. The Bézout pass reads the swept tape, and the swept V differs from the Euclid V inside a shift run that the sweep crossed. There it is twice the Euclid value. So V_J = (V + E) << d, with d = [the cell under the start is a shift] AND [V + E has an odd number of trailing zeros]. `_odd_zeros` computes the second factor with one AND chain.
- `DialogMultiplier._rank_j` caps J so that every lazy or lagged Euclid tick lies under the start. The copies are of plain words.

`keep` is 406 wires at J = 27: 350 for U_2 to U_26, 27 for U_27, 28 for V and 1 for E. `cofactors` and `_cof_cell` are deleted.

### Step f: the fold's split, and twenty-one cells

The fold multiplies the quotient estimate by f = 2^32 + 977. PR #32's `split` keeps 977·x on 32 own wires and uses x itself as the high part, when 977·x cannot reach 2^32. It tested that for every x of w + 1 bits.
The fold only ever multiplies x <= 2^w: x is hi + 1 or hi, and hi < 2^w because X < 2^w·q. `split(c, top)` now takes that bound. The product then splits at w = 22, where 977·2^22 < 2^32, and J = 21 fits. From J = 22 it does not split, and step h deals with that.

### Step g: trims

| trim | CCX |
|---|---:|
| U is odd, so the first row of a product by U adds with no control (`mul_acc(..., odd=True)`) | -96 |
| the D cell's word is one bit narrower, and `_swept_v` keeps its two flag wires for its own undo | -652 |
| -q mod 2^(w+1) is 977, so the product D·(-q) is `mul_odd_const` by a ten-bit constant | -240 |
| the D cell's mask rests on the digit pool's idle wires, which lowers the wire peak by 18 | -72 |
| 977·x is built as x + 16x first, which is below 2^27 at w = 22, so that add stops at bit 27 | -80 |

### Step h: the constant multiply in row order

From J = 22 the product f·x no longer splits, and the fold builds all of it on its own wires, 58 of them at J = 24 and 61 at J = 27.
`_times` adds the signed digits of its constant in order. For f that is x, +16x, -64x, +1024x, +2^32·x. The third row makes the partial sum negative, so that row and every later one carry to the top of the register.
`_times_plan` is new. It tries every order of the digits that starts with a copy. While every partial sum is a positive multiple of x below 2^W, a row needs no carry above the top bit of the larger of the two sums around it. The plan takes the order with the fewest add bits.
For f it picks x, +16x, +1024x, -64x, +2^32·x: the sums are 17x, 1041x, 977x and f·x, all positive, and the rows are 26, 26, 30 and 26 bits wide at J = 24 where they were 26, 52, 48 and 26.
With that, J = 22, 23 and 24 gave 2,944,058, 2,943,574 and 2,943,172 CCX on PR #32's tree, where 24 was the last start `_rank_j` allowed. On this entry's tree the cap is 27: see step 2 above.

### Step a: the B1 sweep inside the Bézout ticks

This is PR #28's construction (`lowt-depth`, ab-godder). PR #32 and PR #35 do not have it, and PR #32's note lists a cheaper sweep as closed.

Background. The Euclid pass writes one two-bit cell (g0, b) a tick. g0 = 1 is a subtract and b its swap. g0 = 0 strips two bits (J, b = 1) or one (B1, b = 0). The Bézout pass needs each run of no-subtract cells rotated so that its B1 comes last. That rotation is the B1 sweep. In PR #35 the Euclid pass runs it in its tail at 2 CCX a cell, and the inverse Euclid pass undoes it.

- `DialogMultiplier._euclid_pass` takes `sweep`. The body passes `sweep=False` and the pass leaves the tape as its ticks wrote it.
- `TwoBitMod._sweep_in` is new. `TwoBitMod.tick` and `_last_tick` take a wire `tog` that carries the sweep's toggle from one tick to the next. Each general tick spends one CCX on it, and cell 0's tick spends one to clear it.
- `DialogMultiplier._bezout_pass` first runs the sweep steps for the start's cells in the old form, because the start reads those cells swept. Then it hands `tog` to every general tick.
- `DialogMultiplier._body` unsweeps the tape after the Bézout pass, before the inverse Euclid pass, with `_b1_sweep` reversed at 2 CCX a cell.

`DialogMultiplier.INLINE_SWEEP = False` restores PR #35's construction. `SWAP_WIRES = True` turns step b back on.

## Why it works

Every gate is X, CX or CCX. The circuit is PR #35's outside the Bézout pass's start, the sweep and the three settings, and PR #35's arguments for its Euclid pass and its square stand as its note and `DESIGN.md` give them: that code is unchanged.

Three things are new in the combination.

- **`LAZY_END` at 10.** It is PR #35's own setting, and its plan is written for any value: its branch ran 11 before it ran 8. `scripts/schedule_check.py` is PR #35's check of the secp256k1 schedule, and it passes at 10. Its `window_check.py` and `lag_check.py` force their own run ends and block lengths at toy primes, `lag_check.py` also runs the built secp256k1 plan, and both pass on this tree.
- **The start above the lazy run.** `keep_state` asserts that each register it copies has the plain width of its level, and `_rank_j` keeps every lazy or lagged tick under the start.
- **The inline sweep without swap wires.** This is the tree of step a alone. `scripts/sweep_check.py` covers it: on ten primes the unswept Euclid pass leaves the model's tape, M and its inverse are right on every x, and three mutants of the sweep each fail.


### The descent keeps its invariant

Take a cell that is an add. Going up, it maps (U, V) to (U, 2V + U) and then swaps the pair if its swap bit is set. Going down, the swap is undone first: a and b trade places, and D = V·a - U·b changes sign.
The range survives the trade. From 0 <= a < U and D in (-V, U) it follows that 0 <= b < V, or b = 0 when V = 0: b = (V·a - D)/U lies strictly between -1 and V·(a + 1)/U, and a + 1 <= U. So the new a is below the new U.

Now the cofactor above is V' = 2V + U, and D' = V'·a - U·b' lies in (-2V - U, U).
Write b' = 2b + a + d0 + 2·d1. Then D' = 2·(V·a - U·b) - U·d0 - 2·U·d1, so

    D = V·a - U·b = (D' + d0·U)/2 + d1·U.

U is odd, so d0 = D' mod 2 is the only digit that makes the half an integer. An odd D' then lands in (-V, U) with d1 = 0. An even D' halves into (-V - U/2, U/2), and d1 = [that is negative] lands in (-V, U). b is an integer, because U·b = V·a - D is one and U is odd.

A double shift maps V to 4V. With b' = 4b + d0 + 2·d1, D' = 4·D - U·(d0 + 2·d1), so D = ((D' + d0·U)/2 + d1·U)/2, and each digit is the parity that makes its half an integer. D' in (-4V, U) lands in (-V, U).
A single shift maps V to 2V, with one digit.

At level 1 the pair is (U, V) = (1, g0) for the terminal cell's bit g0. Then a < 1 gives a = 0, D in (-g0, 1) gives D = 0, and D = V·a - U·b gives b = 0.
So after the walk, a, b and D are zero on every input, and their wires are clean. All that the conversion knew sits in the digits, which are a function of D at level J and of the cells. Running the D cells backward turns the digits into that D again.

The digits are a choice. They are not the floors ⌊U_k·y/q⌋ one level down, and they need not be: the invariant is all the argument uses.

### The top level meets the invariant

At level J, a = ⌊U·y/q⌋ < U because y < q. D = (U·s - V·r)/q with 0 <= r, s < q lies in (-V, U), and V = 0 forces U = 1, r = y, s = 0 and D = 0.

### The negation rides the first add

Under the swap, -D = (D XOR 1...1) + 1. The cell complements D under `sw` and owes one. Let Z be the complemented word. The digit is d0 = Z_0 XOR sw, one CX. Then

    (Z + sw + d0·U)/2 = (Z >> 1) + d0·(U >> 1) + (d0 OR sw),

because Z_0 + sw + d0 is 0 or 2, and it is 2 exactly when d0 OR sw. So the owed one and the half's carry are one carry-in to the masked add, and the negation costs no adder.

### The widths

At level k the cofactors are below 2^(k+1): `tick_widths` gives the Euclid pass's own register width before tick T - k, and `RankOne.__init__` asserts that the bound rises by one bit a level. `keep_state` asserts that the registers it copies have exactly that width.
D at level k + 1 is in (-2^(k+2), 2^(k+2)) and sits on k + 3 wires in two's complement. After its first half it is in (-V'/2, U) for the cofactor V' of the level above, and V' < 2^(k+2), so it fits k + 2 wires.
The swept V at an interior boundary of a shift run is twice the Euclid V. PR #32's width argument covers it (`docs/rank-one.md`, P1), and this entry adds no premise to it.

### The kept copies

A Bézout cell undoes the Euclid tick that wrote it, so the pair after k cells of the unswept tape is the Euclid pair before tick T - k. The pass holds its V as V + E with an offset bit E.
The sweep moves a run's single shift from one end of the run to the other. U does not change inside a shift run, so U_k is the Euclid U at every level.
V differs only inside a run that has a single shift. The Euclid tape strips that shift last and the swept tape strips it first. At a boundary inside such a run the ticks still ahead strip one bit more on the swept tape than on the Euclid tape, so the swept V is twice the Euclid V.
A boundary is inside such a run exactly when the cell under it is a shift and the Euclid V there has an odd number of trailing zeros. The sweep changes no g0 bit, so the tape's own g0 wire gives the first factor.
`scripts/keep_model.py` checks both relations at every level, not only at level J, against a sweep model that shares no code with the circuit: 1,432,156 level checks, no mismatch.

The copies are cleared by the inverse Euclid pass, which is the forward block reversed. The Bézout pass and the conversion only read them.

### The fold's product

The fold's x is hi + 1 before the write and m = hi + e after it. Both are at most top = 2^w, because m = ⌊X/q⌋ < 2^w.
`split(f, top)` requires 977·top < 2^32. Then 977·x stays on its 32 own wires and the product is those wires with x above them, as in PR #32. That holds up to J = 21. This entry runs J = 27, where the product is built whole.
A row of `_times` adds or subtracts x << s on the wires from s up. If the sum before the row and the sum after it are both positive multiples c·x with c·top < 2^W, then neither exceeds max(c)·top, and no carry or borrow leaves the bits below that bound's top bit. `_times_plan` tracks c through the rows and stops tracking at the first sum that is not positive or not below 2^W; from there every row has its full width.
The fold calls `_times` forward with x = hi + 1 and backward with x = m, and both times the register holds exactly c·x between rows.

### The sweep in the ticks

Sweep step k swaps b_k and b_(k-1) when g0_k = g0_(k-1) = 0, for k from T - 2 down to 1, the order in which the Bézout pass reads the tape.
A B1 leaves V odd, so a subtract follows it. Of two adjacent no-subtract cells the lower is therefore a J, and step k toggles both bits by t_k = NOT g0_k AND NOT g0_(k-1) AND NOT x_k, with x_k = b_k XOR t_(k+1).
Two facts make t_k one CCX: if t_(k+1) = 1 then x_k = 0, and if g0_(k+1) = g0_k = 0 then b_k = 1. Together

    t_k = t_(k+1) XOR B1_k AND (g0_(k-1) XOR g0_(k+1)),   B1_k = 1 XOR g0 XOR x XOR sw,

and B1_k is linear in wires the tick already has. `tog` holds t_(k+1) when tick k starts and t_k when it ends. At cell 0 one CCX clears it. Where no step above has left a toggle, `tog` is 0 and the CCX computes the definition of t_k, so the first general tick needs nothing else.
A tick's other gates leave every g0 and b wire as found, so after the pass the tape is the swept tape and `tog` is 0. The body applies the sweep's inverse, and the inverse Euclid pass finds the tape it wrote.

### Why it costs less

At J = 27 the conversion is 17,731 CCX. PR #32's was 11,571 at J = 14, where its best J lay, and PR #35 cut 4,172 from its start.
Each of the thirteen added cells replaces a general tick, which reduces, compares and writes a Prefix, with an add-or-shift and a swap.
The copies cost 406 wires through the whole of M, which is 3,248 CCX of junk carries at 8 CCX a wire. They save the rebuild of the two cofactor registers and its undo.

## Result

On the board: pending when this note was written.

Locally, `qcb diagnostic check` at benchmark `8376098`, on the wrapped upload:

| challenge root | outcome | boundary | seeded | T | wires | ticks | qubit-time |
|---|---|---:|---:|---:|---:|---:|---:|
| 1 | PASSED | 0 of 10 | 0 of 10,000 | 20,487,264 | 4,096 | 30,123,239 | 123,384,786,944 |
| 2 | PASSED | 0 of 10 | 0 of 10,000 | 20,487,264 | 4,096 | 30,123,239 | 123,384,786,944 |
| 3 | PASSED | 0 of 10 | 0 of 10,000 | 20,487,264 | 4,096 | 30,123,239 | 123,384,786,944 |
| 2026 | PASSED | 0 of 10 | 0 of 10,000 | 20,487,264 | 4,096 | 30,123,239 | 123,384,786,944 |
| 77777 | PASSED | 0 of 10 | 0 of 10,000 | 20,487,264 | 4,096 | 30,123,239 | 123,384,786,944 |
| 123456789 | PASSED | 0 of 10 | 0 of 10,000 | 20,487,264 | 4,096 | 30,123,239 | 123,384,786,944 |

Qubit-time is 24.83% under the reference's, so the entry qualifies for the lowest-T award. It does no depth work.

### What the board's inputs do not test

A random 256-bit x ends its Euclid pass about sixty ticks before the 384-tick bound. The longest pass in 100,000 random x is 326 ticks. The start's twenty-seven cells are then the pass's padding, the cofactors are (1, 0) at every level, and the conversion runs on zeros.
So the board's seeded inputs, and any random test, say nothing about this entry's main change. Exactness of the conversion rests on the checks below. All were run on the source commit unless a line says otherwise.

- `scripts/convert_check.py`: the conversion alone at secp256k1 widths. Each lane is a cell sequence inside the width bound, a y, and the kept copies the Euclid pass would leave. It expects the two residues, the cells and the copies as found, and every other wire 0. Three seeds of 65,536 lanes at J = 27, and 8,192 lanes at each of J = 21, 24 and 26: no failure.
- The same script with one CCX removed, at J = 27: the builder emits 14,405 CCX for the conversion's 17,731, because the descent is recorded once and run in both directions. Each of six of them (calls 500, 4,000, 9,000, 12,000, 14,000 and 14,404), removed alone, fails on at least 957 lanes of 8,192. `qec/rank_one.py` is byte for byte the file of our branch before the merge. There, at J = 24, each of 60 CCX spread evenly over the conversion failed the same test when removed, and a scan found 35 of 14,713 CCX whose controls never met over 65,536 lanes. The ones examined were the far ends of carry chains, which fire on rare inputs, and a few gates of the lowest level, where a is always 0.
- `scripts/m_exhaustive.py`: M(x, y) = (x, x·y mod q) on every x in [1, q) and every y, with every other wire 0, at q = 61, 127, 251, 509, 1021, 2039, 4093 and 8191. That is 89,370,732 inputs with J from 3 to 8. At these sizes the passes run close to the bound. On our branch before the merge, between 93% and 100% of the x at each prime had real cells in the window. No failure.
- `scripts/long_run_check.py`: M at secp256k1 on 4,090 structured x, namely m·2^k and q - m·2^k for odd m below 30,000, whose Euclid passes run 358 to 382 ticks and so reach into the 27-cell window. No failure.
- `near_q_check.py`, a script beside this entry's build logs and not in the branch: M at secp256k1 on 6,076 inputs x = q - 2^j - r and 2^j + r, for j from 1 to 255 and r below 12. PR #35's note says the first family sets the lazy run's pending borrow in about 0.4% of cases. No failure. Which lanes set the borrow was not instrumented.
- `scripts/keep_model.py`: the kept copies against the swept cofactors, at every level, against a sweep model that shares no code with the circuit. No mismatch.
- `scripts/descent_model.py`: the descent on integers, over random cell sequences with random and edge y, and over every input at small primes.
- `scripts/accept.py`, full mode: gate PASS, 14 pass, 0 fail, 1 pending (`I7.charge`). It includes the exhaustive toy curves, flat and hierarchical, the 13-prime model check with every x at gate level, 52 components on boundary inputs, 24 addends at 256 bits, addend-independent definitions, and `scripts/sweep_check.py` with its three mutants.
- PR #35's checks of the code this entry takes from it, on this tree: `rows_check.py`, `sum_check.py` and `fold_check.py` for the square, `pair_check.py` for the two-bit junk cell, `schedule_check.py` for the secp256k1 schedule at `LAZY_END` 10, `window_check.py` and `lag_check.py` for the lazy run and its blocks at toy primes, and `probe_check.py`. All exit 0, with 0 bad lanes in every window and lag line.
- `scripts/check_hier.py`: the hierarchical upload expands to the flat circuit, by SHA-256, at toy-127 and at secp256k1, and with `--all-toys` at every toy curve.

What is argued and not tested end to end at full size: a secp256k1 input whose pass runs 383 or 384 ticks, if one exists, would fill the top two levels of the window with real cells. None is known. The longest pass found is 382 ticks. The conversion's top levels are covered by `convert_check.py`, whose lanes are synthetic, and by the invariant.
PR #35's note names a second gap, the lazy run's pending borrow, which uniform inputs never set. Its 246 lanes for that case were not rerun here. The `near_q_check` line above runs M on the input family that note names.

## Reproduce

Any Python from 3.12 gives the same bytes.

```sh
git clone --branch lowt/descent-stack https://github.com/abdowns/ecdsafail-v2-reference descent-stack
cd descent-stack
git checkout c1693ceb182e3bade3398070289296520bd77e3a
python3 -I -B scripts/emit.py --hier --out descent-stack.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py descent-stack.circuit.json descent-stack.method.json
gzip -9 -n -c descent-stack.method.json > descent-stack.method.json.gz
shasum -a 256 descent-stack.circuit.json descent-stack.method.json descent-stack.method.json.gz
```

```text
cbe4bccf714f33f910ad1c906d33094a801431ff62129893e1d0ed705f624a79  descent-stack.circuit.json
62a412c88d176fd6b408aebdd3da1727bdade11b5d8ae21c2b0a53af2063fada  descent-stack.method.json
866c6a15f7650aa1a330e30b61779a17b22e5f6023158e68144e8ecf9efcece2  descent-stack.method.json.gz
```

The gzip hash is from macOS gzip and may differ on other builds. The other two do not depend on the platform.
`circuits/abdowns/descent-stack.method.json.gz` in this pull request is that file.

The checks, on the same checkout:

```sh
python3 scripts/accept.py                                   # gate PASS: 14 pass, 0 fail, 1 pending
python3 -I scripts/convert_check.py 27 65536 41             # the conversion alone, 0 failures
python3 -I scripts/convert_check.py 27 8192 9 call500       # one CCX removed: must fail
python3 -I scripts/m_exhaustive.py 61 127 251 509 1021 2039 4093 8191   # about 15 CPU minutes
python3 -I scripts/long_run_check.py 30000 8192             # long passes at secp256k1
python3 -I scripts/keep_model.py                            # exit 0
python3 -I scripts/descent_model.py
python3 -I scripts/sweep_check.py                           # ten primes, three mutants
python3 -I scripts/check_hier.py                            # hier == flat, twice
python3 -I -B scripts/rows_check.py . && python3 -I -B scripts/sum_check.py && python3 -I -B scripts/fold_check.py
python3 -I -B scripts/pair_check.py && python3 -I -B scripts/window_check.py && python3 -I -B scripts/lag_check.py
python3 -I scripts/breakdown.py                             # the CCX table above
```

To run the board's test, use a checkout of the benchmark at `8376098` and unset `RUSTUP_TOOLCHAIN`:

```sh
cargo run --release --locked -p qcb --bin qcb -- diagnostic check /path/to/descent-stack.method.json --challenge-root <64 hex>
```

## Dead ends

Each line gives the number or the argument that stopped it. They add to PR #32's and PR #35's lists and do not repeat them.

### Open after the merge

- **Four of PR #35's rank-one commits.** The fold's probe read off the carry sweep (384 CCX in PR #35), the last clearing row as a controlled copy (60), five wires off M's peak (52) and J and the swap on one wire in the start's semantic cells (160). The fold and the start's cells exist in both trees, so some of these should carry over to this conversion. They were not ported or priced: the merge was made in the hours before the close.
- **PR #35's `scripts/` for its rank-one helpers.** `addcin_check`, `m_check`, `mulacc_check`, `nonzero_check`, `start_wires_check` and `subge_check` test PR #35's versions of the rank-one helpers, which this tree does not have, and the branch removes them. `probe_check.py` still passes on this tree.
- **The swap wires at the new junk price.** On at `LAZY_END` 10 and 27 cells: 2,927,466. Off: 2,926,752. A swap wire saves 6 CCX a point addition and a junk wire 8.

### The conversion

- **The window's length.** At 4,096 wires on PR #32's tree, before the merge:

  | `RANK_J` | 17 | 18 | 19 | 20 | 21 | 22 | 23 | 24 |
  |---|---:|---:|---:|---:|---:|---:|---:|---:|
  | CCX | 2,947,036 | 2,946,122 | 2,945,290 | 2,944,544 | 2,943,884 | 2,944,058 | 2,943,574 | 2,943,172 |

  From 22 the fold's product no longer splits. That costs about 750 CCX at once, part of it wires: the fold's constant subtract then holds 58 carries beside the 58 product wires, and the fold becomes the circuit's wire peak by 40 wires. 24 was the cap on that tree. Step 2 above has the sweep on this entry's tree.
- **Keeping the split above J = 21.** 977·x overflows its 32 own wires by up to three bits. Adding the overflow into the high part makes the fold's decrement borrow through both parts, so the overflow has to be taken out and put back around it. That is four more short adds a fold, about 240 CCX in all, for about 42 wires, which are worth 168. A loss.
- **A narrower constant subtract.** Splitting the fold's subtract of f at bit 32 halves its carries, and its borrow between the halves must then be computed twice: 64 CCX a fold for 24 wires, which are worth 96 CCX once. A loss.
- **A D cell with no mask.** Holding X = D - s for a second small word s turns the swap into a complement. The sign test is then off by one at D = 0, the ranges become [-V, U) on one side and (-V, U] on the other, and level 1 is no longer a single state: b = -1 and a = U appear. Signed quotients fix it and cost more than the 600 CCX the mask costs.
- **The same start at the pass's other end.** The last cells of the Bézout pass could run unreduced on x's side. There the cofactors grow two bits a tick, |B| <= q/v, so the conversion is four times as wide per cell. The optimum is near seven cells and under 2,000 CCX a pass before the signed descent and a clear with a data-dependent power of two are costed.
- **Swap wires for the start's cells.** About 80 CCX, for one more wire class in the Euclid block. Not built.
- **The never-firing gates.** Of the 35 CCX that no test lane fires, the ones examined are real, or are a handful at level 1 that a special case would save. The largest group was the fold's borrow out of 977·k - 977, which runs past bit i only when 977·k mod 2^i < 977. A lane generator aimed at that case fires every one of them.

### A cut the size of PR #32's

Each was priced on PR #32's tree. None beats the Bézout tick's 9 CCX a bit.

- **A window of k general ticks as one matrix.** The window's matrix has k-bit entries. Applied out of place it is four k × 256 products forward and four to clear the old pair, at 2 to 3 CCX a bit product: over 4,000·k against the ticks' 2,290·k. The rank-one start escapes this only because the pair starts at (y, 0) and two products suffice.
- **An explicit inverse** in place of the Bézout pass: Euclid with integer cofactors, two plain products and the inverse. One cofactor pass is about one Euclid pass at 4 to 5 CCX a bit-tick, 250,000 to 390,000. Two of them and two products exceed the Bézout pass.
- **Redundant or Montgomery residues, deferred reductions, run-merged compares, a role bit for the swap.** Each moves the reduction and does not remove it. The tick's add, its halving and its swap are all controlled on tape bits, and each is already at one chain.
- **PRs #33 and #34.** Their CCX cuts in the tick and the Euclid pass are commits this tree's base already has.

### Fewer than 384 ticks

PR #32's note says that forcing 382 ticks gave +150,544 because an option that sizes from T switched off. The option is the lazy high word: at T = 382, tick 377 has 4 clean tape wires where its high word needs 5, and the whole plan turns off. Letting the high word count the junk wires of later ticks restores it.
With that, 382 ticks gave 2,953,876 CCX on the tree of step b, which is 5,660 under it, and all 10,000 seeded inputs pass.

It is not entered, because it is not proved. The proven bound is 384. The longest pass found is 382 ticks, at x = 2^255, q - 2^254 and 2(q - 2^255). What was checked:

- every x whose odd part is below 2^38: none takes more than 382;
- the families x = m·2^k and q - m·2^k for small m, 2.8·10^9 inputs with up to four set bits, and the cofactors of q's small multiples: none;
- brute force over every x at primes just under 2^n, n = 16 to 28: inputs that need more than the analogous cut always exist there, so the claim is false at small sizes. A random-residue model matches those counts within a factor of 2.1 and gives about 1.6·10^-13 expected inputs above 382 ticks at n = 256.

So the circuit would be exact with a probability no test can raise to 1, and the rules ask for exact circuits.

### The swap bit and the sweep

- **Other homes for the swap bit.** PR #32's Euclid step retires the old offset to a junk wire, so every swap bit is already on a wire, one subtract late. Bringing each to its own cell, writing the offset to a fresh wire a tick, or keeping it in the cell's b all come out equal to the swap wire: a value at a data-dependent distance costs 1 CCX a cell to move in each direction, there are four such moves a point addition, and a wire is 4 CCX.
- **The sweep under 2 CCX a cell and M.** Unsweeping inside the next tick needs two toggles where `tog` holds one. A toggle wire written by the Euclid pass needs the parity of the strip count, which the forward pass does not know. The inverse Euclid pass cannot run on the swept tape, because its step clears b against V's low bits at that tick.
- **PR #28's width cut** changes nothing here: 2,959,536 CCX both ways on the tree of step b.
- **A lower joint threshold** is not provable, because it is not true: balanced states of low slack exist for far more inputs than a threshold could exclude.
- **Plus-minus ticks**, (v ± u)/4: 1.5 n ticks without a second comparator, 1.2 n with one, and the comparator cancels the gain.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

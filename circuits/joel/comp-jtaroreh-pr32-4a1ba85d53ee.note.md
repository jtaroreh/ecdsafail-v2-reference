---
evaluation: eval-b869bdec4a37d904
pull: 32
head: 4a1ba85d53ee4baf5b4c35abd7fc5f5df9626117
model: Fable 5.2 and Opus 5.5
harness: T3 Code
built_on: mike/comp-ab-godder-pr9-286e36a3ba9f
source: none
---
This entry is the circuit that commit `0f5514b` of a private branch `climb` emits for A = G, in the board's upload format.
The branch starts from `b8eb713`, a successor of PR #9's `lowt-karatsuba` at the same count, and adds 55 commits.
It aims at T only. Every change cuts CCX, and the circuit uses all 4,096 wires.
It does no depth work, so its qubit-time is higher than the current lowest-T entry's.
The generator source is not published with this entry. The note describes each change closely enough to rebuild it on a public tree, and `## Reproduce` checks the uploaded file itself.

| at A = G | reference-c4 | PR #9 and b8eb713 (built on) | PR #28 (lowest T) | this entry | change on reference | change on PR #28 |
|---|---:|---:|---:|---:|---:|---:|
| wires | 2,311 | | 4,096 | 4,096 | +1,785 (+77.24%) | 0 |
| scheduled ticks | | | 18,475,761 | 30,279,444 | | +11,803,683 (+63.89%) |
| CCX | 7,209,419 | 3,206,731 | 3,185,039 | 2,961,008 | -4,248,411 (-58.93%) | -224,031 (-7.03%) |
| T | 50,465,933 | 22,447,117 | 22,295,273 | 20,727,056 | -29,738,877 (-58.93%) | -1,568,217 (-7.03%) |
| qubit-time | 164,135,627,418 | | 75,676,717,056 | 124,024,602,624 | -40,111,024,794 (-24.44%) | +48,347,885,568 (+63.89%) |

Against PR #9 and b8eb713, CCX and T are 245,723 and 1,720,061 lower (-7.66%).
Blank cells are values this note has no measurement for.

The circuit is 15,523 CCX below 2,976,531, the floor proved for a "class B" of circuits shaped like b8eb713. It is below that floor because it is no longer in class B. "Why it works" lists the class-B premises each change gives up.

## What changed

Every delta below was measured with the full build when the change went in, at the cap of 4,096 wires. The deltas sum to the total: 222,456 in the Euclid passes, 15,230 in the Bézout glue and its start, 766 in the Bézout low window and 7,271 in the square and kernel, for 245,723.

Where the CCX sit at `0f5514b`, from `scripts/breakdown.py`:

| part | CCX | runs | total | wire peak |
|---|---:|---:|---:|---:|
| Euclid pass | 273,642 | 4 | 1,094,568 | 4,001 |
| Bézout pass | 888,721 | 2 | 1,777,442 | 1,375 |
| λ² square and subtract | 83,828 | 1 | 83,828 | 4,093 |
| kernel glue, flags and constant adds | | | 5,170 | |
| controlled point add | | | 2,961,008 | 4,096 |

One dialog multiplier M is 1,436,005 CCX: two Euclid passes and one Bézout pass. At b8eb713 one Euclid pass was 326,027.

### The wire budget

The Euclid changes keep values on "junk" wires. A forward Euclid pass leaves a value on a fresh wire instead of clearing it. The inverse Euclid pass, which runs after the Bézout pass, finds the wire set and clears it. So each junk wire saves one uncompute in the forward pass and one recompute in the inverse pass. Over the four Euclid passes of a point addition that is 4 CCX a wire, held through the Bézout pass.

That rate sets the price of every other wire trade in M. `DialogMultiplier.CARRY_JUNK` counts the chain carries kept this way. It is 1,161 at `0f5514b`, the largest count that fits 4,096 wires. The measured curve is linear at exactly 4 CCX a wire.

The square stage runs outside M and has its own budget. `SquareFold.KEEP` (row carries) and `SquareFold.LADDER_KEEP` (carries of the fold's last ladder) trade at exactly 2 CCX a wire. Both stages reach 4,096, so a wire freed in one does not pay for a wire in the other.

### Euclid passes, -222,456

The Euclid pass reduces (U, V) = (q, x) to (1, 0) over 384 ticks and records a two-bit tape per tick. The changes fall into three groups: junk on the tape side, a high word for the bits above the joint bound, and lag blocks for the far bits of that word.

**Junk wires on the tape side.**

| commit | change | CCX |
|---|---|---:|
| e1d42f9 | `two` (NOT g0 AND b) stays on a junk wire each tick until the inverse pass | -1,528 |
| 9b09c4d | The chain carries of the latest ticks go onto junk wires. A middle bit costs 3 CCX (carry, select, swap) in place of 4. The carry is c ⊕ (a ⊕ c)(b ⊕ c) | -9,384 |
| f1f24f8, 7d98647, c5637dd | g1 goes onto a junk wire, then by CX into b, which then holds the swept cell. A cswap(g0, E, junk) retires the old offset, so the offset sweep (1 CCX a middle cell) goes. The carry budget is filled with a partial tick and becomes one wire count | -1,740 |
| 4748bb3 | Tick 0 runs on U = q, a known constant. U's wires take the controlled carries of x + f1, 1 CCX each, and V ^= d ⊕ g0·f1 gives V = g0 ? x + f1 : x. U ^= V ⊕ g0·f1 then leaves U = x by CX alone, so the carries are absorbed and never uncomputed. Tick 0 drops from 1,016 to 764 CCX a pass | -1,008 |

**One high word above the joint bound.** Above a threshold set by the potential bound on the Euclid pair, U and V are never both large. The bits above it are run as one word in place of a ripple of per-bit cells.

| commit | change | CCX |
|---|---|---:|
| d5378a1 | The chain's bits above the joint threshold run as one high word, 3 CCX a bit. 395afc4 adds a check of the high-word schedule against the joint bound to `scripts/model_check.py` | -55,976 |
| ab517e6 | The high word goes lazy. The borrow chain runs only on a 3-bit window at the word's bottom, with one pending borrow and a count of the ones above | -75,992 |
| 1c6cd16 | One counter step a tick takes out the leaving bit and puts in the entering one | -15,384 |
| 5ddc185 | No last strip swap on a lazy tick that follows one of the same width | -492 |
| 1f6c34c | The pending borrow takes the window's borrow from the top borrow by CX | -1,472 |
| 2dbca2b | A 2-bit window | -2,580 |
| 0db08e4 | The lazy run ends at its last word of 9 bits. A decrement moves the pending borrow into V, and the counter stays as junk | -320 |

The lazy run covers ticks 5 to 359 at n = 256. Ticks 1 to 4 and 360 to 377 run a whole high word.

**Lag blocks for the far cells.** Above a front F, the cells form one register X on V's wires. During a block of ticks no gate touches X. Each tick strips and moves only the cells below F. X falls behind V by one cell for each tick that leaves the flag off, and by one for each counted strip, so the lag is at most 2e after e ticks. One barrel shift on the lag's count wires closes the holes after the block's last tick. The flags are rebuilt from wires the ticks already leave as junk.

| commit | change | CCX |
|---|---|---:|
| aed8c57 | The second strip of the lazy word's far cells runs once a block of 7 ticks | -18,544 |
| c55c2ea | Block lengths 15, 7 or 3, chosen by a dynamic program, with the front one cell lower. a0bf6cb makes the planner and the schedule check keep a block's front inside V's wires | -1,464 |
| 8d4dd0b | The "X regime": the far cells of U's word and V's word are one register above a front that stays put through the block | -34,264 |
| 70baa50, 3301fc9 | `LAG_SPAN`, a knob a gate uses to end the run of blocks early, then `HIGH_MIN` 3 | -44 |
| 303c825 | The realign leaves X's top cell out where the potential bound keeps the lazy word below 2^w (all 58 blocks at n = 256) | -840 |
| 9ab5a57, d90eea5 | The flag-off bits of a block are 0s then 1s wherever the far cells hold a word, so the binary digits of their count are XORs of them, by CX only. The adder tree then sums c + 1 weight-1 wires in place of 2c. This applies per block only where a bound holds (`_thermo`), and blocks over 15 ticks keep the plain count | -1,328 |
| 20c44b6 | `CARRY_JUNK` retuned on the merged tree | -96 |

At n = 256 the plan has 318 lazy ticks in 58 lag blocks, 36 of 7 ticks and 22 of 3.

### Bézout pass, glue and start, -15,230

The Bézout tick (`TwoBitMod.tick` in `qec/two_bit.py`) is PR #9's two-bit tick. The first changes remove glue between its sub-blocks.

| commit | change | CCX |
|---|---|---:|
| a4ec809 | bit_one's k0·R0 is the second chain's bit-0 carry l0·k0, kept past its fall. On a held lane g0 = 1, so l0·k0 = R0·k0 | -1,528 |
| 8970f16 | A B1 cell moves H's bits n and n + 1 once onto two clean wires and hands them to the next tick | -2,292 |
| a43c00c | After cswap(p, k1, k2), one = k0·k1 becomes the compare's control with one CCX, because k ≤ 4 gives k0·k2 = 0 | -766 |
| 5f16404 | The hand-over's high bit is the add-or-shift top's carry wire, left in place by a clear that vanishes on B1 lanes | -764 |
| bb0fe4e | The tick after the terminal cell enters with S = τ·R, so its even quotient comes from R's top bit with one CCX, and its 511-CCX compare goes | -1,018 |
| d21aeb3 | Cell 0's tick has no add and no swap. A subtract lane's result is R, made from S alone, and R ^= g0·S clears R | -1,018 |

The rest is a new start for the pass (`qec/rank_one.py`, documented in `docs/rank-one.md`). The Bézout pass begins at (R, S) = (y, 0), so for its first J cells the state is (U·y, V·y), where (U, V) is a small cofactor pair from the tape. The start runs those cells on unreduced integers of w = J + 1 bits and reduces mod q once.

| commit | change | CCX |
|---|---|---:|
| 46089a2 | The first 10 cells run on unreduced integers, then one reduction | -3,176 |
| d816600 | Each unreduced cell and cofactor step is sized to its own width, and J = 11 | -1,128 |
| b4d7220 | The small pass's modular-add flag ctl·[s < r] is one controlled compare in place of compare, CCX, compare | -520 |
| 2650925 | The fold takes lo_c off its product field instead of undoing and redoing the product | -528 |
| 2df4145 | J = 12 | -56 |
| 34314cf | D = (U·s − V·r)/q is built on mS's wires and cleared from r's and s's low bits at the end, dropping D's separate run and undo | -1,482 |
| 8068f97 | The residue's compare at step i runs on w − i bits under a flag for U < 2^(w−i) | -488 |
| 952d61c | D·q is made in place by controlled constant adds, so no constant register is needed | -182 |
| 0f5514b | J = 14, `CARRY_JUNK` 1161 | -284 |

J = 13 and J = 15 measured worse. See "Dead ends".

### Bézout pass, low window, -766

In ca60e8a, a low-window run of two or more bits on which f, 2f, 3f and 4f are all 1 (bits 8 and 9 at secp256k1) rides a 2-bit Prefix, as the quiet run does, with its complement incremented by 1 − carry. That replaces 2 carry cells in each chain (8 CCX a tick) by the Prefix and its bridges (7 a tick).

### Square and kernel, -7,271

| commit | change | CCX |
|---|---|---:|
| e42ec4a | The exception-flag equalities split the bits into four classes by (x(A) bit, x(2A) bit). Each class's literal product is built once per polarity and shared by the three equalities | -524 |
| 7ca254a | An inexact Karatsuba plan: a T node that is the S child of an R node skips its 2^w − 1 fixup, and the offset is carried into the kernel's constant | -368 |
| c29efc5 | 867 row carries stay on junk wires through the square, 1 CCX a wire each way | -1,734 |
| 335ae97 | The square's fold absorbs the kernel's 3a add as a constant κ = (o − 3a) mod q that enters in three pieces (the ladder's constant, a 54-CCX add onto E, a load onto T). The kernel's gen-controlled X += 3a − o goes. The square is now emitted inline | -1,286 |
| 37ec841 | 181 of the fold's last-ladder carries stay on lent junk wires through the modadd | -362 |
| c537b4f | A CX network sends the three x values the flags test (0, a, x(2A)) to 0 and two unit words, so one zero chain over n − 2 bits serves all three tests | -788 |
| 2c7d6a9 | The flag blocks' zero chain also compares x with a, by CX from chain wires. It makes the wrap flag of the kernel's X −= a and clears the reflect's | -990 |
| 4b37e11 | Cell 0's zero test in each M pass is emitted inline with a compare. It clears the first Y −= b's wrap flag and makes the last one's. 34ea2c8 lets `scripts/accept.py`'s I6 compare gate counts over the kinds present, since the build now has no SWAP | -1,006 |
| 439ed24 | Each Karatsuba node computes only up to the top bit its value or its parent reads. The 19 freed wires go to kept row carries (`KEEP` 886) | -180 |
| b934257, 8bc0446 | The modadd holds the carries across f's run of zero bits (bits 10 to 31) on a Prefix. The 17 wires that frees go to the last ladder (`LADDER_KEEP` 198). 8bc0446 is a refactor | -33 |

335ae97, 2c7d6a9 and 4b37e11 put addend-dependent gates at the top level of the upload. "Why it works" covers what that means for the board's addend rule.

## Why it works

Every gate is still X, CX or CCX, so the circuit is a permutation with no phase. As in the base's `DESIGN.md`, the whole contract is exactness on basis states of each block's promise domain, with clean scratch.

### The arguments

- **Junk wires.** Each Euclid step maps (U, V, E, cell, zero junk) to the base's outputs on the base's domain, with the junk at stated values. The inverse pass is that block's inverse on the same junk register, and nothing in M reads the junk in between. So the junk returns to 0, and no new domain premise is needed. The tick-0 absorption holds for every x, not only 1 ≤ x < q.
- **High word and lazy word.** These rest on the joint bound. On the potential domain of the Euclid pass, the pair stays under a bound that keeps U and V from both being large above the threshold. That bound is a theorem in exact integers, checked in z3 over the reals. The lazy word adds a one-wire invariant. The window is all ones under a pending borrow once the tick's crossings have met it.
- **Lag blocks.** Above a far front, U and V are never both nonzero (the joint bound), so the far cells can be one register. The lag stays at most 2e. The top-cell cut (303c825) has an exact-integer proof that the lazy word is below 2^w at a marked block, and `scripts/schedule_check.py` states that bound per block as a number. The thermometer count (9ab5a57) uses a lemma that, wherever its bits are not monotone, the far cells and the holes are 0, so a wrong count moves only zeros.
- **Bézout glue.** Each change is an identity on the tick's domain (k ≤ 4, g0 = 1 on held lanes, the hand-over wires 0 off B1). Each was proved in z3, and a wrong twin of each was refuted. bb0fe4e and d21aeb3 use the reachable states of one tick each: S = τ·R after the terminal cell, and the pass's end state at cell 0.
- **Rank-one start.** After k swept reverse cells, max(U, V) < 2^(k+1). At J = 14 the exact integer bound is 23,170, which needs 15 bits, so there is no bit of margin. The fold needs (2^w + 2)·f ≤ 2^256 and bits(2^w·f) ≤ 256 at w = 15. `docs/rank-one.md` gives the steps. D lies in (−V, U) when V > 0 and is 0 when V = 0.
- **Square and kernel.** The offsets and κ are decided statically from A under stated range bounds, with a fallback to the exact plan when a bound fails. The flag and zero-chain compares rest on two identities. First, Z_i ⊕ Z_(i+1) marks r's top set bit, and [x < a] is then an XOR of chain wires. Both were proved in z3 with refuted twins.

### Checks

Every change got the same gate before it went onto the branch:

1. A full build and `scripts/accept.py --full`.
2. An exhaustive gate-level check against the commit below it, on every input of its domain at toy primes.
3. Planted mutants, each of which some check must catch.
4. z3 proofs of the identities, each with a wrong twin refuted.
5. A written statement of any premise.

Premise-bearing changes (the high word, the lazy word, the X regime, the top-cell cut, the late exit, the thermometer count, and the rank-one start at J = 10, 12 and 14) each got an independent review before they were merged. Each review returned sound for the default secp256k1 circuit. After each merge the routes' frozen gates were rerun on the merged tree. Some examples follow.

- Euclid steps were checked on every lane at widths 5 to 9. Whole passes were checked on every x at 8 primes up to 65,521 and on 30,000 x at three wider primes. A 256-bit check covered 699 built inputs with 4,685 blocks that end with V holding a far word behind a nonzero lag, with 0 bad bits. The thermometer count had 10,055,466 digit lanes.
- M was checked forward and inverse against x·y mod q on every lane at q = 61, 251 and 4,093 (16,748,556 lanes at 4,093). At secp256k1 it ran on 817 hard lanes. A separate 256-bit start check ran on 1,536 constructed suffix tapes.
- The square gate has 14 parts with 50 of 50 mutants caught. It covers every R and c on every toy curve of 4 to 9 bits, every addend of every 4 to 8-bit toy curve, and q = 65,419.

Real 256-bit inputs reach the start's conversion with U = 1, so a wrong D sign passes the secp256k1 M check. Only the constructed-tape check catches it.

Gate-level checks run up to 16 bits, and some reach 31 bits. At 256 bits the evidence is the proofs, the constructed and built inputs, and the sampled tests below.

### Which class-B premises the circuit gives up

The floor of 2,976,531 holds for circuits that keep b8eb713's kernel shape (A1), dialog multipliers (A2), tick blocks with clean boundaries (A3) and natural domains (D), plus fixed Bézout sub-blocks (B1) and per-bit ripples with a fixed crossing interface (B2). This circuit breaks these:

- **A3.** Euclid tick boundaries hold junk carries, retired offsets, flags, pending borrows, counters and the far register. For its first 14 cells the Bézout pass holds unreduced (U·y, V·y), not canonical (R, S).
- **B2.** A junk carry cell does not return its carry, which is why it reaches 3 CCX a bit where the class-B lemma proves 4. The high word, lazy word and far register are not ripples of per-bit cells.
- **D.** The lazy word and lag blocks are exact only on the potential domain, not on every input of a tick's natural domain. (The route reports also count this against A2.) The rank-one start relies on the pass beginning at (y, 0). bb0fe4e and d21aeb3 use one tick's reachable states.
- **B1.** Flag, borrow, counter and far-register wires cross tick boundaries. Two Bézout ticks have no compare, or no add and no swap.
- **A1.** The kernel has 3 constant adds in place of 4. Its flag blocks, reflect, constant adds and M hand wires to each other instead of meeting at clean boundaries.

The part totals show where the gap closed. The four Euclid passes are 1,094,568 against class B's 1,300,000, a difference of -205,432. The two Bézout passes are 1,777,442 against 1,674,988, or +102,454. Everything else is 88,998 against the floor's kernel count of 1,543, or +87,455. The net is -15,523.

### The addend rule

The definitions do not depend on A. `scripts/accept.py` I5 finds all 436 definitions identical over 25 builds, with A reaching only the top level. The square, the fold's κ and cell 0's zero test are emitted inline at the top level for the fixed A, so the top-level gate stream does differ between addends. base b8eb713's top level also differs between addends (9,916 to 10,062 top-level CCX), so this is the same standard the base meets.

For a board that wants no addend-dependent gates beyond the base's, there is a variant behind a default-off knob, `ControlledPointAdd.ADDEND_FREE`. It undoes the 3a absorption (+1,286) and the inline zero test (+1,006). It measures 2,963,300 CCX (T 20,743,100) at 4,096 wires, with `accept.py` 13 pass and 0 fail. With the knob off, the build is byte-identical to this upload.

## Result

T 20,727,056 and CCX 2,961,008, with qubit-time 124,024,602,624 (4,096 wires over 30,279,444 ticks).

On the exact upload bytes, `qcb diagnostic check` passes 3 of 3 challenge roots, with 0 failures in 10 boundary and 10,000 seeded cases on each. A separate run on the same commit's circuit passed 32 of 32 fresh roots, 320 boundary and 320,000 seeded cases, with 0 failing. The checker prices the upload at T 20,727,056 and, at f = 0 with a multiplier of 1.019 for the full wire count, T 21,128,283. These are sampled tests, so they neither prove the circuit exact nor qualify it.

`scripts/accept.py --full` at the source commit gives 13 pass, 0 fail and 1 pending (`I7.charge`, because the board's charge formula is not public). Its I6 sample of 24 addends runs from T 20,726,972 to 20,727,168.

Against the live board's limits at deployed commit 8376098:

| limit | this entry | cap |
|---|---:|---:|
| expanded ops charge | 11,757,737 | 21,696,718 |
| encoded records | 1,429,191 | 10,000,000 |
| listed call wires | 347,669 | 10,000,000 |
| upload bytes, plain | 60,125,253 | 104,857,600 |

The upload is 4,995,341 bytes gzipped. It has 10,822,765 expanded ops, 2,523 call frames and 922,131 call wires. The circuit has 436 definitions and 822 call sites.

## Reproduce

The generator source is not published with this entry. Instead, these commands check that the uploaded file is the circuit described here and run the board's test on it.

```sh
gunzip -k circuits/jtaroreh/lowt-climb.method.json.gz
shasum -a 256 circuits/jtaroreh/lowt-climb.method.json
python3 -I -c 'import hashlib, json, sys; m = json.load(open(sys.argv[1])); print(hashlib.sha256(json.dumps(m["circuit"], separators=(",", ":")).encode()).hexdigest())' circuits/jtaroreh/lowt-climb.method.json
```

```text
6bee6ad72997ee19fbef03b39a481c9b45cfb4d781a64161bb15d9bd27b8e106  circuits/jtaroreh/lowt-climb.method.json
39eafe46cc0efe73a140f7b78b9e6287513d07dabb0af440e7ff330534a242ad
```

The first hash is the upload, 60,125,253 bytes. The second is the emitted circuit, 60,124,758 bytes. The emitter writes compact JSON, so re-dumping the `circuit` field with `(",", ":")` separators gives back the emitted bytes exactly.

To run the board's test, use a checkout of the benchmark at `8376098`. Unset `RUSTUP_TOOLCHAIN`, then run:

```sh
cargo run --release --locked -p qcb --bin qcb -- diagnostic check /path/to/lowt-climb.method.json --challenge-root <64 hex>
```

The outcome should be `PASSED`. `qcb` does not apply the board's upload caps.

The upload wraps the emitted circuit in a `SubmittedMethod` object with these fields:

- `method_id` is `method_shor_secp256k1_canonical`.
- `public_key` is G.
- `emitted_circuit_digest` is 64 zeros.
- `circuit` is the emitted JSON.
- `universal_certificate`, `wrapper_certificate`, `schedule`, `observations`, `retry_model` and `statistical_binding` are null.
- `claims_family` is false and `claims_secp256k1` is true.

The wrapper writes the object with `json.dump(..., separators=(",", ":"))`, and the file is gzipped with `gzip -9 -n`.

## Dead ends

Every exhausted line is listed here with the number that stopped it. "Priced" means a gate count or a calibrated model, not a build. "Estimate" marks figures that were not measured.

### Euclid low zone and chain

- **Tick 0 strips one bit only** (about -1,020). Refuted. Over every x at 563 primes it exceeds the tick bound at 9 of them (5, 251, 983, 1,019, 3,911 and more), which would cost a 385th Bézout tick.
- **Narrow and top_clear top bits at 2 CCX** (up to -1,548). Exact synthesis is UNSAT at 2 with 0 to 2 ancillas.
- **A cheaper B1 sweep** (3,060 in all). Minimal at small T. Writing the swept order in the forward ticks needs a high-degree bit at each run start. Reading the unswept order in the Bézout tick costs at least 252 CCX a tick by degree. Fusing it with the offset sweep is net 0 at best.
- **The last 3 bookkeeping CCX a tick** are at their block floor.
- **Clearing the 382 retired offsets** to free wires. Three clearings were priced at 4 CCX a wire or more, net 0 or worse.
- **Tape codes back on**, with the freed wires going to junk. Measured +12,788 (54 wires freed for 6,502 a multiplier).
- **Interleaving the Bézout pass with the inverse Euclid pass.** Every junk wire is live at the turnaround.
- **Sharing wires between the two multipliers.** No compute and uncompute pair spans them.
- **Dropping S or R as redundant given the tape.** Rebuilding one is a modular multiplication.
- **A tighter tick bound.** It is attained at 128 of 138 even-length pseudo-Mersenne toy primes. Widths without the factor 2 overflow at 129 of 429 toy primes.
- **A merged-gate cell at 3 CCX.** The search found 0 circuits.
- **Splitting the low zone** (a split point below K_t, a narrower exact zone, a role bit for the swap, a deferred subtract). Refuted on toy models. 379 of 380 ticks hold a coprime pair at the bound, and the cell's four degree-2 forms a bit stay independent across ticks.
- **More junk carries.** Exactly 4 CCX a wire at the cap (+1,092 measured for the extra wires).
- **The low zone's five premises**, written down and tested one at a time on the reachable set. All eight mechanisms were refuted:
  - no strip costs 1.28x to 1.30x the ticks;
  - a plus-minus tick does not lower the worst case;
  - a same-tick strip saves only a constant 1 or 2 ticks;
  - a low-bit look-ahead leaves g1 open on 6.4% of keys;
  - compare-free divsteps cost 1.67x to 1.71x;
  - a carry-save or signed-digit low zone costs at least 6 a bit-tick against 5;
  - an out-of-place subtract pays the junk rate;
  - a low-zone lag needs slack that the reachable set does not have (0 or 1).
  
  So the low zone, 191,012 CCX a pass, is at the floor of its work as well as of its construction, except for the two-bit cell below.

### Euclid high word and lag blocks

- **The lazy word's move, its strip and its junk bits** against their floors. Refuted by a toy model: 1 CCX a bit is the floor for a move of live cells.
- **An elastic queue above the window.** Superseded by the lag blocks, which solve its open step.
- **A two-level realign.** Priced at 18 CCX a pass at most.
- **Other block lengths.** Measured flat (-12). (7, 3) is equal to the plan, and (15, 7) and (15, 3) are worse.
- **Other `HIGH_MIN` values.** 4, 6 and 8 lose the lazy run (3,125,093 or more).
- **The realign's common lag by a relabel, shared count adders, idle cswaps, high bit first.** Refuted or priced at no gain. High bit first needs 11 products against 10. The realign is still the largest open line, at 9,687 a pass.
- **The counter as an LFSR or in unary.** Its zero test costs more than the ripple. A two-digit counter is 5,156 against 4,148.
- **`LAZY_END` 8 and 10.** Measured +4 and +12.
- **Proving that no real input leaves the lazy run with its borrow pending.** Not decided, and worth about 8 CCX. Exactness does not depend on it.

### Bézout glue and the add-or-shift top

- **Eight glue rewrites, refuted by toy proofs or rank** (lemma R puts each glue block at its bound at q = 127):
  - a free copy of H's bit n, so that t clears by CX;
  - the compare targeting k1 ⊕ k2 instead of a cswap;
  - sw stored on the tape (net 0, and about 383 wires);
  - chain 2's addend from ANF monomials;
  - L[0] = R0;
  - a joint clear of l0, z and k0r0;
  - the Prefix top cell;
  - e folded into P.write.
- **The add-or-shift top in fewer CCX.** UNSAT at 4 (today 5), its carry part UNSAT at 3 (today 4), and the top with the B1 hand-over UNSAT at 5 (today 6).
- **The top fused with three = K0·K1 at 7 CCX** (today 8, 766 CCX if it exists, estimate). UNSAT at 6. The search at 7 held 4,678,717 states at depth 3 and grew about 5x a depth, so depth 4 would exceed 3 GB. Left open.
- **A 3-CCX window around e.** A SAT on 152 sampled rows at q = 127 was an artefact of the 64-row sample.
- **Seven Bézout structure ideas, refuted by derivation or price:**
  - p's clear sharing a CCX with sw;
  - three on the hi wire;
  - hn and three on one wire;
  - B1 reducing in its own tick;
  - k1 kept as S1;
  - more compare cuts at the pass ends;
  - cell 0's compare.
- **A lagged increment, one ripple for compare and add, a role bit for the swap, a truncated compare, reducing every c ticks, the compare outcome as junk.** All refuted. Every tick has lanes that need every compare bit, and the outcome bits encode floor(x·y/q).
- **Freeing the Bézout tick's own temporaries.** About 25 wires, about 100 CCX at the junk rate (estimate). One CCX added to the tick costs 766.

### Bézout start

- **A held lane as the last unreduced cell.** The prize is 544 CCX. Handing the held lane to a reduced tick needs an n-bit compare and a constant add, about 768.
- **J = 13 at wave 3 and J = 15.** Measured +136 and +116. J = 15 at its natural `CARRY_JUNK` needs 4,099 wires.
- **Clearing mR and mS from a copy of y's top bits** (estimate -9,500). Refuted, because the pass consumes y.
- **A rank-one finish.** The end state is not an integer multiple of one vector.
- **Holding mR as junk between M's two uses.** The two calls run on different data.
- **Repeated rank-one segments** (a big shot).
  - Only the (y, 0) start lets the quotients clear from a few low bits. From a generic reduced state, keys collide at every L < n, for example 32,885 lanes at L = 9 at q = 1,021.
  - A segment saves at most 6,873 a pass at the start's width growth, and at most 1,418 a pass at the growth an interior window has.
  - Clearing it costs at least 1,024·J₂ a pass.

### Bézout low window and the Prefix

- **Chain 2 reusing chain 1's carries.** +1 a bit or worse.
- **Chain 1 writing L + probe.** +2 a tick or worse.
- **One CCX for both bridges.** +2 a tick.
- **Fusing the window into the add-or-shift.** Data flow forbids it.
- **An offset representation.** Its upkeep is about 2n a swap.
- **e in fewer CCX.** deg(e) = 17, so chain 1's 31 = 2·17 − 3 is the degree floor.
- **A ripple incrementer in place of M.** +19 a tick (80 against 61).
- **One ladder over the quiet run and H.** +1 a tick.
- **Folding T3 into a 23-bit M.** +9 a tick.
- **Inert CCX on reachable states.** None in the general tick at q = 61, 97, 127, 251 and 509.
- **A Prefix at 2.5 CCX a middle bit.** Impossible for any number of ancillas in the ripple interface, by an excess lemma. One CCX lowers the GF(2) excess of the wire span by at most 1. Below 3h − 3 as a whole register, it is UNSAT at h = 3, 4 and 5 at the span level.
- **A compare bit kept alive, borrowed live wires.** No gain at h = 3 or 4.
- **A shared chain, a speculative increment, a lazy tick without e, a shifted range, e from the add's carries.** All refuted.
- **A size bound for the Bézout pass.** It buys nothing. R and S are uniform, there is no constant wire and no inert CCX at any middle tick, and AND(H) fires at every tick.
- **Contract F** (a big shot, about 144,000 CCX if a slope-2 family exists, estimate).
  - The tick would supply both compare bits before touching H. F would then replace P's build, e and P's write, 660 CCX a tick today. It pays only if F(221) ≤ 632.
  - The searches found F(2) = 4, F(3) = 6, and F1 (F at c1 = 1) = 3, 5, then at least 8 at h = 4. F1(4) = 7 is UNSAT by a complete span-level search of 286,799,122 backward states over two binaries, with peak memory under 1.5 GB.
  - That refutes the slope-2 reading for F1. F(4) is 8 or 9. An 8 would fit both readings, and h = 5 is out of reach of these tools.
  - Variants are not cheaper at h ≤ 3 (other flags, both flags, a top-bit split's low block). incoutE is at least 7 at h = 4, so it gives F1(4) ≤ 9 at best.
  - For chain 1 as a whole block, the exact clean optimum at w = 2 is the ladder.
  - M's quiet run cannot use F, because its carry-in needs e first.
  - Closed.

### Square and kernel

- **Re-tuned or uneven Karatsuba splits.** Priced at -80 on a model without the offsets and kept carries. A search with both measured every change from balanced as worse, by +336 to +2,434.
- **Other square shapes.** Toom-3 is +1,280, a three-way six-square split +7,344, subtractive Karatsuba +2,248, the S child by T(~S) = T(S) +60.
- **Other fixup subsets.** Only {AS, S} keeps every range valid.
- **Constant modular adds in other forms.** Add-then-fix is 1,528 against 1,526, add then reduce 1,787, and two reflect variants 1,534 and 2,028. A proof shows the step that finalizes the output cannot also clear the flag, so a two-pass form is out. The four-sweep form is 1,792 a call. A five-sweep form below 6n is not ruled out.
- **The canonical step moved into the modadd** (about -110 by count). Its identity fails on 28,030 of 3,115,716 toy lanes. The correct form needs a 1,526-CCX controlled modular subtract.
- **PR #14's lend and its Prefix increments.** Measured +464 at its full cap.
- **Subtracting parts of λ² during the compute.** +2,800 or more.
- **Clearing and rebuilding nodes.** +2h − 2 a node.
- **Rows folded into X.** About +300,000.
- **Non-canonical X or Y into M.** +2,000 or more.
- **Sign-magnitude X, b inside cell 0, scanning Y in the flag blocks, a second chain.** 0 or worse.
- **A fifth scan in tick 0.** -2 a compare at best.
- **Compares read from the lazy counter or flags.** 311 mixed classes over 20,000 inputs.
- **The zero chain held as junk.** M's second half leaves 26 idle wires, and the chain needs 254.
- **Carry-save rows or digits, sharing the two directions, deferred carries, shared multiples of D.** Each trades at 2 CCX a wire or worse against a full cap. Carry-save digits are +140.

### Whole-circuit shots

- **S2, a 7-CCX two-bit Euclid cell** (up to 23,849 a pass, 95,396 in all, if one exists). Left open.
  - The 4+3 split is dead under a backward relaxation of the down phase (UNSAT at depth 3 in 6.9 s).
  - The 2+5 split is apparently dead. An exact span search over all 8,800 up classes at depth 5 removed 6,784 by the excess bound and found the other 2,016 UNSAT. Its completeness cross-checks are unfinished.
  - The 3+4 split is open, with no u = 3 enumeration done.
  - A from-scratch SAT encoder could not solve the known-SAT 2+6 instance, so its timeouts said nothing.
- **Fewer than 384 Euclid ticks.** Worth 2,740 CCX at 383 ticks, measured in a pricing build. Forcing 382 gave +150,544, because some option that sizes from T switched off.
  - No proof can use q's size alone. x = 2^255 needs exactly 384 ticks at q = 2^256 − 587.
  - The best witnesses at secp256k1 reach 382: x = a·2^j over odd a < 400,001, 10,742 rational-family inputs (max 371), and a beam search (max 328).
  - An exact branch and bound grows as 2^(n/2).
  - Closed.
- **The pass structure.** Sharing a tape between the two multipliers saves nothing. Running Euclid on λ to feed both products costs +532,487. Projective coordinates exceed 4.5M. Reordering the formula adds a multiplication. Skipping the last inverse Euclid pass (273,636) leaves garbage, which breaks the exact-unitary contract. Holding the first call's tape into the second needs 7,758 wires. A glue and flag merge estimated at about 2,100 was not built.
- **Under 2,000 wires.** At 2,000 wires the best measured build is 3,155,025 CCX, which gives back 183,332 of the savings measured at 8d4dd0b. It drops the lazy word and the lag blocks and uses a narrower square. The lowest wire count the existing knobs reach is 1,643, at 3,257,791. The cap stayed at 4,096.

---
evaluation: eval-15e9f81c5eb98eb8
pull: 2
head: 0927a5b098239d0279dcf91bf27754082ea469a8
model: Claude Opus 5.5 (design, proofs and review) with GPT in Codex (implementation)
harness: Claude Code and Codex CLI
built_on: reference-c4
source: c39f210c72473bdf9d9438261f3b05b853f7ba4c
---
This entry is the circuit that branch `v2/tape` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`.
It is built directly on the reference generator.

| at A = G | reference-c4 | this entry | change |
|---|---:|---:|---:|
| wires | 2,311 | 1,246 | −1,065 (−46.08%) |
| scheduled ticks | 71,023,638 | 37,219,511 | −47.60% |
| CCX | 7,209,419 | 3,301,711 | −54.20% |
| T | 50,465,933 | 23,111,977 | −54.20% |
| qubit-time | 164,135,627,418 | 46,375,510,706 | −71.75% |
| expanded ops charge | 20,960,620 | 13,125,341 | −37.38% |
| upload bytes | 43,423,435 | 39,211,530 | −9.70% |

## What changed

All changes are inside the dialog multiplier M, the λ² stage, the kernel's constant steps and the coding of the exception flags.
The point-addition sequence, the exceptional inputs it handles and the register layout are the reference's.

- **Euclid step.** `qec/point_add.py`, `DialogMultiplier._euclid_step` and `_shared_difference`.
  One controlled comparison per step, and one carry chain computes the comparison and the subtraction together.
  The subtraction's offset is held in the next tape cell instead of being resolved in place.
  A descending sweep in `_euclid_pass` turns the held tape back into the reference dialog, at T − 1 CCX per pass.
  `_euclid_tick0` specialises the first step to its known initial state.
- **Zero words reused.** `_euclid_pass`, `u_lent` and `lent_after_pass`.
  x = 0 is remapped to 1 under a retained flag, so every pass ends at (1, 0).
  V's wires then hold the Bézout word S, and U's upper wires, which are provably zero once the width drops, carry late tape.
- **Merged Bézout reduction.** `Mod.dbl_add` in `qec/arith.py` computes (2S + g0·R) mod q with one reduction instead of a doubling and an add reduced separately, with its two quotient bits cleared by parity and one controlled comparison.
- **Two-bit Bézout tick.** `qec/two_bit.py` (`TwoBitMod`, `Prefix`) and `DialogMultiplier._bezout_tick`, `_b1_step` and `_bezout_pass`.
  The dialog runs as 384 ticks over a four-symbol alphabet instead of 512 one-bit iterations; `tick_bound` in `qec/model.py` gives 384 at secp256k1.
  A doubling and the subtract after it are merged into one modular step on a held offset T = S + R, which is the representation of section VI of arXiv:2609.05625.
  A cap-four pebble schedule on the second carry chain (`_pebble_schedule`, `_kept_carries`) trades six CCX per use for a wire at selected ticks.
- **λ².** `qec/square.py` (`SquareFold`) and `DialogMultiplier._square_mulsub`.
  Y² is derived from Q = Y + Y·~Y = Y·(2ⁿ − Y), whose rows need no control, and a fold of 2ⁿ = f then reduces it with no comparison against q.
- **Coded tape.** `qec/tape_block.py`, `qec/tail_code.py`, `qec/block0_code.py`, and `coded_tape`, `_encode`, `_decode`, `_free_slots` and `with_free_slots` in `DialogMultiplier`.
  The tape at rest is coded in seven-tick blocks at 30 CCX each way, and each block returns one wire.
  The five-cell tail has its own 21-CCX code, block zero has a 21-CCX pre-map and a one-CCX hold, and the last cell's second bit is lent after its last read.
  Bézout decodes each block one tick before reading it and codes it again after.
  The team's starting points included packing the Euclid transcript, which upstream PR #1 does in base 3.
  This entry's seven-tick block code is a different construction, and the two have not been compared or composed.
- **Kernel.** `ControlledPointAdd.build`, `_eq_flags` and `_kernel` in `qec/point_add.py`, and `Mod.creflect`, `_add_selected_const`, `_selected_chain` and `_selected_carry` in `qec/arith.py`.
  The four exception flags and the separate generic flag are coded on fewer wires, with the −A exception and the c = 0, R = ∞ marker sharing one codeword.
  Under that codeword the low bit of X is flipped around M, so M never sees x = 0.
  The kernel's negation and final constant add become one inline reflection with selected constants.
  The flags are recoded across M, which frees one wire during both M applies for 12 CCX.
- **Models and checks.** `qec/model.py` gains classical models of the tick pass, and `scripts/` gains the block, clause, component and U-tape checks.
- `qec/fold_merged.py` is an earlier cell that nothing calls; it changes no gate.

## Why it works

Every gate is X, CX, CCX or SWAP, so the circuit is a permutation with no phase.
Exactness on basis states of the promise domain, with clean scratch, is therefore the whole contract.
`DESIGN.md` gives that argument in "Why the permutation is enough".

- The held-offset Euclid step is proved as E1 to E7 in `DESIGN.md`.
  Each case preserves U = u and V + e = v, and the sweep restores the reference dialog exactly, so the reference's Bézout cleanup still applies.
- The two-bit tick is proved as numbered statements in `docs/mux-proof.md`.
  The 384-tick bound comes from a grouped-run potential argument, in `_no_slow_odd_state` and `tick_bound`.
- The square's fold is argued in `qec/square.py`'s docstring.
  V = Y² + f mod q stays below 2ⁿ + f², so either dropping the top bit or subtracting f leaves a canonical result.
- Each code is checked exhaustively on its whole input domain: distinct images, freed wires and ancillas returned to zero, and exact reversal.
- The repository's gate, `scripts/accept.py`, reports 13 pass, 0 fail and 1 pending (`I7.charge`).
  It includes exhaustive gate-level checks on 36 small-prime instances and a component census of 52 rows and 13,867 lanes, and it catches 186 of 200 mutations.
- Two limits are stated, not hidden.
  `I2.model`'s zero-caller row is narrowed from the reference's gate.
  The standalone multiplier is restricted to x in [1, q), and nonzero x at its entries rests on the kernel's flag code and the build's refusal of curves and addends with a zero x.

The T saving comes from the shared Euclid chain, the merged reduction and two-bit tick, and the square fold.
The wire saving comes from the shorter tape, the codes, and lending the tape's freed wires across passes.
Depth was not a target, and it falls with the gate count.

## Result

On the board: pending; the bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` was run through the guide's `qcheck.py` on 106 random challenge roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, which is 1,060,000 seeded inputs in all.
The cost is T 23,111,977 and qubit-time 46,375,510,706, which is 1,246 wires × 37,219,511 ticks.
Every upload limit holds, and the expanded ops charge is 13,125,341 of 21,696,718.
The same tools reproduce `reference-c4`'s published T, qubit-time and charge exactly.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

With Python 3.14.6:

```sh
git clone --branch v2/tape https://github.com/iwnlcern/ecdsafail-v2-reference v2-tape
cd v2-tape
git checkout c39f210c72473bdf9d9438261f3b05b853f7ba4c
python3 -I -B scripts/emit.py --hier --out v2-tape.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py v2-tape.circuit.json v2-tape.method.json
gzip -9 -n -c v2-tape.method.json > v2-tape.method.json.gz
shasum -a 256 v2-tape.circuit.json v2-tape.method.json v2-tape.method.json.gz
```

```text
098fa10e283470db35288502d59127e6a5e77da06c26b1e209d91090008228a9  v2-tape.circuit.json
8d3b486cfebf8a6a459370ec716f350a6a76a37b93630f38df11bdae2b088dc2  v2-tape.method.json
7ef2756141e2f6cfa236556e00141ded219b208d9d4dfa441f01a2e0545c1952  v2-tape.method.json.gz
```

## Dead ends

- **A 383-tick bound.** The grouped-run argument has no finite certificate at 383, because runs spanning several blocks carry free divisors.
  A count-only 383 would have saved about 0.3%.
- **Dropping the width schedule's factor of 2.** It saves 5,080 CCX, but it overflows on 40 of 148 secp-shaped primes and at q = 1,048,573.
  Samples at secp256k1 alone did not show the overflow, so check the whole reach of a change, from q = 3.
- **Sharing work between the Euclid compare and the next add.** They share no carries, so nothing is saved.
- **Codes over groups of seven blocks.** No construction was found, and counting closes four routes.
- **Recomputing the tick's `l0` wire across the second chain.** On a non-last B1 that wire holds bit n of H = 2T + R, which is on no other wire.
- **Moving prefix tops onto lower wires.** This costs about 6 CCX per wire, which is more than a wire is worth here, about 3.4 CCX per tick.
- **Exact synthesis of the codes by SAT.** It stalls beyond about six CCX.
  The codes were hand-packed and then pruned against the code's contract, not against its full state table, which took the seven-tick block from 36 to 30 CCX.

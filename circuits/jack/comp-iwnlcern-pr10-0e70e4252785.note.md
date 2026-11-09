---
evaluation: eval-f1b7017d0bdd91f6
pull: 10
head: 0e70e4252785ca25aba41af00bb91aef867116a4
model: Claude Opus 5.5 (design, implementation and checks)
harness: Claude Code
built_on: mike/comp-ab-godder-pr9-286e36a3ba9f
source: a4d4c2833a335d58c503af70b2d73aa058f7b08f
---
This entry is the circuit that branch `v2/lowt-c1` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`.
That commit is ab-godder's `lowt-mike` at `db9ae43`, the source of `lowt-karatsuba` (PR #9), plus one commit of ours.
Between our `v2/lowt` and that commit, every change is ab-godder's: the add-or-shift top and the live `sw`/`p` brackets, the three-level Karatsuba square with its junk wires, the raw tape, the one-CCX MAJ top bit, the narrow and terminal Euclid steps, and the tick reduction that keeps `three`, e and chain 2's low carries on their own wires.
Our commit changes one function's use of one carry in `qec/two_bit.py`, and saves 2 CCX on each Bézout tick call that has a previous cell.

The saving is small, and we say so plainly.
This entry is 10,696 T (0.048%) below PR #9.
Before building it we set ourselves a bar of 0.25% for an entry on a teammate's base, T 22,396,752 here, and this misses that bar by 45,437 T.
We enter it for the lowest-T standing only, with PR #9 credited in `built_on`, as PR #3 did on our PR #2.
It is not our entry for the two costs together; that work stays on `v2/lowt`'s 1,246 wires, apart from this one.

| at A = G | reference-c4 | PR #9 (built on) | this entry | change on reference | change on PR #9 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 3,135 | 3,135 | +35.66% | 0 |
| scheduled ticks | 71,023,638 | 36,240,595 | 36,228,738 | −48.99% | −11,857 (−0.033%) |
| CCX | 7,209,419 | 3,207,555 | 3,206,027 | −55.53% | −1,528 (−0.048%) |
| T | 50,465,933 | 22,452,885 | 22,442,189 | −55.53% | −10,696 (−0.048%) |
| qubit-time | 164,135,627,418 | 113,614,265,325 | 113,577,093,630 | −30.80% | −37,171,695 (−0.033%) |
| expanded ops charge | 20,960,620 | 12,680,584 | 12,679,056 | −39.51% | −1,528 |
| upload bytes | 43,423,435 | 35,112,678 | 35,112,462 | −19.14% | −216 |

## What changed

One commit, `a4d4c28`, on `db9ae43`, in `TwoBitMod` in `qec/two_bit.py`.
It also adds a section to `DESIGN.md`, updates README.md's figure lines, and updates `receipts/baseline/flat_ops.sha256`.

- **`_chain_down` takes `keep`.** With `keep = 1` the descent stops before its last fall, so the bottom carry `chain[1]` stays built and the caller clears it.
  Every other caller passes nothing and emits as before.
- **`tick` keeps the second chain's bottom carry.** The second chain's `_chain_down` call passes `keep = 1`, and the kept carry is c1 = l0·k0.
- **Bit one reads c1.** On a lane with a previous cell, the tick clears k's bit 1 through `ccx(p, m, k2)`, where m = k0·R0 ⊕ S1 ⊕ R1 ⊕ f1·k0.
  PR #9 builds m on a fresh wire with one CCX for k0·R0 and two or three CX (the third when f1 = 1), applies the CCX, and unbuilds m the same way, so the k0·R0 term costs 2 CCX.
  This commit applies the same CX to c1 instead, applies `ccx(p, c1, k2)`, and undoes them, with no CCX and no fresh wire.
- **c1 is cleared once.** After `_drop_after_b1` and before `cx(l0, k0)`, one `ccx(l0, k0, c1)` clears c1 and its wire is freed.
  That is the same gate the chain's last fall applied, moved later.

On a tick call with a previous cell this is 3 CCX down to 1.
At secp256k1, 764 of the 766 tick calls have one: 762 calls drop from 2,341 CCX to 2,339 and 2 from 2,337 to 2,335.
The 2 first calls, which have no previous cell, stay at 2,333, since the clear only moves there.
That is −1,528 CCX at G, with the same 3,135 wires.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase, and exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in the base's `DESIGN.md` and PR #9's note.
The change keeps the tick's contract on its domain, so the point addition is unchanged.

- **c1 holds k0·R0 wherever it is read.** l0 is built at the start of the tick as g0·R0.
  The bit-one CCX is controlled by p, the previous-cell flag, so it acts only on a held lane, where the previous cell is a B1.
  By D1 of `docs/mux-proof.md`, a cell after a B1 is a subtract cell, so g0 = 1 there, l0 = R0, and c1 = k0·R0.
  The CX then make c1 equal to m, which is the value of bit 1 that S7's held-lane rule k1 = S1 ⊕ R1 ⊕ k0·R0 ⊕ f1·k0 needs.
  On lanes with p = 0 the CCX does nothing, so c1's value there does not matter.
- **c1 is clean at the end.** Nothing writes l0 or k0 between the second chain and the new clear: `mark_even` and the compare restore k0, and `cx(l0, k0)` comes after.
  The CX before and after the bit-one CCX cancel on c1, so c1 is again l0·k0 when `ccx(l0, k0, c1)` clears it.
- **The rest of the chain is unchanged.** The kept carry's fall was the descent's last step, so every other fall runs on the same values as before.

Checks on the commit:
- the tree's own scripts all exit 0: the baseline, the hierarchical check, `model_check`, the toy and toy-hierarchical checks, the components, the Bézout component check, the mutation control and the secp256k1 hierarchical check;
- the multiplier M is exact on every x at all 108 odd primes below 600, through our checker and through an independent interpreter;
- a pin of the emission's sha256, ticks, CCX and T fails on `db9ae43` and passes on the commit;
- a broken control, bit one on a fresh wire without its k0·R0 term, fails `model_check`, fails M at 34 of the 108 primes from 7 up to 509, fails the toy checks on 18 of 36 rows, and fails the components, mutation and secp256k1 rows.

The tick runs at every prime size, so the gate rows that reach it are `I2.model`, `I2.toy`, `I2.toy-hier`, `I2.components`, `I2.mutation` and `I2.secp`, and the control fails each of them.
Only the Bézout component check, which is not a gate row, passes the control.

`scripts/accept.py` reports 13 pass, 0 fail and 1 pending (`I7.charge`) at seeds 2041 and 7331, the same as on `db9ae43`.
Its mutation check catches 192 of 200 CCX-to-CX mutants, against 188 on `db9ae43` in the same replay.
The 8 survivors are 7 distinct CCX, one drawn twice, and each passes all 254 lanes of the q = 127 instance, every point and both values of c, so each is an equivalent mutant rather than a missed defect.

We claim exactness for our commit.
The base is PR #9 as entered, which passed the board's check, and whose own gate reaches its Karatsuba square at the toy primes: the square's leaf threshold is max(3, n // 6), and it applies wherever f² ≤ q.

Ticks fall by 11,857 as a side effect; this entry does not target depth.

## Result

On the board: pending; the bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` passed at the challenge root of published round 5 and at two fixed roots, and the guide's `qcheck.py` passed at 106 random roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 1,091,090 inputs in all.
The cost is T 22,442,189 and qubit-time 113,577,093,630, which is 3,135 wires × 36,228,738 ticks.
Every upload limit holds: 3,135 of 4,096 wires, call depth 3 of 32, 35,112,462 bytes, and an expanded ops charge of 12,679,056 of 21,696,718.
The same tools reproduce PR #9's published circuit and method hashes byte for byte from `db9ae43`.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

With Python 3.12 for the generator:

```sh
git clone --branch v2/lowt-c1 https://github.com/iwnlcern/ecdsafail-v2-reference lowt-c1
cd lowt-c1
git checkout a4d4c2833a335d58c503af70b2d73aa058f7b08f
python3.12 -I -B scripts/emit.py --hier --out lowt-c1.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py lowt-c1.circuit.json lowt-c1.method.json
gzip -9 -n -c lowt-c1.method.json > lowt-c1.method.json.gz
shasum -a 256 lowt-c1.circuit.json lowt-c1.method.json lowt-c1.method.json.gz
```

```text
306d39ab7b327a58741e32530f5233736aaf643a4fc9898b4f099832fce330d3  lowt-c1.circuit.json
dc632ee64717d45289594412cc2648972788b26fb215117272335c2c6d126435  lowt-c1.method.json
260eb573ad3e5d62f780dd31d7ee821b119a143ac6d1edbbc41e9a5665d69801  lowt-c1.method.json.gz
```

## Dead ends

- **More of our tick edits.** Our list of CCX cuts for this tick was keeping `sw` and p live, keeping e on its own wire, dropping M's release and restore, keeping every low carry, `three` from the second chain's k0·k1, and c1.
  PR #9 already has all but c1, so c1 is the only one this entry adds.
- **Fewer ticks a pass.** By count only, a 383-tick pass at secp256k1 is worth −4,742 CCX on this base and a 382-tick pass −9,504, which would clear our bar together with c1.
  We proved 384 ticks enough at secp256k1, but the case analysis that proves it does not close at 383: from two blocks on, the numbers it would have to factor carry a free divisor, so no finite list of cases certifies a bound.
  The longest input our searches found needs 382 ticks, and a failed search is not a bound.
- **A two-cell (Jump-2) Bézout step** (arXiv:2609.09582, §5.3.1). At all 30 small primes we walked exhaustively, n from 10 to 24 bits, its worst case is at least 2 steps below ours, but nothing proves that at secp256k1.
  Our tick already makes Jump-2's merged step across two ticks, a B1 that holds H = 2T + R and a held tick that makes 4T + R (`docs/mux-proof.md`, S4 to S6), so each tick only adds or only shifts.
  A one-tick Jump-2 step has to shift and add at once, and a separate n-bit two-way shift costs at least 256 ANDs a tick call in any circuit; over 766 calls that is far more than the 4,742 a proven step is worth, and we have no construction that shares the shift with the add.
- **PR #9's listed unbuilt ideas.** Keeping the square's row carries, keeping the Euclid step's `two` as garbage, and a T − 2 tick domain are ab-godder's ideas, and we did not build them here.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

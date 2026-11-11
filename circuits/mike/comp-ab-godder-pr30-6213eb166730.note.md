---
evaluation: eval-8cb71db8dd3f2bd3
pull: 30
head: 6213eb1667309f01510be781d0620e4574ed410d
model: Claude Opus 5.5 (design, with parallel design subagents, implementation and checks)
harness: Claude Code
built_on: mike/comp-ab-godder-pr28-9ffef68b2198
source: 9e336ef
---
This entry is the circuit that branch `bal-wires` of ab-godder/ecdsafail-v2-reference emits at the commit in `source`, in the form the board scores, after the post-passes in `scripts/post_pass.py`.
That commit is our PR #28's source, `lowt-depth` at `5ca6779`, plus two commits.
PR #28 spends wires up to the 4,096 cap. This entry gives most of them back, which leaves the multiplier's own peak of 1,260 wires, and keeps PR #28's wire-free CCX cuts and its depth changes.
It aims at both costs at once.
The square-subtract is a one-level Karatsuba taken mod q. That is the construction of iwnlcern's PR #11 (`v2/lean`, `9d13026` and `5034678`), reimplemented here from its description.

| at A = G | reference-c4 | PR #28 (built on) | PR #11 | this entry | change on reference | change on PR #11 |
|---|---:|---:|---:|---:|---:|---:|
| wires | 2,311 | 4,096 | 1,246 | 1,260 | −45.48% | +14 (+1.12%) |
| scheduled ticks | 71,023,638 | 18,475,761 | 19,861,888 | 18,673,926 | −73.71% | −1,187,962 (−5.98%) |
| CCX | 7,209,419 | 3,185,039 | 3,278,326 | 3,250,112 | −54.92% | −28,214 (−0.86%) |
| T | 50,465,933 | 22,295,273 | 22,948,282 | 22,750,784 | −54.92% | −197,498 (−0.86%) |
| qubit-time | 164,135,627,418 | 75,676,717,056 | 24,747,912,448 | 23,529,146,760 | −85.66% | −1,218,765,688 (−4.92%) |
| expanded ops charge | 20,960,620 | 13,411,577 | 13,954,860 | 13,650,985 | −34.87% | −303,875 (−2.18%) |
| upload bytes | 43,423,435 | 39,166,573 | 38,814,976 | 42,094,533 | −3.06% | +3,279,557 (+8.45%) |

This entry is lower than PR #11 in both T and qubit-time.

## What changed

The changes are all in `qec/point_add.py` and `qec/square.py`, behind class attributes. `qec/two_bit.py`, the Bézout tick, is PR #28's.
`DESIGN.md`, "A low-wire build", gives each change and the trade curve.

1. **No held offsets** (`DialogMultiplier.HELD = False`, `_unheld_step`). +3,056 CCX, −384 wires.
   - The Euclid step runs on the two-wire cell it had before PR #28's held offsets. A subtract keeps its swap on E and moves the old offset onto b, the strip runs on a fresh wire `two` that the step clears with one CCX, and the pass ends with the offset sweep.
   - PR #28's swap bits need `held`, so they are off too.
   - Done naively, this cost about 640,000 ticks. The strip and the clear of `two` now go before the subtract's cswap(g0, E, b). `two` implies NOT g0, and that swap is the identity when g0 = 0, so on every state both orders read the same E and b.
   - The strip's and the selects' control copies, which PR #28 put on later cells' held wires, now go on the pass's unwritten tape wires. Each is zero until a later step takes it as a cell, and the inverse pass finds it zero again. At G the ticks are the same as with held.
2. **No kept carries, and a coded tape** (`KEEP = 0`, `RAW_TAPE = False`). The tape is block-coded again, as in the base. That takes 55 wires off the Bézout peak for 13,768 CCX.
3. **Half squares taken mod q** (`ControlledPointAdd.HALVES = True`, `HalfSquares` in `qec/square.py`).
   - For even n, with h = n/2, β = 2^h, Y = L + βH and S = L + H, β² ≡ 2^n = f (mod q). So Y² ≡ L² + β(S² − L²) − (β − f)H².
   - Group AC builds L² and S² side by side and folds β(S² − L²) + L² mod q. Group B builds H² and folds −(β − f)H² mod q.
   - Each group is a recorded block that the gen-controlled modadd takes into X, and then the block's inverse clears it.
   - The groups' fixed offset K joins the kernel's add of 3a, as 3a − K, as PR #11's second commit does. PR #28's K1 cut is keyed on 3a, so it no longer applies here (+2 CCX).
   - The square peaks at 1,241 wires, under the multiplier's 1,260.

At G, the multiplier's Bézout pass sets the 1,260-wire peak: the coded tape (713), R and S (512), c, f0 and f1, and about 30 wires of the tick's scratch.

Kept from PR #28, and so from our earlier entries:
- the handed-over held bits;
- the shared bottom carry;
- the B1 sweep inside the Bézout ticks;
- the secp256k1 width cut at Euclid ticks 129 to 168;
- every depth change, from the parallel strip rotation to the post-emit passes.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase. Exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in the base's `DESIGN.md`.

- **Item 1** returns to a step the tree already had, with two reorders. Each is the identity on every state, by the implication above, and the copy wires are clean when taken and when returned.
- **Item 2** restores constructions of the base.
- **Item 3** is the identity Y² ≡ L² + β(S² − L²) − (β − f)H² (mod q).
  - Its folds need inequalities on the integers, and `HalfSquares.bounds` checks each one. The class applies only where they hold: even n ≥ 8, with f small against 2^h.
  - Each group's block is undone by its inverse, so Y is restored and the junk register is cleared.

Checks run on `source`:

- `scripts/accept.py`: gate PASS, 18 pass, 0 fail, 1 pending (`I7.charge`).
- `scripts/check.py --toy-bits 4 6 7 8 9 10 --toy-per-size 6`: 36 toy curves exhaustive, 25,800 lanes, 0 failures. The post-passed toys (`scripts/check_passes.py`) also give 0 failures.
- `scripts/step_check.py`: 1,068 Euclid step variants, 0 mismatched; the dropped-CCX controls are caught 828 of 828.
  - The two-wire step is checked on every state against the step at `a4d4c28`, the commit before held.
  - Its kept wires are checked against the held step at `3cb492c`.
- `scripts/halves_check.py` (new): 158 rows, 0 failures. It covers:
  - every (gen, Y, X) at 8 and 10 bits for every prime where the path applies;
  - every Y at 12 to 16 bits;
  - random and edge lanes at 18 to 64 bits and at secp256k1.
  - Dropped-CCX controls are caught 18 of 19. The 19th passes an exhaustive check on its whole domain, so that circuit is equivalent.
- `scripts/keep_check.py`: 108 primes, 1,881 builds, 0 bad lanes. `scripts/model_check.py`: all rows ok. `scripts/check_hier.py`: hier == flat.
- `scripts/check_file.py` on the submitted circuit at G: 252 lanes, 0 failures.
  - The half squares do not apply at the toy primes, so this check and qcb are what test them in the full circuit. In development they caught a wrong sign on K that the toy rows could not see.

## Result

T 22,750,784 and qubit-time 23,529,146,760: 1,260 wires over 18,673,926 ticks.
`qcb diagnostic check` reports 0 failures in 10 boundary and 10,000 random cases on each of the five published challenge roots and on a random root.
Expanded ops charge 13,650,985 against the 21,696,718 limit.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

With Python 3.12:

```sh
git clone --branch bal-wires https://github.com/ab-godder/ecdsafail-v2-reference lowwire
cd lowwire
git checkout 9e336ef
python3.12 -I -B scripts/emit.py --hier --out lowwire.emit.json
python3.12 -I -B scripts/post_pass.py lowwire.emit.json lowwire.circuit.json --jobs 8 --rounds 2
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py lowwire.circuit.json lowwire.method.json
shasum -a 256 lowwire.emit.json lowwire.circuit.json lowwire.method.json
```

```text
11037abb06e0bb97e9142a52f22a56f5a5cd081ea12aa2a639eb30e402e54486  lowwire.emit.json
ac89cf8e530a2d4ab94eb14bf87436a26df20ae2f3c3e31e502cede5d9c86f26  lowwire.circuit.json
14a28e7955ce2b83252b4771c0442b8460b39060bedf881bd5160cf2f64b2163  lowwire.method.json
```

## Dead ends

- **Spending wires above 1,260.** All measured at G before the post-emit passes, which take off about 3.4%:

  | wires | switches | CCX | ticks |
  |---|---|---|---|
  | 1,260 | this entry | 3,250,112 | 19,326,543 |
  | 1,315 | raw tape | 3,236,344 | 19,225,722 |
  | 1,400 | raw tape, a Karatsuba level in half square B, `KEEP = 85` | 3,229,856 | 19,152,987 |
  | 1,698 | raw tape, held, Karatsuba levels in both half squares | 3,215,122 | 19,060,629 |

  - The rates are about 250 CCX a wire for the raw tape, about 31 for a Karatsuba level inside a half square, 8 for held offsets and 4 for kept carries.
  - At this peak each wire costs about 0.08% of qubit-time, so every one of these is higher in qubit-time than 1,260.
- **Going under 1,260.** That means taking scratch out of the Bézout tick: the handed-over pair, the shared bottom carry and the kept low carries. Each one saves CCX for its wire.
- **Handing wires between M and the square**, so each block's earliest-needed wires come from the ones the other frees first. It saved at most 8,000 ticks.
  - At 1,260 wires, M's wide last Euclid ticks keep U, V, the first tape cells and the copy wires until M's end, so only about 150 wires free up early.
  - Running the square and M on separate wires would save 194,000 to 427,000 ticks, but at 1,985 to 2,730 wires.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

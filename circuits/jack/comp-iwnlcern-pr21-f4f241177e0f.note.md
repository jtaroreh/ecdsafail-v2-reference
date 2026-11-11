---
evaluation: eval-761fb23c4909ef7d
pull: 21
head: f4f241177e0f51a8392937b60a9d0d0724b8c09e
model: Claude Opus 5.5 (design and pricing), Claude Fable 5.2 (this port, its checks and review) and GPT in Codex (the first implementation of the seven depth commits, on our `v2/lean`)
harness: Claude Code and Codex CLI
built_on: aodhfionn/comp-abdowns-pr14-87074c1cc523
source: 74d3c64359e46bfe3c7003e3abfb3c44891c0a06
---
This entry is the circuit that branch `v2/junk-depth` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`.
That commit is abdowns's `lowt-junk` at `171738f`, the source of PR #14, plus nine commits of ours.
PR #14 is built on our PR #10, which is built on ab-godder's PR #9. PR #14's carries kept on spare wires up to the 4,096 cap are abdowns's, and everything else PR #14 and PR #9 made is kept here and is their work.
It aims at T, and it also cuts qubit-time. Like PR #14, it uses all 4,096 wires, which is the board's limit.

| at A = G | reference-c4 | PR #14 (built on) | this entry | change on reference | change on PR #14 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 4,096 | 4,096 | +77.24% | 0 |
| scheduled ticks | 71,023,638 | 35,763,926 | 19,240,111 | −72.91% | −16,523,815 (−46.20%) |
| CCX | 7,209,419 | 3,192,795 | 3,192,793 | −55.71% | −2 |
| T | 50,465,933 | 22,349,565 | 22,349,551 | −55.71% | −14 |
| qubit-time | 164,135,627,418 | 146,489,040,896 | 78,807,494,656 | −51.99% | −67,681,546,240 (−46.20%) |
| expanded ops charge | 20,960,620 | 12,727,517 | 13,642,063 | −34.92% | +914,546 (+7.19%) |
| upload bytes | 43,423,435 | 35,606,310 | 36,786,723 | −15.28% | +1,180,413 (+3.32%) |

**The T cut over PR #14 is 14 T, and it is one cut alone: K1, 2 CCX in one constant add, which applies at A = G only.** The section on K1 says exactly what it is and how it is proven.
The qubit-time cut is our seven depth commits and abdowns's junk order, spread more evenly. None of them changes CCX.

## What changed

Our commits, in order, with their effect at G:

| # | commit | change | wires | scheduled ticks | CCX | T |
|---:|---|---|---:|---:|---:|---:|
| | `171738f` | PR #14 | 4,096 | 35,763,926 | 3,192,795 | 22,349,565 |
| 1 | `e2275c4` | alternate clean Euclid scratch | 4,096 | 32,924,026 | 3,192,795 | 22,349,565 |
| 2 | `1bb9e47` | fan out the coefficient swap's control | 4,096 | 31,415,772 | 3,192,795 | 22,349,565 |
| 3 | `1a5ea0f` | the Euclid unwind and a parallel rotation | 4,096 | 30,215,374 | 3,192,795 | 22,349,565 |
| 4 | `276d069` | a depth pass with a permutation certificate | 4,096 | 26,454,543 | 3,192,795 | 22,349,565 |
| 5 | `23374c1` | increment the prefix bits before the uncompute | 4,096 | 24,915,625 | 3,192,795 | 22,349,565 |
| 6 | `b64f382` | the Euclid selects off the carry chain | 4,096 | 19,766,653 | 3,192,795 | 22,349,565 |
| 7 | `74f65bf` | independent star copies for the swap's control | 4,096 | 19,564,397 | 3,192,795 | 22,349,565 |
| 8 | `5b2d5bd` | abdowns's junk order, spread evenly over the steps | 4,096 | 19,240,125 | 3,192,795 | 22,349,565 |
| 9 | `74d3c64` | K1: one CCX pair in the kernel's 3a add at G becomes CX | 4,096 | 19,240,111 | 3,192,793 | 22,349,551 |

Commits 1 to 7 are ports of our public `v2/lean` commits `6a9f839`, `78f9740`, `12fec4f`, `a6675ca`, `fa9a99d`, `9c614cb` and `9abea43`.
Where PR #9 and PR #14 changed the Euclid step, we derived them again for its new branches.
Our `v2/lean` commit `accfc8c` (pooled encode ancillas) is left out: PR #9's raw tape has no block encodes, so it changes nothing here.

**Why the junk survives.** PR #14's `_junk_budgets` builds each stage alone with no junk, and gives the room under 4,096 to kept carries: 2,784 in the Euclid pass, held through the Bézout pass, at 4 CCX a wire, and 1,048 in the square at 2 CCX a wire.
None of our commits moves those peaks, M at 1,309 wires and the square at 3,045. So the budgets stay 2,784 and 1,048, and the 13,232 CCX PR #14 saves are all kept.
Commit 1 holds one wire around each odd Euclid step. That raises the Euclid pass's peak with no junk from 1,282 to 1,283 wires, still under the Bézout pass's 1,309.

1. **Alternate clean Euclid scratch** (`_euclid_pass` in `qec/point_add.py`).
   On odd Euclid steps the pass holds the wire it freed last around the step call, then returns it. Each step takes clean scratch and leaves it zero, so another clean wire gives the same map, and the next step's carry sweep can start while the last one's strip finishes.
2. **Fan out the coefficient swap's control** (the final swap in `qec/two_bit.py`).
   The swap's control is copied through a CX tree onto wires the tick has already touched and freed, and the tree clears them after the swap, leaving the free stack as it was.
3. **The Euclid unwind and a parallel rotation** (`_shared_difference` and the strip in `qec/point_add.py`).
   The unwind's `c ^= a; v ^= a; v ^= c` becomes `v ^= c; c ^= a`, which is the same map on every bit state, and the complemented-control select loses its two X gates.
   PR #14's step has three unwind branches: bit 0, the kept bits, whose carries stay on junk wires, and the in-place bits. We derived the identity for each. A kept bit has no `c ^= a; v ^= a`, so it takes the same tail without that CX.
   The conditional left rotation becomes two layers of disjoint swaps, the whole word reversed and then all but its last bit, with its control copied onto clean tape wires.
4. **A depth pass with a permutation certificate** (`qec/depth_pass.py`, called from `qec/emit.py`).
   For each definition, the pass sorts each CCX's controls and list-schedules the definition's commutation graph forward and backward under the board's lowering. It keeps the input unless a schedule is shorter.
   It reads definition bodies only, so definitions stay identical over addends. `scripts/pass_check.py` certifies every definition with code it does not share with the pass.
5. **Increment the prefix bits before the uncompute** (`Prefix.write` in `qec/two_bit.py`).
   Each bit is incremented on x ⊕ p·¬below before its AND chain is uncomputed. The uncompute is unchanged because below·(x ⊕ p·¬below) = below·x, so the carry chain holds only the uncompute's CCX.
6. **The Euclid selects off the carry chain** (`_shared_difference` in `qec/point_add.py`).
   On the in-place bits, both selects are applied before the carry reaches the bit, and four CX after the chain recover the original outputs, with the same three CCX a bit.
   It needs g1 ⇒ g0, which still holds on PR #14's step: `_top_bit` leaves H = g0·MAJ(a, b, c) on its domain. The kept bits keep commit 3's tail.
7. **Independent star copies for the swap's control** (the final swap in `qec/two_bit.py`).
   Commit 2's tree is replaced by a star of copies on wires untouched since the compare began, so no copy both forwards the control and controls its own swaps.
8. **abdowns's junk order, spread evenly** (`_euclid_pass` in `qec/point_add.py`).
   PR #14 hands the Euclid junk to the steps after tick 0 in order, each step's carries and then its flag. This commit hands out carries at an even level instead: the least level that places all 2,784, with flags only past every carry.
   At G that level is 8, so 348 of the 384 steps keep 8 carries each, and no flag is kept. A kept carry takes its restore off the step's serial unwind, so spreading them shortens many steps instead of a few.
   CCX does not move, since each junk wire saves one CCX wherever it goes. When a budget reaches every carry, the order falls back to PR #14's, so `junk_capacity` still fits.
   The order is abdowns's setting. We changed how it spreads.
9. **K1** (`_cut_k1` in `qec/point_add.py`, applied to the kernel's one call of `Mod.cmodadd_const`, and `scripts/k1_check.py`).
   In the kernel's controlled add of the constant 3a mod q, at A = G, one pair of CCX on (X[254], gen) becomes CX from gen. These are op 2,929, the selected add's carry clear, and op 5,373, the second comparison's carry.
   The cut is a table with one entry, keyed by (q, 3a mod q), so every other curve and addend emits as before, and it asserts that each op is that CCX before it replaces it.
   It is exact on every b < q and either gen, which is the call's contract: at this call b is X after M's inverse, and M's contract already requires X canonical there. So K1 adds no condition the circuit did not already rest on.
   Two independent proofs:
   - a BDD over (gen, b), from a scan of the kernel's constant adds that we ran on 2026-11-03;
   - a z3 miter on the call alone, in which the pair is unsat, while each half alone and the one other CCX pair on the same controls each give a counterexample.

   With the b < q bound dropped, z3 finds a counterexample at some b ≥ q.
   We found the pair on 2026-11-03 and did not build it then, for 2 CCX. It is the whole of this entry's T margin over PR #14.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase. Exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in the base's `DESIGN.md`.

- **Commits 1, 2 and 7** change only which clean wires hold scratch and copies. Each is clean when taken and clean when returned.
- **Commits 3, 5 and 6** replace gate sequences with ones that give the same map on every input of their domain, as each paragraph above says.
- **Commit 4** only reorders commuting gates and swaps CCX controls, and its certificate is checked per definition.
- **Commit 8** moves which carries are kept, and every kept carry is a compute and uncompute pair as in PR #14.
- **Commit 9** is proven twice on the call's contract.

Checks on the head:
- `scripts/accept.py` reports 16 pass, 0 fail and 1 pending (`I7.charge`) at seeds 2041 and 7331 on `74d3c64`. Its mutation check catches 192 of 200 CCX-to-CX mutants, against a floor of 163.
- New gate rows, each with controls that fail:
  - `I2.step` runs each rewritten `_euclid_step` against PR #14's at widths 2 to 7, on every variant of top_clear, narrow, last, kept (0, 1, half, full) and keep_two, 176 variants. Four wrong edits fail it on 118 to 122 variants each.
  - `I2.pass` certifies the depth pass on every definition at a toy prime and at secp256k1 G, with a swap control that it rejects.
  - `I2.k1` reruns the BDD on the head's kernel block at G. It checks that each half alone fails and that the pair (2,920, 5,378), the other cut of the same form, is refused, and it checks the table's one entry.
- PR #14's own `scripts/junk_check.py 600 16384` runs M on every x at every odd prime to 600, for eleven junk budgets each up to over-full. This reaches the spread order's flag branch and the kept bits, and it fails each of the four controls.
- Each commit's emission pin fails on its parent and holds on its own.
- All 264 definitions are identical over the gate's 25 builds. Built at all 40 addends of both gate samples, the circuit differs from commit 8's only at G.

## Result

On the board: pending; the bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` passed at the challenge root of published round 5 and at two fixed roots, and the guide's `qcheck.py` passed at 106 random roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 1,091,090 inputs in all.
The cost is T 22,349,551 and qubit-time 78,807,494,656, which is 4,096 wires × 19,240,111 ticks.
Every upload limit holds: 4,096 of 4,096 wires, call depth 3 of 32, 36,786,723 bytes, and an expanded ops charge of 13,642,063 of 21,696,718.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

The emit needs Python 3.12:

```sh
git clone --branch v2/junk-depth https://github.com/iwnlcern/ecdsafail-v2-reference junk-depth
cd junk-depth
git checkout 74d3c64359e46bfe3c7003e3abfb3c44891c0a06
python3.12 -I -B scripts/emit.py --hier --out junk-depth.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py junk-depth.circuit.json junk-depth.method.json
gzip -9 -n -c junk-depth.method.json > junk-depth.method.json.gz
shasum -a 256 junk-depth.circuit.json junk-depth.method.json junk-depth.method.json.gz
```

```text
44a47dec5b83c450e51034f28d96a66c72b26bd4925e5e61eaf80babb47ca575  junk-depth.circuit.json
979697dc96e24a168720e3e6922b4bb25f779ddf8cbeccf8f931a4a3202b5ebd  junk-depth.method.json
294a7dc3af366b4fd5398f833dcaead34e45d1273015bfa010a70a5e5ac3ae75  junk-depth.method.json.gz
```

The gzip hash is from macOS gzip and may differ on other builds.

## Dead ends

All numbers are measured, at G.

- **Two other junk orders.** Carries handed out in proportion to each step's capacity give 956 ticks fewer than the even spread, at the same CCX, and need more code, so we kept the even spread. PR #14's rule run from the pass's end is 2,192 ticks better than PR #14's own order, and 322,080 worse than the spread.
- **Pooled encode ancillas** (`accfc8c` on `v2/lean`) emit PR #14's circuit unchanged, since the raw tape has no block encodes.
- **Two more of our tick commits** (`4676584`, `2f3732d` on `v2/lean`) do not port. PR #9's tick holds `sw` and p each on one wire for the whole tick, so neither can move.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

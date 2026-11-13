---
evaluation: eval-2b371c0a216dad0d
pull: 42
head: 146c8b7007a0db8f7f559cd1258326a09731a997
model: Claude Opus 5.5 (design, pricing and review) and GPT in Codex (the Jump-4 cap and its proof, implementation, checks, an independent check of each commit, and review of this entry)
harness: Claude Code and Codex CLI
built_on: joel/comp-jtaroreh-pr32-4a1ba85d53ee
source: aae57fed1c69b0e7970213a3f14ae67cf88cd153
---
This entry is the circuit that branch `v2/jump-four` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`.
That commit is jtaroreh's PR #32 source, [jtaroreh/lowt-climb](https://github.com/jtaroreh/lowt-climb) at `0f5514b`, plus two commits of ours.
The first, `9504e74`, replaces the two-bit Euclid step in the dialog multiplier with Jump-4: each step strips up to four zero bits, then swaps and subtracts.
The Jump-4 step extends the Jump-2 Euclidean algorithm in §5.3.1 of the ECDSA.Fail paper (arXiv:2609.09582) to strip up to four bits; its cap, coefficient cell, selectors and quiet-run specialization are ours.
The first commit also adapts PR #32's eager high word and retained carries to the four-bit step.
The second, `aae57fe`, checks Jump-4's transcripts and schedules against independent semantic, physical-tape and schedule expectations, and emits the same circuit as the first.
Everything we keep from PR #32 is as PR #32 has it, with the credits PR #32 gives.
That includes jtaroreh's eager high word, and ab-godder's `arith.add` with a one-CCX top bit, from PR #9 and PR #28, which every cell's add uses.
The shrinking tape pool, the held offset E and the quiet-run `Prefix` that this entry builds on are ours, first entered in our PR #2, and PR #32 carries them.
Like PR #32, it aims at T only, uses all 4,096 wires and does no depth work.
At G its T is 20,621,048, 106,008 under PR #32's, and the board schedules it in 4,120,735 fewer ticks, so its qubit-time is 13.61% under PR #32's at the same 4,096 wires.

| at A = G | reference-c4 | PR #32 (built on) | this entry | change on reference | change on PR #32 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 4,096 | 4,096 | +77.24% | 0 |
| scheduled ticks | 71,023,638 | 30,279,444 | 26,158,709 | −63.17% | −4,120,735 (−13.61%) |
| CCX | 7,209,419 | 2,961,008 | 2,945,864 | −4,263,555 (−59.14%) | −15,144 (−0.51%) |
| T | 50,465,933 | 20,727,056 | 20,621,048 | −29,844,885 (−59.14%) | −106,008 (−0.51%) |
| qubit-time | 164,135,627,418 | 124,024,602,624 | 107,146,072,064 | −34.72% | −16,878,530,560 (−13.61%) |

## What changed

The point-addition sequence, the exceptional inputs, the register layout, the square and subtract, and the kernel are PR #32's.
With `QEC_JUMP4` unset, both commits emit PR #32's circuit byte for byte.

We built the first commit in three steps:

| step | change | CCX | T |
|---:|---|---:|---:|
| | PR #32 at `0f5514b` | 2,961,008 | 20,727,056 |
| 1 | Jump-4 on the shrinking tape pool | 2,996,796 | 20,977,572 |
| 2 | PR #32's eager high word and retained carries, for Jump-4 | 2,961,240 | 20,728,680 |
| 3 | quiet chains in the coefficient cell, and a first reverse cell at S = 0 | 2,945,864 | 20,621,048 |

Each row is measured at G on the whole point addition, at 4,096 wires.
We did not schedule steps 1 and 2, so the table gives no ticks for them.

Where the CCX sit after step 2, against PR #32's own breakdown:

| part | PR #32 | step 2 | runs | change |
|---|---:|---:|---:|---:|
| Euclid pass | 273,642 | 265,282 | 4 | −33,440 |
| coefficient (Bézout) pass | 888,721 | 905,054 | 2 | +32,666 |
| λ² square and subtract | 83,828 | 83,828 | 1 | 0 |
| kernel glue, flags and constant adds | 5,170 | 6,176 | | +1,006 |
| controlled point add | 2,961,008 | 2,961,240 | | +232 |

PR #32's figures are from its `scripts/breakdown.py`, which counts each pass built on its own.
With `QEC_JUMP4` set, the kernel does not cut the coefficient pass (item 1), so that pass costs the same on its own and in place.

1. **Jump-4 on the shrinking tape pool** (`qec/two_bit.py`, `qec/point_add.py` and `qec/model.py`).
   The Euclid pass now steps by Jump-4: each step strips up to four zero bits from V, then swaps and subtracts, with no halving step.
   It extends the Jump-2 macro-step of the ECDSA.Fail paper (arXiv:2609.09582, §5.3.1, "The Jump-2 Euclidean Algorithm") from two bits to four.
   At secp256k1 the pass runs 320 cells on widths that sum to 49,280.
   Each cell records four raw tape bits, the first strip has a three-bit header, and our held offset E stays live through the pass.
   Each cell of the coefficient pass inverts one step: it maps (R, S) to (R, 2^d·S + g·R mod q), with d from 1 to 4 and g in {0, 1}, and swaps the two when g and b are both set.
   Its reduction mod q reads the quotient digit through fixed selector masks at secp256k1.
   The tape takes the wires U vacates as it narrows, as in our PR #2's tape pool.
   With `QEC_JUMP4` set, `split()` returns false, so the kernel takes its unsplit path and does not cut the coefficient pass at cell 0's zero test.
   At G the point addition is 2,996,796 CCX, 35,788 above PR #32.
   Its Euclid pass is 274,171, against PR #32's 273,642, and its coefficient pass 905,054, against 888,721.
2. **PR #32's eager high word and retained carries, for Jump-4** (`qec/point_add.py`).
   PR #32 keeps the bits of U and V above their joint bound in a high word, and keeps chain carries on junk wires until the inverse pass.
   This step does both for Jump-4.
   It keeps a high word wherever the gap is at least three bits, with same/drop flags recorded for the inverse to replay.
   It retains 1,933 majority carries, allocated from the last step backward.
   Each retained carry removes one CCX a pass: with 0, 1, 967 and 1,933 carries the Euclid pass is 267,215, 267,214, 266,248 and 265,282.
   The high word alone saves 6,956 a pass against step 1.
   A 1,934th carry would take the forward multiplier to 4,097 wires.
   The Euclid pass falls to 265,282, 8,360 under PR #32's.
   The coefficient pass stays at 905,054, 16,333 a pass over PR #32's.
   The glue is 1,006 over PR #32's: that is the saving of PR #32's `4b37e11`, which cuts the coefficient pass at cell 0's zero test, and this entry's kernel takes the unsplit path.
   At G the point addition is 2,961,240 CCX, 232 above PR #32.
3. **Quiet chains in the coefficient cell, and a first reverse cell at S = 0** (`qec/two_bit.py` and `qec/point_add.py`).
   The cell's two reduction chains now carry each quiet run on our `Prefix`, as our PR #2's two-bit tick does with its quiet run.
   A quiet run is a run of bits on which every addend the cell can select is zero.
   The run comes from both chains' existing selector masks, and it must be quiet for every K from 0 to 16 in both chains; at secp256k1 it is bits 15 to 31.
   One `Prefix` serves both chains: the read-only first chain builds it, and the second writes the run and restores it.
   The coefficient pass starts at (R, S) = (y, 0), so its first reverse cell has S = 0.
   That cell alone keeps R and sets S ^= g·R, and every later cell keeps the full map.
   Quiet chains take the generic cell from 2,824 to 2,808 CCX and the header from 1,374 to 1,358, and the first reverse cell costs 256.
   The coefficient pass falls to 897,366, 7,688 under step 2's and 8,645 over PR #32's.
   At G the point addition is 2,945,864 CCX, 15,376 under step 2 and 15,144 under PR #32.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase.
Exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in PR #32's `DESIGN.md`.

- **Step 1** reaches the same endpoint, and its coefficients are the same full canonical words.
  Each step is exact on every input of its domain, and the pass ends within 320 cells at secp256k1.
  The cap is proved for every input x at secp256k1, from the potential u·v·(u + v)^3, which falls by a factor of at least 16^k over each group of k steps.
  A refined potential gives each cell's width, and the widths sum to 49,280.
  Our model check runs every prime up to 1,021, 80,017 cases on 172 primes, and 20,258 secp256k1 cases, the longest of them 314 cells.
- **Step 2** keeps the high word, its flags and every retained carry until the inverse pass, which clears them by replaying the same recorded steps.
- **Step 3** changes no value.
  On a quiet run every selectable addend is zero, so each carry across the run depends only on the carry into the run and the run's own bits below it, a prefix the `Prefix` computes.
  The first reverse cell sees S = 0, since the pass starts at (y, 0).
  Our cell checks run both reducers on every legal word at q = 31, 127, 991 and 4,079, and on 8,000 generic-cell and 7,562 header inputs at secp256k1, forward and inverse.
  A wrong quiet span fails 7,522 generic-cell and 7,094 header lanes each way, and the S = 0 cell put in place of a later cell makes the multiplication fail.

Checks on the head, `aae57fe`:
- `scripts/accept.py` reports 13 pass, 0 fail and 1 pending (`I7.charge`) at seeds 2041 and 7331, with `QEC_JUMP4=1`.
  Its mutation check catches 170 of 200 CCX-to-CX mutants, against a floor of 163.
- Its rows run 36 toy instances, 25,800 lanes, on the flat circuit and on the hierarchical emission; the hierarchical upload expanded at 24 secp256k1 addends, 252 lanes each; the flat build on 252 lanes at G; and 52 components on 13,867 lanes.
  `I5` finds all 325 definitions identical over 25 builds.
- `I2.model` runs the second commit's model and schedule checks: exhaustive over x at 13 primes, and at secp256k1 the 320-cell bound and 2,000 random products.
  With `QEC_JUMP4` unset, both checks print what they printed before that commit, byte for byte.
- `I6` counts T 20,621,048 at G, and at most T 20,621,188 over 24 sampled addends at the two seeds.
- Independent checks of the first commit's code, as built at step 3, and of the second commit, each on frozen source, passed.

## Named limits

These are PR #32's, and this entry keeps them: its `INVARIANTS.md` states them as PR #32's does, word for word.

- `I2.model`'s zero-caller row is narrowed from the reference's gate.
- The dialog multiplier M's standalone contract is narrowed to x in [1, q), and x ≠ 0 at M's entries rests on the kernel's flag code.
  Jump-4 keeps that contract: it changes how M computes, not the map it computes.
  On PR #32's source, the direct evidence for that premise is an instrumented check that X ≠ 0 at both of M's entries on all 25,800 toy lanes and at the secp256k1 special points.
- The build refuses curves with a finite point at x = 0, and addends with x(A) = 0 or x(2A) = 0.

## Result

On the board: pending; the bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` passed at the fifth published challenge root, at two fixed roots and at 106 fresh random roots, and the guide's `qcheck.py` passed at the first three.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 1,091,090 inputs in all.
The cost is T 20,621,048 and qubit-time 107,146,072,064, which is 4,096 wires × 26,158,709 ticks.
Every upload limit holds: 4,096 of 4,096 wires, call depth 3 of 32, 55,453,347 bytes, and an expanded ops charge of 10,417,979 of 21,696,718.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

With Python 3.14.

```sh
git clone --branch v2/jump-four https://github.com/iwnlcern/ecdsafail-v2-reference jump-four
cd jump-four
git checkout aae57fed1c69b0e7970213a3f14ae67cf88cd153
QEC_JUMP4=1 python3.14 -I -B scripts/emit.py --hier --out jump-four.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3.14 -I -B wrap.py jump-four.circuit.json jump-four.method.json
gzip -9 -n -c jump-four.method.json > jump-four.method.json.gz
shasum -a 256 jump-four.circuit.json jump-four.method.json jump-four.method.json.gz
```

```text
ce0aa800bf49e097ef558ef422667186fc43a3931d10bc57e912f4162c0ff6d1  jump-four.circuit.json
659cdbb19bf3bbe3a2799a3383278da4018068a334931f35a6d46b603b2545d6  jump-four.method.json
1b4492cef32130fca3055504fea6765e7823c0c287b9d5b1b58f28ebfb16a63c  jump-four.method.json.gz
```

`QEC_JUMP4=1` selects Jump-4; with it unset, the same commands give PR #32's circuit.
`wrap.py` is the reference's `SubmittedMethod` wrapper, from release `qcb-8376098` of jtaroreh/ecdsafail-v2-reference, and it makes the method file from the emitted circuit.
The gzip hash is from macOS gzip and may differ on other builds.

## Dead ends

- **PR #32's lazy high word on the four-bit step.**
  We worked it out for the four-bit step and planned it on step 2, but did not build it for this entry, so the high word here is eager.
- **PR #32's lag blocks, tick-0 absorption and rank-one start, and its later cell trims.**
  We have not proved them for the four-bit step, so this entry has none of them.
- **PR #32's cut at cell 0's zero test** (`4b37e11`).
  We did not adapt the cut to the four-bit cell; this entry's glue pays its 1,006 CCX.

---
evaluation: eval-59ccd89b5830fa90
pull: 46
head: c1ac659af468eb4e39263baf0d9c62ba9182d4c3
model: Claude Opus 5.5 (design, pricing and review), Claude Fable 5.2 (the first build of the filter that commit 1 ports) and GPT in Codex (implementation, checks, an independent check of each commit, and review of this entry)
harness: Claude Code and Codex CLI
built_on: mike/comp-ab-godder-pr29-dcb7883b4b7a
source: f8e920f6e95fca68d447b5e50f2809e36222b36b
---
This entry is the circuit that branch `v2/union-par` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`, in the form the board scores, after ab-godder's post-passes in `scripts/post_pass.py`.
That commit is ab-godder's PR #29 source, `qt-mike` at `61e0c50`, plus eight commits: the six in the table below, `3bb03f1`, which records the head's pins and costs in `README.md`, and `f8e920f`, which corrects three descriptions in `DESIGN.md` and `README.md`.
PR #29 is abdowns's PR #18 at `b9b464a` plus ab-godder's commits, and everything PR #29 and the entries it names made is kept here and is their work.
That includes PR #29's pebbled high increment, its tick and Euclid step latency changes, its CCX market and its post-passes (`qec/split_dirs.py`, `qec/reorder_pass.py`, `scripts/post_pass.py`), which are ab-godder's. The post-pass `qec/gate_pass.py` is 2elliot's, from PR #3.
It aims at qubit-time, and keeps T under the reference's.

| at A = G | reference-c4 | PR #29 (built on) | this entry | change on reference | change on PR #29 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,546 | 1,546 | −33.10% | 0 |
| scheduled ticks | 71,023,638 | 1,920,902 | 1,856,756 | −97.39% | −64,146 (−3.34%) |
| qubit-time | 164,135,627,418 | 2,969,714,492 | 2,870,544,776 | −98.25% | −99,169,716 (−3.34%) |
| CCX | 7,209,419 | 7,208,765 | 7,187,137 | −22,282 (−0.31%) | −21,628 (−0.30%) |
| T | 50,465,933 | 50,461,355 | 50,309,959 | −155,974 (−0.31%) | −151,396 (−0.30%) |
| expanded ops charge | 20,960,620 | 19,201,248 | 19,144,695 | −8.66% | −56,553 (−0.29%) |
| upload bytes | 43,423,435 | 76,683,894 | 75,309,234 | +73.43% | −1,374,660 (−1.79%) |

**Over PR #29 this entry cuts qubit-time by 3.34%, at the same 1,546 wires and with 21,628 fewer CCX.**
T stays 155,974 under the reference's, which is 22,282 CCX of room.
On the board when this was written, no entry has both lower T and lower qubit-time than this one.

## What changed

Our commits, in order, with a screen of each.

### Per-commit screens, before PR #29's reorder pass

Each row is read from a saved price file, measured at G after the emission, PR #29's direction split and `qec/gate_pass.py`, without PR #29's reorder pass, which runs on the head alone.
The last row is the head's final check, after the whole post-pass.

| # | commit | change | wires | scheduled ticks | CCX | T |
|---:|---|---|---:|---:|---:|---:|
| | `61e0c50` | PR #29 | 1,546 | 1,988,246 | 7,208,765 | 50,461,355 |
| 0 | `2e86512` | the component check narrowed to the multiplier's promise | 1,546 | 1,988,246 | 7,208,765 | 50,461,355 |
| 1 | `96249b2` | dead joins dropped from the constant trees | 1,546 | 1,988,032 | 7,204,497 | 50,431,479 |
| 2 | `a359d7b` | ab-godder's secp256k1 width cut, from PR #28 | 1,546 | 1,987,908 | 7,202,689 | 50,418,823 |
| 3 | `5a5302d` | the k update on the trial chain's copies | – | – | – | – |
| 4 | `7bf3326` | the add's backward CX wave as one network | 1,546 | 1,963,122 | 7,202,689 | 50,418,823 |
| 5 | `1c179af` | the square's three row runs side by side | 1,546 | 1,910,922 | 7,187,137 | 50,309,959 |
| | `f8e920f` | the head after PR #29's whole post-pass, final check | 1,546 | 1,856,756 | 7,187,137 | 50,309,959 |

Commit 3's emission was pinned but not priced alone; it was priced together with commit 4, so its row is blank.
Commit 5's row is also `3bb03f1`'s final-check figure before its reorder pass.

0. **The multiplier's component check, narrowed to its promise** (`scripts/component_check.py`, with `COVERAGE["component_lanes"]` in `scripts/accept.py` and a line in `INVARIANTS.md`).
   On PR #29's source as entered, `component_check.py` fails `dialog_mul` on 6 of 57 lanes, exactly the six with x = 0, as PR #29's note reports for unmodified `b9b464a`. x = 0 is outside the multiplier's narrowed domain.
   This commit drops those six lanes, as our own line has since `c39f210`, and the check passes: 49 components and 8,945 lanes, with `dialog_mul` at 51 lanes and 0 failures. It changes no circuit.
1. **Dead joins out of the constant trees** (`_live` in `qec/arith.py`, called by `gt_const_tree`, `add_lookahead` and `_add_selected_tree`).
   The kernel's constant adds and reflection, and tick 0's selected add, run 2elliot's `gt_const_tree` and `_add_selected_tree` (`737c638`), which reach PR #29 through abdowns's PR #18. The second adds through our `add_lookahead` (`6f84671`), which 2elliot extended.
   Many of those trees' CCX have a control that holds 0 on every input: a leaf at one of the constant's 0 bits, a fresh temp, a wire that the tree writes and restores, or a wire that holds the XOR of two copies of one selector.
   `_live` is a pass over a recorded gate list that is told what some wires hold where the list starts. It follows each wire as an XOR of symbols, and drops every CX or CCX one of whose controls is the empty sum.
   It takes 4,268 CCX at G, and 214 ticks on the footing of the table above. The filter is ours.
2. **ab-godder's width cut at secp256k1** (`tick_widths` in `qec/model.py`, with `scripts/width_check.py` and `scripts/width_search.c`), from PR #28's `e1b32bb`, merged as `5ca6779`.
   At secp256k1, Euclid ticks 129 to 168 run one bit under the generic width schedule. Every other prime keeps the generic schedule.
   The proof and the exhaustive search are ab-godder's, in PR #28's `DESIGN.md`: 53,613,560,142 nodes and three candidate inputs, none of which breaks the cut. We cite that proof and run, and do not rerun the full search.
   `SECP256K1_CUT_LAST = -1` restores the generic schedule and the previous commit's circuit. It saves 1,808 CCX, and 124 ticks on the table's footing.
3. **The k update on the trial chain's copies** (`KPAR` in `TwoBitMod.tick`, `qec/two_bit.py`). This is ours.
   With abdowns's `WIDE_ASIDE` and ab-godder's `KUPD_FAST`, both in PR #29's source, the tick's k update ran its three CCX one after another, each waiting for the last to write what it reads.
   `KPAR` has each update read the old k0, k1 and k0·k1 from the trial chain's copies, which nothing changes before `clear_sc`, and e from two copies, one on `one`'s clean wire. The three CCX then run side by side, at the same CCX.
   On one tick alone it saves 22 ticks.
4. **The add's backward CX wave as one network** (`WAVE_BACK_ONE` in `add_or_shift_la`, `qec/two_bit.py`). This is ours.
   PR #29 runs the add's first CX wave as a log-depth network in 64-bit segments (ab-godder's `WAVE_SEG`), and its backward wave ran the segments' reverse one after another.
   `WAVE_BACK_ONE` runs the reverse of one `adjacent_xor` network over all of U instead, which computes the same map. On one tick alone it saves 11 ticks, at the same CCX.
   `adjacent_xor`, a Brent–Kung network (Brent and Kung, 1982), is ours, from our `d9ecbbe`, and PR #18's source carries it unchanged.
   Commits 3 and 4 together save 24,786 ticks on the table's footing.
5. **The square's three row runs side by side** (`SQ_PAR` in `SquareFold`, `qec/square.py`, with `scripts/square_par_check.py`). This is ours.
   The folded square now makes S = Y0 + Y1 from the untouched Y first, then runs its three runs of rows, Y0's on lo, Y1's on hi and S's on K, side by side on disjoint temps.
   The halves' runs take 44 temps each, and S's run the rest, with 6 more wires. The square's call sites have exactly those 6 spare wires, so the circuit's peak stays in the Bézout tick's P tree.
   S's rows are trees only from 200 bits up (`SQ_PAR_MIN`), so its run is chains, like the halves'. At small moduli, where the fixed split does not fit, each run takes a third of the temps.
   It saves 15,552 CCX and 52,200 ticks on the table's footing.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase. Exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in the base's `DESIGN.md`.

- **Commit 0** changes a check, not the circuit.
- **Commit 1** drops a gate only where one of its controls is 0 on every state where the told facts hold, and those facts are ones the unfiltered code's own map already needs.
  The reverse sweep runs on the states the forward sweep made from such inputs, so the kept list's reverse is the same inverse. None of this depends on the constant or on b.
- **Commit 2** is ab-godder's argument, about the algorithm in `qec/model.py`, which is the same on PR #28 and PR #29 apart from the schedule's lines.
- **Commit 3**: each update reads only old values, k2 ^= (k0·k1)·e, three ^= k1·e, k1 ^= k0·e and k0 ^= e, and the trial chain's copies hold those values until `clear_sc`, which PR #29's tick already relies on. e's copies are made and cleared with CX.
- **Commit 4**: one `adjacent_xor` network over U and the 64-bit segments compute the same map, so their reverses are the same inverse, run in a different order and depth.
- **Commit 5** computes the same row products and sums, in another order on other clean wires.

Checks on `3bb03f1`; the head, `f8e920f`, differs from it only in `DESIGN.md` and `README.md`:
- `scripts/accept.py` reports 16 pass, 0 fail and 1 pending (`I7.charge`) at seeds 2041 and 7331. Its mutation check catches 191 of 200 CCX-to-CX mutants.
- Its rows run 36 toy instances, 25,800 lanes, on the flat circuit and on the hierarchical emission; the hierarchical upload expanded at 24 secp256k1 addends, 252 lanes each; the flat build on 252 lanes at G; and 49 components on 8,945 lanes. `I5` finds all 270 definitions identical over 25 builds.
- `I6` counts the emission before PR #29's post-passes: 7,188,435 CCX and T 50,319,045 at G, and at most T 50,320,557 over 24 sampled addends at the two seeds. The post-pass removes 1,298 of those CCX, which gives the scored figures.
- Two new gate rows check commit 1. `I2.filter` (`scripts/filter_check.py`) runs each filtered block and its unfiltered twin on a binary decision diagram over (ctrl, b), and `I2.filter-small` (`scripts/filter_exh.py`) runs the kernel's constant blocks at small primes on every b < q.
  `I2.filter` covers 65 blocks at 16 addends, whose filtered lists are the unfiltered ones with 30,866 CCX removed, in order. `I2.filter-small` covers 160,198 blocks, at every constant of 170 primes up to 1,021.
  Their controls tell the filter a wrong zero. That changes 4 of 5 blocks at G, and fails 12,048 of 12,100 blocks up to 251.
- A new gate row, `I2.square-par`, checks commit 5 on every n-bit Y at its toy primes and on 14 edge and 240 seeded random secp256k1 inputs: the square's residue, every Y and J wire against the square without `SQ_PAR`, and clean scratch. Its control turns a middle CCX into a CX and must fail at secp256k1.
- Each lever's flag, set off, restores the previous commit's emission byte for byte.
- `f8e920f` changes only `DESIGN.md` and `README.md`, and every other file is byte for byte `3bb03f1`'s. The full gates above and the roots under Result ran on `3bb03f1`, not on `f8e920f`.
  From a clean linked worktree at `f8e920f`, the emission, the post-pass and the wrap give the three hashes below again, with the same price and the same flat-circuit digests at toy-127 and at secp256k1.

## Named limits

- `I2.model`'s zero-caller row is narrowed from the reference's gate, as on PR #29's source.
- The multiplier's standalone contract is narrowed to x in [1, q), and x ≠ 0 at its entries rests on the kernel's flag code, as on PR #29's source. Commit 0 makes the component check say so.
- On PR #29's source as entered, `scripts/check_hier.py` is red against the flat receipt in `receipts/baseline/`. The hierarchical build orders and labels gates differently from the flat one: at toy-127 both have 32,943 ops with the same counts by kind, and they first differ at op 4, where a CCX has its two controls exchanged.
  We do not edit the receipts. Instead, each commit checks both builds against the specification on every toy input.

## Result

On the board: pending; the bot's comment will give it.

Locally on `3bb03f1`, `qcb` 8376098 `diagnostic check` passed at the challenge root of published round 5 and at two fixed roots, and the guide's `qcheck.py` passed at 106 random roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 1,091,090 inputs in all.
The cost is T 50,309,959 and qubit-time 2,870,544,776, which is 1,546 wires × 1,856,756 ticks.
Every upload limit holds: 1,546 of 4,096 wires, call depth 3 of 32, 75,309,234 bytes, and an expanded ops charge of 19,144,695 of 21,696,718.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

With Python 3.12. The post-pass is ab-godder's and is deterministic.

```sh
git clone --branch v2/union-par https://github.com/iwnlcern/ecdsafail-v2-reference union-par
cd union-par
git checkout f8e920f6e95fca68d447b5e50f2809e36222b36b
python3.12 -I -B scripts/emit.py --hier --out union-par.circuit.json
python3.12 -I -B scripts/post_pass.py union-par.circuit.json union-par-pp.json --jobs 1 --rounds 300
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py union-par-pp.json union-par.method.json
gzip -9 -n -c union-par.method.json > union-par.method.json.gz
shasum -a 256 union-par.circuit.json union-par-pp.json union-par.method.json union-par.method.json.gz
```

```text
1feca2c1ea0c67e274bd5320c702a0d91373daf9c9824e6de25b5682ee13a609  union-par.circuit.json
5649f4139a389cf7ae03b53aff0877e7c81592993d74c32686d299850e9e4d02  union-par-pp.json
6ad4f26ff1233d48f5b4acebbefc32789d955f5410509a28b1401feeb75f93e3  union-par.method.json
0b26274dacc150e81a16e550f5df3fd75a197423978743929d77a3c470724fa0  union-par.method.json.gz
```

`wrap.py` is the reference's `SubmittedMethod` wrapper, from release `qcb-8376098` of jtaroreh/ecdsafail-v2-reference, and it makes the method file from the post-passed circuit.
The gzip hash is from macOS gzip and may differ on other builds.
The post-pass took 4,206 s with one job, and 1,738 s with 2 in our final check, which gave the same file.

## Dead ends

Each line says where it was measured: on the footing of the commit table, on one tick or one square alone, or on PR #29's entered method.

- **Our critical-hop search** (from our PR #23), on PR #29's entered method: 2,511 more ticks, at 20,000 and at 40,000 evaluations. A copy that scores only the directions each body runs changes no body, because the reorder pass already holds every move ours makes.
- **Trees for the long rows of the square's runs**, one square alone. With 42 or 43 temps a run, a tree row is slower than the chain it replaces. On S's run, tree thresholds from 128 down to 32 bits take the square from 69,471 ticks to 69,930 to 82,997, for 474 to 24,596 more CCX.
- **The square's split adds as trees** (`SPLIT_TREE`), one square alone: at best 254 fewer ticks for 1,504 more CCX.
- **`SQ_PAR` with more wires**, on the table's footing: with 45 extra wires the head is 1,585 × 1,903,731, worse.
- **PR #28's items that we did not carry over**, estimated, credited to ab-godder: the held offsets need 384 wires through the Euclid pass; the shared bottom carry makes the second chain wait on the trial's undo, about 8,000 to 27,000 more ticks against at most about 1,070 that its 766 CCX buy; the swap bits on the tape need about 380 wires; and the B1 sweep in the ticks comes out about even.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---
evaluation: eval-68d2e1e94fa382e9
pull: 23
head: 8ca189baa587a3da04870a7d9a82760b4853ad10
model: Claude Opus 5.5 (design and pricing), Claude Fable 5.2 (implementation, checks and review)
harness: Claude Code
built_on: aodhfionn/comp-abdowns-pr18-b9b464adf474
source: 35b66cba24cdabc898980dc183298531226786d4
---
This entry is the circuit that branch `v2/hop-search` of iwnlcern/ecdsafail-v2-reference emits at the commit in `source`, in the form the board scores, after two post-passes.
That commit is abdowns's `qt/base` at `2b9bdc8`, the source of PR #18, plus four commits of ours: two that change the circuit, one that adds its checks and one that corrects text.
PR #18 is built on abdowns's PR #17. Everything PR #18 and the entries it names made is kept here and is their work, and the post-pass `qec/gate_pass.py` is 2elliot's, from PR #12.
It aims at qubit-time. The wires, CCX and T are PR #18's.

| at A = G | reference-c4 | PR #18 (built on) | this entry | change on reference | change on PR #18 |
|---|---:|---:|---:|---:|---:|
| wires | 2,311 | 1,555 | 1,555 | −32.71% | 0 |
| scheduled ticks | 71,023,638 | 2,404,728 | 2,323,117 | −96.73% | −81,611 (−3.39%) |
| qubit-time | 164,135,627,418 | 3,739,352,040 | 3,612,446,935 | −97.80% | −126,905,105 (−3.39%) |
| CCX | 7,209,419 | 7,162,267 | 7,162,267 | −0.65% | 0 |
| T | 50,465,933 | 50,135,869 | 50,135,869 | −0.65% | 0 |
| expanded ops charge | 20,960,620 | 18,436,850 | 18,501,034 | −11.73% | +64,184 (+0.35%) |
| upload bytes | 43,423,435 | 73,573,976 | 73,596,584 | +69.49% | +22,608 (+0.03%) |

**The cut over PR #18 is all ticks, at the same 1,555 wires, CCX and T.** Most of it is one new pass: a search over the order of commuting gates that does by machine the hop-by-hop walk PR #18 did by hand.
The rest is two of PR #18's own settings at new values.
T is 330,064 below the reference, as in PR #18, which is 47,152 CCX of room.

## What changed

Our commits, in order, with their effect at G after the post-passes:

| # | commit | change | wires | scheduled ticks | CCX | T |
|---:|---|---|---:|---:|---:|---:|
| | `2b9bdc8` | PR #18 | 1,555 | 2,404,728 | 7,162,267 | 50,135,869 |
| 1 | `767e707` | the critical-hop search, after the post-pass | 1,555 | 2,334,421 | 7,162,267 | 50,135,869 |
| 2 | `01f45ee` | the search's certificate and toy check, as gate rows | 1,555 | 2,334,421 | 7,162,267 | 50,135,869 |
| 3 | `3744b5d` | 20 copies of J for the tick's add, and 48 of p for its un-copy | 1,555 | 2,323,117 | 7,162,267 | 50,135,869 |
| 4 | `35b66cb` | a corrected reason in DESIGN.md | 1,555 | 2,323,117 | 7,162,267 | 50,135,869 |

1. **The critical-hop search** (`scripts/cp_search.py`, run on the method `qec/gate_pass.py` writes).
   PR #18's rule: on the board's schedule, two gates that hold the same wire in the same role, both reading it as a control or both only XOR-ing into it, take turns on it although they commute, and the one emitted first goes first.
   PR #18 walked the critical path of one Bézout tick and of one Euclid step by hand, marked every hop where the next gate waited only for its turn, and reordered the routines behind those hops.
   The search does that walk on the scored method, for every definition without calls.
   It times each body alone, forward and inverse, under the board's tick model with every wire ready at 0, and walks the critical path of each run.
   At every hop where a gate's start was set by the gate before it on a wire both read or both only XOR into, it tries four moves in order: the waiting gate just before the other, the other just after it, and either CCX with its controls exchanged.
   A move needs the moved gate to commute with every gate it passes, and it is kept only if the body's forward plus inverse ticks fall. The walk then restarts on the new path.
   It stops when no hop's move helps, or after 20,000 evaluations of one body, which no definition here reaches.
   It keeps every gate, so CCX, T and the expanded ops charge do not move. It reads only the body it changes and has no randomness, so definitions that are identical before it are identical after it.
   It leaves out the square-subtract's definition, 501,276 ops, because each evaluation re-times the whole body. At commit 1, on PR #18's settings, all 267 other definitions without calls change: at G their standalone forward plus inverse ticks go from 259,582 to 247,245, and the circuit's from 2,404,728 to 2,334,421.
   The circuit gains more than these standalone sums because the definitions run many times: weighted by their forward and inverse runs, the standalone gains add to 80,627 ticks, more than the circuit's 70,307.
   It is critical-path list scheduling (Adam, Chandy and Dickson, CACM 17(12), 1974) run as a local search over the orders of commuting gates, as commutation-based gate reordering does in quantum circuit compilers (Itoko et al., *Integration* 70, 2020, arXiv:1907.02686).
   PR #12's note lists "Running the depth pass again after the post-pass" as a dead end, and this is a different pass.
   The depth pass list-schedules each definition from its dependencies, and its own tie-break makes the waits the search removes.
   Running PR #18's depth pass a second time after the post-pass gives −64 ticks; the critical-hop search on the same method gives −70,307.
2. **The search's certificate and toy check** (`scripts/cp_cert.py`, `scripts/cp_toy_check.py`, and four rows in `scripts/accept.py`).
   `scripts/cp_cert.py` shares no code with the search. It certifies the searched circuit at G against the post-passed one: on every wire of every changed definition, the same sequence of role runs, each holding the same gates with CCX controls sorted, and nothing else changed.
   `scripts/cp_toy_check.py` runs 22 toy point additions, two addends on each of 11 curves from q = 13 to 577, on every lane, as emitted, after `qec/gate_pass.py`, and searched.
3. **More copies of J for the tick's add, and of p for its un-copy** (`WIDE_ADD_COPIES` and `P_FAN` in `qec/two_bit.py`).
   Both are abdowns's code. This commit sets them to 20 and 48, against PR #18's 16 and 8, values our sweep of the two settings chose.
   The tick's add writes each sum bit under a copy of J, and the rotation's swaps take copies of J as controls too. After the add, p is un-copied from every word bit through `P_FAN` copies of p, each copy un-copying one slice.
   More copies let more of those gates run at once. They change only how many wires carry a copy and which copy a gate reads, so CCX and T do not move. The copies add CX gates, so the expanded ops charge rises by 64,184. The circuit's peak is later in the tick, in its high increment, so the wire count does not move either.
   Before the search the circuit goes from 2,404,728 to 2,393,643 ticks, 11,085 fewer. With the search, 2,323,117: 11,304 under the search alone, so the two gains add.

## Why it works

Every gate is still X, CX, CCX or SWAP, so the circuit is a permutation with no phase. Exactness on basis states of each block's promise domain, with clean scratch, is the whole contract, as in the base's `DESIGN.md`.

- **Commit 1** only moves a gate past gates it commutes with, and exchanges CCX controls. Gates that share only wires they all read, or wires they all only XOR into, commute, so equal role runs on every wire mean the same permutation. `scripts/cp_cert.py` checks that per definition.
- **Commit 3** changes only how many clean wires hold a copy. Every copy of J holds J and every copy of p holds p, and each is cleared before its wire is freed.

Checks on `3744b5d`, whose circuit `35b66cb` re-emits byte for byte:
- `scripts/accept.py` reports 18 pass, 0 fail and 1 pending (`I7.charge`) at seeds 2041 and 7331. Its mutation check catches 187 of 200 CCX-to-CX mutants, against a floor of 163.
- New gate rows, each with a control that must fail:
  - `I2.search` certifies the searched circuit at G on this head, with commit 3's settings: 267 reordered definitions, standalone forward plus inverse ticks of 259,401 before the search and 247,052 after, and the certified circuit is the upload byte for byte.
  - `I2.search-control` swaps the first adjacent pair of gates that do not commute in each of the 267, and the certificate refuses every one.
  - `I2.search-toy` checks the 22 toy instances in all three forms, 9,812 lanes, and the search changes every instance.
  - `I2.search-toy-control` swaps one adjacent non-commuting pair in a searched definition, and the lanes catch it in all 22.
- Commit 3's settings: `scripts/check.py --toy-bits 4 6 7 8 9 10 --toy-per-size 6`, flat and `--hier`, 36 instances and 25,800 lanes, 0 failing; `scripts/add_or_shift_check.py . 8 tick` OK; `scripts/wide_check.py .` OK.
  The wide add runs at every n above 3 and the fan at every n, so the toy rows reach both settings. An un-copy that skips the fan's first word bit fails 18 of the 36 toy instances.
- Each circuit commit's pin fails on its parent and holds on its own.

## Result

On the board: pending; the bot's comment will give it.

Locally, `qcb` 8376098 `diagnostic check` passed at the challenge root of published round 5 and at two fixed roots, and the guide's `qcheck.py` passed at 106 random roots.
Every root fails 0 of 10 boundary cases and 0 of 10,000 seeded inputs, 1,091,090 inputs in all.
The cost is T 50,135,869 and qubit-time 3,612,446,935, which is 1,555 wires × 2,323,117 ticks.
Every upload limit holds: 1,555 of 4,096 wires, call depth 3 of 32, 73,596,584 bytes, and an expanded ops charge of 18,501,034 of 21,696,718.
This is a sampled test, so it neither proves the circuit exact nor qualifies it.

## Reproduce

The emit needs Python 3.12. We ran the post-pass and the search under Python 3.14:

```sh
git clone --branch v2/hop-search https://github.com/iwnlcern/ecdsafail-v2-reference hop-search
cd hop-search
git checkout 35b66cba24cdabc898980dc183298531226786d4
python3.12 -I -B scripts/emit.py --hier --out hop-search.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py hop-search.circuit.json hop-search.raw.method.json
python3 -I qec/gate_pass.py hop-search.raw.method.json hop-search.post.method.json
python3 -I scripts/cp_search.py hop-search.post.method.json hop-search.method.json
gzip -9 -n -c hop-search.method.json > hop-search.method.json.gz
shasum -a 256 hop-search.circuit.json hop-search.post.method.json hop-search.method.json hop-search.method.json.gz
```

```text
50f542faee3ba97c51be0f92a72700ac9c3372e23acacc115c13698864751b3c  hop-search.circuit.json
b6d3c47562b6397ca7ec202bc12921aa50eab9d046a915f8760f35c1f2d9d6e8  hop-search.post.method.json
a381b21e2c1a0ce4f4e39c4cba2b9d578fe0e62e30485b0b4c9f3f03a4f88ec7  hop-search.method.json
f7a605e39884d73dbcdc4f25338c5922fd99fce1dc192c938f96e84a1d77d922  hop-search.method.json.gz
```

The gzip hash is from macOS gzip and may differ on other builds.
The search takes about 18 minutes and 659 MB of memory on one core.

## Dead ends

All numbers are measured, at G, after the post-pass, against this entry's settings and before the search unless a line says otherwise.

- **Other values of the two settings**, against PR #18's 2,404,728 ticks. `P_FAN` alone at 6 is +3,820 and at 64 is −9,330. `WIDE_ADD_COPIES` alone at 14 is +6,007 and at 32 is −1,422.
  With 20 copies, `P_FAN` 96 is −11,633 before the search, 548 ahead of 48, but 544 ticks behind it after the search, so we kept 48.
- **Settings at the circuit's 1,555-wire peak.** The peak is the Bézout tick's high increment, P's prefix tree, while it writes. Blocks of 3 there (`WIDE_P` 3) give 1,573 wires and 15,480 fewer ticks, +0.50% qubit-time. Larger blocks are PR #18's own dead end, so we do not list them.
  `WIDE_M` 2 and 4 and `WIDE_LOW` 2 lose too, by +0.12% to +1.98%.
- **The square's temps.** The square peaks 9 wires under the circuit, with its rows at the peak. `ROW_TEMPS` 40 is 1,556 wires for 290 fewer ticks. `FOLD_TEMPS` 160 is 1,580 wires. `FOLD_TEMPS` 135 is free and gives 72 fewer ticks, and the same 72 after the search on another setting; we left it out as too small to carry.
- **Which copies control the final swap.** The swaps on the fan tree's leaves only cost 7,037 ticks. On every node but `sw` they cost 271 before the search, and after it they gave −326 here but +598 on another setting, so we left them out.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

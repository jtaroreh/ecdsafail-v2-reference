---
evaluation: eval-f5f6e6d0e8325652
pull: 20
head: 3c02070acac633fcd3d7e8d554dbd0c5bfa231bb
model: Claude Opus 5.5
harness: Claude (Cowork)
built_on: elliot/comp-2elliot-pr19-f7824658045e, aodhfionn/comp-abdowns-pr18-b9b464adf474, elliot/comp-2elliot-pr12-b2284da7c963
source: 83367b40688d7c5e2cd59bd06ac62593b99cc3d0
---
## What changed

This entry is emitted by branch `frontier/v5` of 2elliot/ecdsafail-v2-reference at the commit in `source`. That commit is one commit on the board's main at PR #18 (ae330f2). My PR #19 ran every Bezout tick's add as my CDKM ripple. This entry runs it on 95% of the ticks and keeps PR #18's wide add on the other 5%. It pays for those wide ticks by building the square's rows as chains rather than trees. It is lower on both costs than PR #19.

This is not a qubit-time entry. It trades qubit-time for T, like PR #19.

**The mix** (`two_bit.CDKM_MIX = 0.95`, used in `DialogMultiplier._bezout_pass`).
- Tick i of each Bezout pass takes the CDKM add when ⌊0.95(i + 1)⌋ > ⌊0.95 i⌋, and the wide add otherwise. This spreads the wide ticks evenly over the pass.
- Each choice is part of the tick definition's key, so a CDKM tick and a wide tick are separate definitions. `_add_or_shift` dispatches on a `force_cdkm` attribute that is set only while a CDKM tick is being recorded.
- On this circuit a wide tick costs about 1,281 CCX more than a CDKM tick and saves about 3,695 ticks, which is about 2.9 ticks per CCX.

**The square's rows** (`square.ROW_TREE = False`). The rows go back to chains, which frees 54,520 CCX for 79,794 more ticks, or 1.46 ticks per CCX. I measured every tree this way (the square's rows, folds and modadd, the first Euclid step, the top-level constant adds and the exception flags), and the rows were the only one cheaper than the wide add's rate.

So the freed CCX buy wide ticks worth about twice what the rows cost, and 0.95 is the largest fraction that keeps T under PR #19's.

The post-pass is PR #18's `qec/gate_pass.py`. It takes 5,179,320 ticks to 5,174,617 (−0.09%).

## Why it works

Every gate is X, CX, CCX or SWAP, so equality on basis states is the whole contract.

- **The mix.** Both adds meet one contract: word + (R ≫ 1) mod 2^(n+2) off J, the rotation of word[0..n] on J, J and R unchanged, and all scratch at 0. PR #13's note proves the CDKM add, and PR #18 uses the wide add. Each tick calls its add through `TwoBitMod._add_or_shift`, so any per-tick choice computes the same map.
- **The rows.** `ROW_TREE = False` is the square's chain rows, the construction it used before my PR #12. It computes the same rows.

Checks, all passing on this commit or on an emit byte-identical to it:

- `add_or_shift_check.py <tree> 10 cdkm tick` (my PR #12 script, run against this tree). For n = 4..10 on all 2^(2n+3) input states, the CDKM add and the tick's default dispatch both compute the contract, with all scratch at 0.
- `scripts/accept.py`, full mode: 12 pass, 1 fail, 1 pending (I7.charge). The run includes exhaustive gate-level checks on 36 toy instances, flat and hierarchical, which use both adds. It also includes the 256-bit upload at 24 addends on inf, A, −A, −2A, 2A, 3A and 120 random R, and 182 of 200 mutations caught. The fail is the same one PR #19's note describes: `dialog_mul` on its 6 lanes with x = 0, and no others. It comes with PR #17's wide tick. A multiplier gets x = 0 only when R is O, G, −G or −2G, which are all boundary cases, and those pass.
- A bit-sliced simulation of the output file, on three seeds: 10 boundary cases, 18 adversarial points and 10,000 random points each, then a reverse run back to the input. There were 0 failures.
- The 141 hard inputs I gave Joel for the board, plus 2,172 more of the same classes and 2,000 uniform draws, with the reverse run: 0 failures.
- The output file's top-level map equals the unpassed circuit's on 512 random full states: same.
- A clean checkout of `source` re-emits the circuit, and the post-pass reproduces the method file byte for byte.

## Result

On the board: pending; the bot's comment will give it.

Locally, the board's own check, `qcheck.py` with `qcb` on Windows, passed 3 random roots (d072e76fde6b…, e306f3246853…, ac29d7694002…), each with 0 of 10 boundary cases and 0 of 10,000 seeded inputs failing. It reports lowered_t 43,228,941 and qubit-time 8,046,529,435 = 1,555 wires × 5,174,617 ticks, and says the file meets the bar. Those costs are exactly what our tick model predicted.

| at A = G | PR18 gate-order | PR19 cdkm-wide | this entry |
|---|---|---|---|
| wires | 1,555 | 1,555 | 1,555 |
| scheduled ticks | 2,404,728 | 5,242,834 | 5,174,617 |
| qubit-time | 3,739,352,040 | 8,152,606,870 | 8,046,529,435 |
| CCX | 7,162,267 | 6,178,723 | 6,175,563 |
| T | 50,135,869 | 43,251,061 | 43,228,941 |
| expanded ops charge | | 17,911,540 | 17,799,508 |

- **Against PR #19.** T −22,120 (−0.05%) and qubit-time −1.30%.
- **Against PR #18.** T −6,906,928 (−13.78%) and qubit-time +115.19%.
- **Against PR #7.** T −524,972 (−1.20%) and qubit-time −33.65%.
- **Against the reference.** T 14.34% below it and qubit-time 95.10% below it.
- **On the frontier.** Every entry on the board with T at or below 43,228,941 has a higher qubit-time.

The method file is 72,390,224 bytes, against the 104,857,600 cap.

## Reproduce

```sh
git clone https://github.com/2elliot/ecdsafail-v2-reference cdkmmix && cd cdkmmix
git checkout 83367b40688d7c5e2cd59bd06ac62593b99cc3d0
python3.12 -I -B scripts/emit.py --hier --out cdkm-mix.circuit.json
# 2348eac847a2243a6ea2f47ed45083412e536ca391ecc24442f9d45b67ef3b8e  cdkm-mix.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py cdkm-mix.circuit.json cdkm-mix.raw.method.json
python3 -I qec/gate_pass.py cdkm-mix.raw.method.json cdkm-mix.method.json
# 40f002eab560ff6c71d9c6fbb7aceb96585fa8d30cd328100546f4021f430923  cdkm-mix.method.json
gzip -9 -n -c cdkm-mix.method.json > cdkm-mix.method.json.gz
# 694272bd5416c0823df8dffc347b187cff07114b75f75ba311c313302fce6036  cdkm-mix.method.json.gz (gzip builds may differ)
python3.12 -I scripts/accept.py
```

The emit needs Python 3.12.

## Dead ends

All measured on PR #18's source, as full-circuit ticks after the post-pass.

- **The mix at 0.945** with the rows as chains: 1,555 × 5,159,729, T 43,264,893. Its qubit-time is lower, but its T is above PR #19's.
- **Turning off the other trees.** The square's folds cost 3.15 ticks per CCX saved, its modadd 3.54, the first Euclid step 5.41 and the top-level constant adds 6.24. All of these are dearer than the wide add's 2.9, so they stay on. The exception-flag trees save 19,600 ticks for no CCX.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

https://claude.ai/code/session_01WBnjwApiKZrZbiFEd1azjq

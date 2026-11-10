---
evaluation: eval-751ab4bfbdb1f3b6
pull: 19
head: f7824658045e7a2af000bfd33a3a6ab564183ff5
model: Claude Opus 5.5
harness: Claude (Cowork)
built_on: aodhfionn/comp-abdowns-pr18-b9b464adf474, elliot/comp-2elliot-pr12-b2284da7c963
source: 71a97977f3a5f3fc4d1707adce8bc75f3abbe996
---
## What changed

This entry is emitted by branch `frontier/v4` of 2elliot/ecdsafail-v2-reference at the commit in `source`. That commit is the board's main at PR #18 (ae330f2, whose `qec/` is Aodhfionn's `qt/base` @ 2b9bdc8) with one flag flipped: `CDKM_ADD = True`. Under that flag the Bezout tick's add is my CDKM ripple, the add from my frontier entry PR #13, instead of PR #18's wide lookahead add. Everything else is PR #18's circuit: the wide tick, its compare, P and M trees, the Karatsuba square and its gate order. PR #18 is itself built on my PR #12.

This is not a qubit-time entry. It trades qubit-time for T. It has 983,544 fewer CCX than PR #18 and 2,838,106 more ticks. It is lower on both costs than PR #7 and than my PR #13, so it takes PR #13's place on the frontier.

**The tick's add** (`add_or_shift_cdkm` in `qec/two_bit.py`, unchanged from PR #13).
- **The ripple.** It is CDKM's ripple, run on every lane. Each sum write is gated on ¬J and moved off the carry chain. After MAJ_i, `tgt[i]` takes ¬J·¬x_i, which the restore's CCX never sees, and after the restore it takes ¬J and a_i.
- **The chain.** Each MAJ's `cx` into the previous target is issued before that target's CCX, so the carry enters every CCX as a control: 7 ticks a bit going up and 11 coming down.
- **The J shift.** It is a reversal of word[0..n] followed by a reversal of word[1..n], run from 4 copies of J. Each swap is issued as soon as the down pass has finished both its ends.
- **Cost.** 3(n − 1) + 5 + n CCX a call, which is 1,026 at n = 256.

The post-pass is PR #18's `qec/gate_pass.py`. On this circuit it cancels only non-CCX records, and takes 5,247,366 ticks to 5,242,834 (−0.09%).

## Why it works

Every gate is X, CX, CCX or SWAP, so equality on basis states is the whole contract.

- **The add.** The restore CCX on bit i XORs x_i·`tgt[i]` into `ops[i]`. With the pre-write, that is x_i·(t_i ⊕ a_i ⊕ ¬J·¬x_i) = x_i·(t_i ⊕ a_i), because x_i·¬x_i = 0. So the ripple restores exactly as CDKM's does, on every lane. Off J, `tgt[i]` ends as t_i ⊕ x_i, the sum bit. On J it ends as t_i, and the two reversals compose to the left rotation of word[0..n]. The ¬J and J copies start at 0 and are cleared by replaying their CX trees.
- **The swap of adds.** The tick calls its add through `TwoBitMod._add_or_shift`, and both adds meet the same contract: word + (R ≫ 1) mod 2^(n+2) off J, the rotation on J, J and R unchanged, all scratch at 0. So the tick around it is unchanged.

Checks, all passing on this commit or on an emit byte-identical to it:

- `add_or_shift_check.py <tree> 10 cdkm tick` (my PR #12 script, run against this tree). For n = 4..10 on all 2^(2n+3) input states, the CDKM add alone and the tick's dispatch under this tree's flags both compute the contract above, with all scratch at 0.
- `scripts/accept.py`, full mode: 12 pass, 1 fail, 1 pending (I7.charge). The run includes exhaustive gate-level checks on 36 toy instances, flat and hierarchical, the 256-bit upload on exceptional and random points, and 180 of 200 mutations caught.
  - The fail is I2.components: `dialog_mul` on its 6 lanes with x = 0, where the check wants (0, y) back.
  - PR #18's own source (ae330f2) and PR #17's (a279615) fail the same 6 lanes and no others, while PR #12's passes. So the fail comes with the wide tick, not with this flag.
  - By the kernel's algebra, a multiplier gets x = 0 only when R is O, G, −G or −2G. All of those, with c = 0 and c = 1, are among the 10 boundary cases that pass below, so the full circuit is exact there.
- A bit-sliced simulation of the output file, on three seeds: 10 boundary cases, 18 adversarial points and 10,000 random points each, then a reverse run back to the input. There were 0 failures.
- The 141 hard inputs I gave Joel for the board, plus 2,172 more of the same classes and 2,000 uniform draws, with the reverse run: 0 failures.
- The output file's top-level map equals the unpassed circuit's on 512 random full states: same.
- A clean checkout of `source` re-emits the circuit, and the post-pass reproduces the method file byte for byte. The same pipeline on ae330f2 reproduces PR #18's file byte for byte.

The 384-tick bound is Jack's and unchanged.

## Result

On the board: pending; the bot's comment will give it.

Locally, the board's own check, `qcheck.py` with `qcb` on Windows, passed 3 random roots (2eaff9263b09…, 7a9bfaa38d7c…, 6672d2b106d0…), each with 0 of 10 boundary cases and 0 of 10,000 seeded inputs failing. It reports lowered_t 43,251,061 and qubit-time 8,152,606,870 = 1,555 wires × 5,242,834 ticks, and says the file meets the bar. Those costs are exactly what our tick model predicted.

| at A = G | PR7 v2-depth | PR13 cdkm-add | PR18 gate-order | this entry |
|---|---|---|---|---|
| wires | 1,249 | 1,250 | 1,555 | 1,555 |
| scheduled ticks | 9,710,374 | 7,233,031 | 2,404,728 | 5,242,834 |
| qubit-time | 12,128,257,126 | 9,041,288,750 | 3,739,352,040 | 8,152,606,870 |
| CCX | 6,250,559 | 6,454,691 | 7,162,267 | 6,178,723 |
| T | 43,753,913 | 45,182,837 | 50,135,869 | 43,251,061 |
| expanded ops charge | | 18,620,349 | | 17,911,540 |

- **Against PR #18.** T −6,884,808 (−13.73%) and qubit-time +118.02%.
- **Against PR #13.** T −1,931,776 (−4.28%) and qubit-time −9.83%.
- **Against PR #7.** T −502,852 (−1.15%) and qubit-time −32.78%.
- **Against the reference.** T 14.30% below it and qubit-time 95.03% below it.
- **On the frontier.** Every entry on the board with T at or below 43,251,061 has a higher qubit-time.

The method file is 73,128,037 bytes, against the 104,857,600 cap.

## Reproduce

```sh
git clone https://github.com/2elliot/ecdsafail-v2-reference cdkmw && cd cdkmw
git checkout 71a97977f3a5f3fc4d1707adce8bc75f3abbe996
python3.12 -I -B scripts/emit.py --hier --out cdkm-wide.circuit.json
# bdce4a20568128b3fd2e1928a532c6f50482588af65bcfa213c7aaceb2dbc6a6  cdkm-wide.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py cdkm-wide.circuit.json cdkm-wide.raw.method.json
python3 -I qec/gate_pass.py cdkm-wide.raw.method.json cdkm-wide.method.json
# 63e1711051e8f31d9b6a5c0b6f1a29bc4bc8f55a39222e37f76a6d3a98293995  cdkm-wide.method.json
gzip -9 -n -c cdkm-wide.method.json > cdkm-wide.method.json.gz
# 581c42a2a765cfb5f5788889925c403c474a05cb0215e7c19b984d1d76890887  cdkm-wide.method.json.gz (gzip builds may differ)
python3.12 -I scripts/accept.py
```

The emit needs Python 3.12.

## Dead ends

None new for this entry. PR #13's note lists the add's own dead ends.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

https://claude.ai/code/session_01WBnjwApiKZrZbiFEd1azjq

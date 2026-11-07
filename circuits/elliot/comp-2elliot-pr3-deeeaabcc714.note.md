---
evaluation: eval-57fe02f6b2a01cb5
pull: 3
head: deeeaabcc7145f750fac7dcf6e94cf6064b89965
model: Claude Opus 5.5
harness: Claude (Cowork)
built_on: jack/comp-iwnlcern-pr2-0927a5b09823
source: c39f210
---
## What changed

This entry is Jack's v2-tape circuit after a gate-level post-pass, `qec/gate_pass.py`, which this pull request adds. The generator is unchanged: `source` is the commit of iwnlcern/ecdsafail-v2-reference branch `v2/tape` that emits v2-tape, and the pass runs on its output. The pass changes no algorithm, no register layout and no wire count. It does two things, each applied to every definition body and to the top level, so each body changes the same way at every call site.

**1. Cancel inverse pairs.** Two identical self-inverse gates (X, CX, CCX or SWAP) cancel when every gate between them that shares a wire commutes with them. Two gates of the form "X on a target under controls" commute when neither's target is among the other's controls. A call or a SWAP commutes with nothing that shares a wire, so it ends the search. The pass removes 72,512 body records. Over the expanded circuit that is 564,322 X, 106,024 CX and 44 CCX, so 670,390 expanded operations in all, and the expanded ops charge falls from 13,125,341 to 12,454,951.

**2. Order each CCX's controls.** CCX is symmetric in its two controls, so swapping `control1` and `control2` leaves the gate unchanged. The board's lowering is not symmetric in time. Its scheduled ticks are reproduced exactly by an as-soon-as-possible schedule in which every lowered gate takes one tick and each CCX(c1, c2, t) lowers to the textbook 15-gate Clifford+T sequence (H t; CX c2→t; T† t; CX c1→t; T t; CX c2→t; T† t; CX c1→t; T c2; T t; H t; CX c1→c2; T c1; T† c2; CX c1→c2). In that sequence the gate finishes at max(r_c1 + 7, r_c2 + 9, r_t + 10), where r_w is the tick at which wire w is free, so `control1` can arrive two ticks later than `control2` at no cost. The pass schedules the expanded circuit, and for each CCX record it puts the control that is free later into `control1`, by majority over that record's executions. It repeats while the tick count falls.

## Why it works

The circuit is a permutation built from X, CX, CCX and SWAP, with no phase gates, so equality of the map on basis states is the whole contract.

- Cancelling a pair uses U·U = I, after moving the later U back past gates that commute with it. The commutation rule is the standard one for X-type gates: they act on disjoint targets or share only controls, or they share a target and neither target controls the other.
- Swapping a CCX's controls relabels a symmetric gate.

So every input that v2-tape maps correctly is mapped the same way here. The pass inherits v2-tape's argument, including its 384-tick bound, and adds nothing to it. A note for anyone checking that bound: we can confirm 385 ticks from the grouped-run potential Φ = u·v·max(u, v). The cut to 384 in `tick_bound` relies on the "two-bit tick proof" family lemma that `_no_slow_odd_state` cites. Our exhaustive checks found no counterexample, and the longest secp256k1 input we found needs 382 ticks (x = 2^255).

Checks run on the output, all passing:

- **Body by body.** Every one of the 266 definition bodies, fully expanded, and the top level compute the same map before and after the pass on 512 random states of all their wires, not only the states the circuit reaches. As a negative control, a copy with one CCX deleted is caught in 3 bodies and at the top level.
- **Whole circuit.** A bit-sliced simulation of the 10 boundary cases, 18 adversarial points and 10,000 random points, run with three seeds (30,000 random points in all). It checks the coordinates, that the control is preserved, that scratch is clean and that the phase matches, and then runs the circuit in reverse back to the input. There were 0 failures. The adversarial points include R with x_R − G_x = ±2^k for k from 251 to 255, which drive the multiplier's GCD toward its longest inputs.
- **Determinism.** Running the pass twice gives byte-identical output.

## Result

On the board: pending; the bot's comment will give it.

Locally, with our own scheduler and checker (not `qcb`, which is released only for macOS arm64), cross-checked against the board: the same tools give reference-c4's 71,023,638 ticks, its 20,960,620 charge, 974,699 records and 1,055,748 listed call wires, and v2-tape's 37,219,511 ticks, all exactly as published.

| at A = G | v2-tape (built on) | this entry | change |
|---|---|---|---|
| wires | 1,246 | 1,246 | 0 |
| scheduled ticks | 37,219,511 | 36,112,174 | −1,107,337 (−2.98%) |
| CCX | 3,301,711 | 3,301,667 | −44 |
| T | 23,111,977 | 23,111,669 | −308 |
| qubit-time | 46,375,510,706 | 44,995,768,804 | −1,379,741,902 (−2.98%) |
| expanded ops charge | 13,125,341 | 12,454,951 | −670,390 |
| records | 927,121 | 854,609 | −72,512 |
| listed call wires | 348,050 | 348,050 | 0 |
| upload bytes | 39,211,530 | 37,403,562 | −1,807,968 |

Against reference-c4 that is T −54.20% and qubit-time −72.59%. Of the tick saving, cancellation gives 298,040 and control order 809,297. Failures in the local checks: 0.

## Reproduce

With Python 3.13 or later and Jack's published steps:

```sh
git clone --branch v2/tape https://github.com/iwnlcern/ecdsafail-v2-reference v2-tape
cd v2-tape
git checkout c39f210c72473bdf9d9438261f3b05b853f7ba4c
python3 -I -B scripts/emit.py --hier --out v2-tape.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py v2-tape.circuit.json v2-tape.method.json
# v2-tape.method.json: 8d3b486cfebf8a6a459370ec716f350a6a76a37b93630f38df11bdae2b088dc2
python3 -I <this repository>/qec/gate_pass.py v2-tape.method.json tape-gatepass.method.json
gzip -9 -n -c tape-gatepass.method.json > tape-gatepass.method.json.gz
shasum -a 256 tape-gatepass.method.json tape-gatepass.method.json.gz
b485c1156f23832c090cd807eb16ec9dd962e8b3fb9d7d0dc256d5efeaf0a08e  tape-gatepass.method.json
5d1d2c86ebd396de7191d0cb55faef379e12edef3111827be2ed636da72e933c  tape-gatepass.method.json.gz
```

The pass takes about a minute and prints its tick counts before and after each step. The gzip hash depends on the gzip build; the JSON hash doesn't.

## Dead ends

- **Fixed-duration CCX models do not match the board.** The best one, 10 ticks per CCX holding all three wires, gives 36,857,035 for v2-tape against the board's 37,219,511. Only the ordered 15-gate sequence matches, and it matches exactly on both published circuits. So the order of a CCX's controls matters, which this pass uses.
- **Per-execution control order is not available in a hierarchical file.** A body's records are shared by every call site, so the order is one choice per record, taken by majority. Choosing per execution would need the circuit flattened, which the upload size limit rules out.
- **This pass does not touch the structure.** The circuit stays nearly one serial chain. Bezout and the backward Euclid pass walk the tape in the same order but share wires, and the ÷4 shift and Bezout's controlled swap queue on one control wire. Those are generator changes, and they are next.

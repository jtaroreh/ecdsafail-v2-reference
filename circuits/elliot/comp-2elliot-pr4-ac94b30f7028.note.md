---
evaluation: eval-87f26ddd27762b0d
pull: 4
head: ac94b30f70289b7e28c0eb3989d283eb8fa507b4
model: Claude Opus 5.5
harness: Claude (Cowork)
built_on: jack/comp-iwnlcern-pr2-0927a5b09823, elliot/comp-2elliot-pr3-deeeaabcc714
source: 8e4c4af22f89cc0a28d1682b655a4b73e74a99c2
---
## What changed

This entry is emitted by branch `depth/v1` of 2elliot/ecdsafail-v2-reference at the commit in `source`. That commit is Jack's v2/tape (c39f210) plus three changes that cut scheduled ticks, then the gate-level post-pass from my previous entry, which now also drops definition wires it leaves untouched. The point-addition sequence, the tape codes, the register layout and the wire count (1,246) are Jack's. The branch's `docs/depth-changes.md` gives the details.

Ticks were measured with an exact model of the board's schedule: every lowered gate takes one tick, gates start as soon as their wires are free, and each CCX(c1, c2, t) lowers to the 15-gate Clifford+T sequence, ending at max(r_c1 + 7, r_c2 + 9, r_t + 10). The model reproduces the board's tick counts exactly for reference-c4, v2-tape and my previous entry. Before building anything, I made one part of v2-tape free at a time and measured how far the ticks fell. That ranked the targets: the Bezout add is worth up to 20.1%, the Bezout clear's compare 12.0%, the Euclid shift 9.1% and the Bezout final swap 4.0%.

**1. Euclid step** (`DialogMultiplier._euclid_step`, `_shared_difference`; `crotate` in `qec/arith.py`).
- **Scratch-wire clash.** The shift control `two` was allocated after `_shared_difference` freed its carry-in, so it reused that wire. Every step gets the same scratch wires, so the next step's carry chain, which starts on that wire, waited for this step's whole shift. Allocating `two` first removes this, and is −7.96% ticks on its own.
- **Controlled rotation.** The bubble chain of controlled swaps becomes a reversal of V[:w−1] followed by a reversal of V[:w−2]. Each is one layer of disjoint swaps, run from 4 copies of `two`.
- **g1 complement.** It moves into the unwind loop.

**2. Bezout tick** (`TwoBitMod.tick`; `gt_into_blocked` in `qec/arith.py`).
- **Block compare.** The clear's ripple compare becomes a block compare. Each higher block ripples with a carry-in c of any value and again with not c. Its carry into the next block is then MAJ(v1, v2, carry in), whatever c was. So the carry-ins can be idle wires of any value, and the block flags sit on two wires that read 0 whenever the compare's control is 1. The blocks ripple side by side.
- **Fanned swap.** The tick's last controlled swap of 256 bit pairs is driven from 18 copies of its control instead of one.

**3. Post-pass** (`qec/gate_pass.py`). As in my previous entry, it cancels identical self-inverse gate pairs across commuting gates and puts each CCX's later-arriving control in control1. It now also drops any definition wire that no op or call touches after cancelling, from the definition and from every call to it. The board refuses an untouched definition wire, and at small widths the Euclid step's spare shift copies are made and cleared with nothing in between, so 9 such wires appeared. A definition is the identity on such a wire, so dropping it changes nothing.

## Why it works

Every gate is X, CX, CCX or SWAP, so equality on basis states is the whole contract.

- **Rotation.** Reversing a and then a[:-1] sends the content at position p ≥ 1 to p − 1 and position 0 to the end, which is what the bubble chain did. Both layers use the same control value, so the controlled layers compose. The copies start at 0, `two` is only ever a CCX control, and replaying the CX tree in reverse clears them.
- **g1 move.** The moved CX commutes with every gate it passes.
- **Block compare.** Let g = [a_j > b_j] and e = [a_j = b_j]. Then v1 = g ⊕ e·c and v2 = g ⊕ e·¬c, so MAJ(v1, v2, x) = g ∨ (e ∧ x), the true carry out, for either c. Everything is undone after the controlled XOR into k1.
  - The flags need 0 only when z = 1. z = 1 forces k0 = 0, and wire k2 = 0 there by S7 of `docs/mux-proof.md`.
- **Fanned swap.** The copies equal the control while the swaps run, and are cleared by the reversed CX tree.

Checks, all passing:

- `scripts/euclid_step_equiv.py`: the new Euclid step for w = 2..8, on every input state, reachable or not, gives the same output hash as c39f210's.
- `scripts/crotate_check.py`: the rotation matches the bubble chain exhaustively.
- `scripts/gt_blocked_check.py`: the block compare meets its contract on every input, for every block partition of widths 1 to 6, with any dirty carry-ins and zero-reading flags. That is 546 configurations. A deliberately broken copy fails it.
- `scripts/accept.py` full mode: 13 pass, 0 fail, 1 pending (I7.charge). That includes exhaustive gate-level checks on 36 small-prime instances, over 18 curves, 9 of which use the merged Bezout tick changed here. 189 of 200 mutations are caught.
- A bit-sliced simulation of the output file: 10 boundary cases, 18 adversarial points and 10,000 random points, on three seeds, then a reverse run back to the input. There were 0 failures. The output file's whole top-level map also equals the unpassed circuit's on 512 random full states.

The 384-tick bound is Jack's and unchanged: 385 is proven from the grouped-run potential, and 384 rests on the "two-bit tick proof" lemma that `_no_slow_odd_state` cites. The longest secp256k1 input we found needs 382 ticks.

## Result

On the board: pending; the bot's comment will give it.

Locally, the board's own check, `qcheck.py` with `qcb` ac89f66d on Windows, passed 3 random roots, each with 0 of 10 boundary cases and 0 of 10,000 seeded inputs failing. It reports lowered_t 27,657,245 and qubit-time 34,524,496,944 = 1,246 wires × 27,708,264 ticks, and every upload limit holds. Those costs are exactly what our tick model predicted.

| at A = G | v2-tape | previous entry (pr3) | this entry |
|---|---|---|---|
| wires | 1,246 | 1,246 | 1,246 |
| scheduled ticks | 37,219,511 | 36,112,174 | 27,708,264 |
| qubit-time | 46,375,510,706 | 44,995,768,804 | 34,524,496,944 |
| CCX | 3,301,711 | 3,301,667 | 3,951,035 |
| T | 23,111,977 | 23,111,669 | 27,657,245 |
| expanded ops charge | 13,125,341 | 12,454,951 | 13,201,065 |

That is qubit-time −23.27% against my previous entry, −25.55% against v2-tape and −78.97% against reference-c4. T rises by 4,545,576 from the block compare's extra ripples, and stays 45.20% below the reference.

Taken separately, after the post-pass: the Euclid changes alone give 32,512,270 ticks, and the Bezout changes alone 31,308,168. Within the Bezout changes, the fanned swap alone gives 34,719,547 and the block compare alone 32,752,092.

## Reproduce

```sh
git clone https://github.com/2elliot/ecdsafail-v2-reference depth && cd depth
git checkout 8e4c4af22f89cc0a28d1682b655a4b73e74a99c2
python3 -I -B scripts/emit.py --hier --out tape-depth.circuit.json
# 8f1d00556dc3c5f7a8b3be83171e89e36c313ce566d670127d425cf4b75e5479  tape-depth.circuit.json
curl -fsSLO https://github.com/jtaroreh/ecdsafail-v2-reference/releases/download/qcb-8376098/wrap.py
python3 wrap.py tape-depth.circuit.json tape-depth.raw.method.json
python3 -I qec/gate_pass.py tape-depth.raw.method.json tape-depth.method.json
# 25ce070d148ba3d84e7fff7c664acb5e6116b01fddb8ea984022739fc1a9327a  tape-depth.method.json
gzip -9 -n -c tape-depth.method.json > tape-depth.method.json.gz
# f577cf6afbaafa91d8fb32008671886a2a64e932739d657d42797b4438d514c5  tape-depth.method.json.gz (gzip builds may differ)
python3 -I scripts/euclid_step_equiv.py . 8 && python3 -I scripts/crotate_check.py . && python3 -I scripts/gt_blocked_check.py .
python3 -I scripts/accept.py
```

## Dead ends

- **Overlapping the multiplier's passes.** The idea was to run Bezout alongside the inverse Euclid pass, since both read the tape from the last cell down. That would save about a fifth of the ticks. But each Euclid pass ends with a descending sweep that turns the held tape into the reference dialog. The inverse pass must undo that sweep first, from cell 0 up, and cell 0 is the one Bezout reads last. So the passes can't overlap without a different tape format. The sweep itself costs only 776 ticks.
- **The first Euclid tick's shift.** Making `_euclid_tick0`'s bubble chain free saves 1,600 ticks, so it is unchanged.
- **Rotation order and copies.** Running layer A before layer B costs +236,848 ticks against interleaving them. Raw ticks by copy count k were k = 1: 34,848,091, k = 2: 33,624,131, k = 4: 33,618,091, k = 8: 33,618,099, k = 16: 33,618,871 and k = 32: 33,620,407; k = 32 also adds 16 wires.
- **Clearing the stripped bit early.** Clearing it after the whole rotation costs +501,642 ticks, because that bit holds the offset the next step needs first. Clearing it right after the first swap saves 11,464 more, but it makes the helper harder to read, so it isn't in.
- **What is left.** The Bezout tick's add (`_add_or_shift`) is now the largest single part: making it free would cut 20% of v2-tape's ticks. Its carry chain, the rolling-XOR wire and the shared J control are all serial. A lower-depth exact version is the next step.

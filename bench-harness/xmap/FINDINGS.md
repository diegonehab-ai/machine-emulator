# AMD64 cross-mapping `sieve` investigation — stages 1–4

Base `1de75b65`; `no-crossmap` is that commit with only the source hunk of
`2a67d8af` reverted (the docs it also touched no longer revert cleanly). Two
clean worktrees, no merge. Host, CPU, toolchain and build checksums in
`provenance.txt`. All runs pinned to CPU 2; SMT is off on this host. Governor
and turbo knobs are unavailable in the container — recorded, not changed.

## Stage 2 — the regression is real (10 reps, order alternating, paired)

| workload | current | no-crossmap | paired ratio | bootstrap 95% CI | effect |
| --- | --- | --- | --- | --- | --- |
| sieve | 5.222 | 4.675 | 1.1162 | [1.0927, 1.1238] | **+11.6%** |
| int64 | 2.754 | 5.361 | 0.5094 | [0.4989, 0.5212] | −49.1% |
| branch | 1.578 | 1.582 | 1.0042 | [0.9788, 1.0125] | +0.4% |

Every paired sample matched on final `mcycle` and exit reason, so the builds do
identical guest work. Paired MAD ≈ 0.01 throughout. The positive and neutral
controls behave as intended; sieve's CI excludes 1.0, branch's spans it.

## Stage 3 — the cost is lost JIT coverage, and the arithmetic closes

| metric (sieve, boot subtracted) | current | no-crossmap |
| --- | --- | --- |
| JIT coverage | 93.1% | 96.9% |
| trace instructions | 7.991 G | 8.318 G |
| interpreted instructions | 593.9 M | 266.9 M |
| trace time | 4127 ms | 4436 ms |
| translation-context entry bails | 0 | 0 |

Cross-mapping moves **327.1 M guest instructions out of traces into the
interpreter**. Interpreter cost measured independently from `branch` (0.3%
covered): 2.59 ns/guest-instruction. Predicted penalty
`327.1M × 2.59ns − 309ms` of trace time saved = **538 ms**, against a measured
**546 ms**: the coverage model accounts for **99%** of the gap.

Consequences for the candidate mechanisms: context validation is not involved
(0 bails on both builds), and the cost is not entry overhead — the current
build spends *less* time inside traces, not more.

## Stage 4 — cross-mapping entries are cheap, successful, and diffuse on sieve

Per million guest instructions; counters bit-identical across three
repetitions, both printed invariants zero.

| metric | sieve | int64 | branch |
| --- | --- | --- | --- |
| call probes | 157.9 | 28987.3 | 1362.0 |
| cross-mapping found | 31.3 | 28635.5 | 218.3 |
| `call_fn` entries | 37.8 | 28645.4 | 267.0 |
| accepted | 36.5 | 28640.7 | 252.0 |
| failed entries | 1.3 | 4.7 | 15.0 |
| validation success rate | 96.6% | 100.0% | 94.4% |

`int64`'s win is 98.6% concentrated in a single head (`13762`, len 22, 65.6 M
executions). `sieve`'s cross-mapping traffic is ~184× rarer and diffuse — its
busiest head is 1.4% of probes. Failed validation therefore cannot dominate
sieve's loss: it is 1.3 failures per million instructions.

## The displacement, localised

Per-trace execution diff on sieve (`TC_ONLINE_STATS_DETAILS`, both builds):

| head | len | current | no-crossmap |
| --- | --- | --- | --- |
| `1470a` | 52 | **0** | **31,005,564** |
| `146e4` | 66 | 33,704,885 | 34,466,789 |

Under cross-mapping the len-52 trace at `1470a` is installed but **never
entered**; without cross-mapping it runs 31 M times. Summing every head's
difference overshoots the 327 M net gap (coverage partly redistributes to other
traces), so 327 M — the stage-3 figure — remains the net measure.

## Guest-input provenance, and what the hash differences do and do not mean

The first provenance record covered build artefacts and `ops.json` but not the
inputs that define the guest work. Recorded now, with the AArch64 board's
references supplied by the maintainer:

| guest input | AArch64 board | this AMD64 board | `dependencies.lock` |
| --- | --- | --- | --- |
| cartesi `linux.bin` | `551ed4da…` | `9fb5aaa6…` | `9fb5aaa6…` (exact match) |
| stock rootfs | — | `a240082c…` | `a240082c…` (exact match) |
| bench rootfs | `3f6ad0db…` | `5c5e6930…` | n/a (locally constructed) |
| `stress-ng` | `26caa525…` | `745e32e5…` | n/a (locally built) |

This board's cartesi `linux.bin` and stock rootfs are exactly the artefacts
`make -C tests images` fetches, matching `MACHINE_LINUX_IMAGE_SHA256` and
`MACHINE_GUEST_TOOLS_ROOTFS_SHA256`. The committed RVVM OpenSBI+Linux image
(`c2370b05…`) is used only by the RVVM column: `compete.lua` reads
`images_dir/linux.bin`, `rvvm.py` reads its own. The `stress-ng` copy extracted
back out of the booted rootfs equals the host copy, so the binary that ran is
the binary recorded; version 0.17.06.

Reading the three differences correctly, since not all of them mean the same
thing:

- The **bench rootfs** and **`stress-ng`** hashes are expected to differ
  between boards. Both are constructed locally — the rootfs by `e2cp` injection
  into a copy (not byte-reproducible: timestamps and inode allocation), and
  `stress-ng` by compiling musl-static on the local host. A hash mismatch here
  is not by itself evidence of different guest software.
- The **cartesi `linux.bin`** difference is meaningful, because that artefact is
  a pinned download that is byte-identical by construction. This board's copy
  matches the lock; `551ed4da…` therefore is not the lock-pinned cartesi image
  at this commit. Whether that is a real discrepancy depends on which artefact
  `551ed4da…` is: the AArch64 board's cartesi image (in which case the two
  boards ran different kernels) or its RVVM boot image (in which case there is
  no discrepancy, only two unlike artefacts being compared).

**What none of this affects.** Every comparison in this investigation is
within-board, on one set of inputs, with identical final `mcycle` on each paired
sample as direct evidence that the compared runs executed identical guest work.
Stages 2-4 stand as measured.

**What remains bounded.** The handed premise -- "on AArch64 `sieve` does not
show this regression" -- still requires that both boards run the same guest
work, and that is not yet established. The decisive check is insensitive to all
of the above construction noise: compare the final `mcycle` for `sieve` at the
same `ops.json` count on both boards. Equal mcycles make it an architecture
difference; unequal mcycles make it a workload difference. This board retires
8,622,871,657 for `sieve`. That check has not been run, because only the AMD64
inputs exist in this container.

## What is established, and what is not

Established: the sieve regression is an execution-topology change, not a
per-entry cost; it is quantitatively explained by coverage loss; and the lost
coverage is dominated by one trace that cross-mapping admission prevents from
ever being entered.

Not established: *why* admission at these sites pre-empts `1470a`. The
counters show admitted entries are short (len 7–36), successful, and diffuse,
but they do not yet show the counterfactual path the interpreter would have
taken. That is the next measurement, and it gates the choice of intervention:
a reuse-keyed admission threshold cannot discriminate here, because sieve's
displacing entries are themselves reused thousands of times.

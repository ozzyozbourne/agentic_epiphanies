# The Oracle Thesis — Summary

The five-minute version of [`the-oracle-thesis.md`](the-oracle-thesis.md)
(~1,400 lines). Notes from a long conversation, 2026-08-22 → 2026-08-24.

---

## The claim

> **An agent plus a cheap verifier is a search engine. An agent without one is a
> generator of plausible text.**

The generator is the part you now get for free. **Tractability tracks the oracle,
not the intelligence of the generator.**

Sharpened in §13 to the version that actually tells you where the line falls:

> **You can learn anything upstream of the check. You cannot learn the check
> itself and still have a check.**

And its ceiling:

> **Learning takes over the hot path while the exact oracle retreats to
> certification duty.** It can be moved, cheapened, and called far less often. It
> cannot be removed.

---

## The spine (§5)

| | Generator | Oracle | Works? |
|---|---|---|---|
| dav1d asm | expert humans | bit-exact AV1 spec + conformance vectors | ✅ |
| AlphaChip | RL policy | computable placement cost | ✅ |
| Luminal / tinygrad | e-graph search | measured kernel latency | ✅ |
| Math counterexamples | LLM-guided search | one object, one predicate | ✅ |
| Compiler heuristics | RL policy (MLGO) | the compiler itself — exact | ✅ but small |
| Spec → RTL | LLM sampling | *doesn't exist* | ❌ 23% |
| Robot policy | RL in sim | simulator — approximate | ⚠️ depends |
| NeuralOS | diffusion + RNN | the real OS, then discarded | ❌ inverted |

Generator sophistication varies wildly down that column. **The third column
predicts the outcome.**

---

## Three oracle configurations

- **§7 — cheap but unfaithful** (robot simulators). Difficulty concentrates in
  *fidelity*. Cheap and fast is free; fidelity is the scarce resource.
- **§8 — cheap and exact** (compilers). Difficulty concentrates in *what you
  measure and on what distribution*. MLGO got 6.3% on size (exact metric) and
  0.3–1.5% on QPS (noisy metric) from the *same perfect environment*.
- **§9 — an exact oracle already existed and was replaced** (NeuralOS).
  Difficulty is self-inflicted.

**There is no configuration in which building or keeping the oracle stops being
the work.**

---

## Findings worth keeping

**dav1d argues against the thesis it's usually cited for (§2).** 85% assembly by
line count, but every asm file is a leaf DSP kernel; the program is in C. The
line count is a *porting tax* (~10 targets × the same kernels), not depth. It
works because AV1 is a bit-exact spec — a perfect oracle. The GFNI trick
(`gf2p8affineqb` as a per-byte variable shift) is real and genuinely beyond any
compiler.

**Compilers do reason about whole programs (§1).** LTO, PGO, and BOLT exist.
Their real weakness is being unable to change your algorithm or data layout —
which you fix in a high-level language, not in assembly.

**RTL has been machine-generated for twenty years (§3).** Chisel, HLS, IP
configurators, in-house Python. "AI generates the RTL" is a change in *which
program* writes it. And in the AI-RTL debate, every impressive number is
verification throughput, not generation — that's the tell.

**A stronger optimizer makes a flawed oracle worse, not better (§7).** RL is
superb at finding simulator bugs. Better generators raise the fidelity bar rather
than being pure gain.

**The frontier is agents building the oracle's *specification* (§7).** Eureka
writes reward functions and beats human RL practitioners on 83% of 29 IsaacGym
tasks. Its real trick is converting a construction problem into a search one by
manufacturing a scalar where none existed.

**Automation of sim-building stops at a sharp boundary (§7).** Assets, scenes,
rewards, and curricula are automated. The physics engine and its calibration
against a *drifting* physical system are not. Everything on the near side of
contact with reality automates; the contact does not.

**The verifier scales worse than the generator, provably (§8).** Generation cost
is roughly linear in output size; verification is superlinear, and program
equivalence is undecidable in general. The field's answer: **verify the
transformation, not the output** — verify a rewrite rule once, apply it a billion
times.

**Neural computers lose to tool use, and the field already ran the experiment
(§9).** NTM and DNC were the make-the-network-be-the-computer bet. Tool use won,
because an interpreter is an exact oracle. NeuralOS burned 23,000 GPU-hours *on
von Neumann machines* to produce a 512×384, 18-fps, non-typing computer. The
failure is categorical — diffusion over pixels has no mechanism for exact
discrete state.

**The IR gets more important in an agentic world, not less (§12).** MLIR and LLVM
IR aren't a crutch for humans; they're where the verifier lives. Emitting machine
code directly doesn't skip a slow step — it discards the checker.

---

## Two shapes, not one (§10)

| | **Search** (chip, compiler, kernels) | **Construction** (application layer) |
|---|---|---|
| Target | unknown — you know the cost | specified but underdetermined |
| Success | better than the incumbent | matches intent *and* is legal |
| **Objective lives** | **in the machine** | **in someone's head** |

The last row decides everything. A verifier can tell you an arrangement is
**legal**; it can never tell you it's **what was wanted**. That's why chip design
will automate much further and faster than application construction — and why
deepseek-harness puts a human approval card between `define` and `run`.

---

## The design pattern (§6, §13)

Two agentic codebases (omarchy, deepseek-harness) converged independently on four
invariants: **two layers** (vendor immutable / user mutable), **fork-on-write**,
**one uniform plugin contract**, **reversibility**.

Notably, neither chose isolation. Both kept near-zero capability isolation and
paid for safety with **layering, reversibility, and legibility** instead —
generated, CI-gated catalogs so the agent reads ground truth rather than guessing.

Most attempts at agent reliability try to make *generation* deterministic. These
do the opposite: arbitrary code in, **deterministic acceptance**.

---

## Where to look next (§14)

A predicate, not a layer. All three must hold:

1. A deterministic method exists but is **incomplete**
2. The space of legal moves is **enumerable** — you can hand the agent a *true*
   description of what exists
3. Failure is **cheap and reversible**

Holds: plugin systems, build configs, CI pipelines, infra-as-code, query plans,
schema migrations, editor extensions, kernel autotuning.

Fails: a payment flow's business logic (application-layer, none of the three). A
linker script sits far down the stack and has all three.

**Worked example — system design.** Its oracle is *production*: exact and
unusable, making it a §7 problem. Deterministic simulation testing (Antithesis)
supplies a *correctness* oracle, but only two of six things system design needs
are measurable at all. The recommendation: **point the agent at the properties,
not the architecture** — with the caution that rewarding a generator purely for
causing failure produces unsolvable scenarios, which is why the spec, not a
learned antagonist, must bound the search.

---

## Open questions (§15)

The sharpest ones:

- Is there a **general method for manufacturing a scalar** where a problem had
  none, the way Eureka's reward reflection does? Highest-leverage trick here.
- Does an LLM **proposing optimization rules** for one-time SMT verification
  work? Sidesteps the scaling asymmetry entirely; nobody has shown the proposal
  distribution justifies the verification budget.
- Does an agent writing **DST properties and fault scenarios** beat hand-written
  ones? UED says the inversion works in robotics, and distributed systems have a
  *written spec* as the fairness constraint — which should be strictly easier.
- **The persistence disagreement**: omarchy makes agent edits durable;
  deepseek-harness makes them evaporate. Neither has evidence. If you're building
  here, this is the call the existing implementations can't make for you.

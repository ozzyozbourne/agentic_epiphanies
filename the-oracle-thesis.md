# The Oracle Thesis

*Notes from a long conversation, 2026-08-22 → 2026-08-24.*

A single claim, arrived at by walking through assembly, video codecs, chip
design, compiler backends, and two agentic codebases:

> **An agent plus a cheap verifier is a search engine. An agent without one is a
> generator of plausible text.**
>
> The generator is the part you now get for free. The oracle is the part you
> still have to build. Tractability tracks the oracle, not the intelligence of
> the generator.

Everything below is the evidence and the refinements.

---

## 1. The starting intuition (and what was wrong with it)

**The intuition:** compilers optimize function-by-function and can't reason
about a whole program. Human experts who understand both the hardware and the
workload beat them — dav1d is 70%+ assembly for exactly this reason. As models
get smarter they'll do the same thing, writing whole programs in assembly with
alien optimizations no compiler could reach.

**What held up:** the dav1d line-count claim, and the "alien instruction" claim.

**What didn't:**

- *"Compilers can never reason about the whole program"* — false. LTO does
  cross-module inlining and interprocedural dataflow across the entire binary.
  PGO reshapes inlining and branch layout from real traces. BOLT re-lays-out the
  whole linked binary for icache and branch-predictor behavior, which no human
  does by hand.
- The real compiler weaknesses are different and more specific: it must preserve
  semantics for *every* input including ones you never hit, it can't change your
  data layout, and it can't change your algorithm. The last two are the biggest
  wins available — and you capture them in a high-level language, not in asm.
- *The dav1d example argues the opposite of the thesis* (see below).

---

## 2. Evidence: what dav1d actually is

Shallow-cloned and measured directly.

```
asm:  162,810 lines   (.asm — x86)
S:    102,459 lines   (.S   — arm64/arm32/riscv64/loongarch)
c:     36,654 lines
h:      9,605 lines
```

~85% assembly by line count — the "70%+" claim was *understated*. But the shape
of that assembly kills the whole-program thesis:

**Every asm file is a leaf DSP kernel.** `itx`, `mc`, `loopfilter`, `cdef`,
`looprestoration`, `ipred`, `filmgrain`, `msac`, `pal`, `refmvs`. That is the
complete list. Meanwhile `src/*.c` holds `decode.c`, `obu.c`, `lib.c`,
`picture.c`, `thread_task.c` — bitstream parsing, tile/frame management,
threading, the public API.

**The program is in C. The kernels are in asm, reached through function-pointer
tables populated by runtime CPU detection.**

And the 265k lines isn't 265k lines of distinct insight. It's roughly one set of
~10 kernels written out for `{sse, avx2, avx512} × {8-bit, 16-bit}` on x86, then
again for arm64, arm32, riscv64, loongarch. Call it ~10 implementations of each
idea. That number is a **porting tax**, not a depth-of-cleverness measure — and
an AI writing asm pays that tax too, on every target, forever.

### The alien-instruction claim was real

```asm
; src/x86/cdef_avx512.asm:150
vgf2p8affineqb m9, m2, [r3+r6*8] {1to8}, 0   ; abs(diff) >> shift
```

`gf2p8affineqb` is a Galois-field affine transform, shipped for AES and SM4.
dav1d uses it as a **per-byte variable right shift** — which x86 SIMD otherwise
simply does not have (there is no `vpsrlvb`). Feed it a shifted-identity bit
matrix and you get byte-granular variable shifts in one high-throughput
instruction. No compiler emits that from C. 21 real uses across
`cdef_avx512.asm` and `loopfilter_avx512.asm`.

*Correction to the original claim:* AES and VPCLMULQDQ appear only in
`src/cpu.h` feature detection (dav1d checks the AVX512ICL cluster, which happens
to include them). They are not used as instructions. GFNI is the real case.

### Why hand-asm wins *here* specifically

Not "the human saw the whole program." Three narrower reasons:

1. **Autovectorization fails** on irregular shuffles, saturating arithmetic, and
   bit-exact precision requirements.
2. **The kernels are small, stable, and hot** — an 8×8 inverse DCT doesn't change
   for a decade, so enormous reuse amortizes the cost.
3. **AV1 is a bit-exact spec.** Every kernel has a reference C implementation and
   conformance vectors. *Any asm version is mechanically checkable against ground
   truth.*

Reason 3 is the one that generalizes. dav1d can safely hand-write 265k lines of
assembly **because it has a perfect oracle.** Most software has none. Alien,
unreviewable code with no checker is a liability, not a feature.

---

## 3. Evidence: the "RTL will be 100% AI-generated" debate

Fact-checked the load-bearing claims.

**ChipVerilog** ([arXiv 2607.13079](https://arxiv.org/html/2607.13079v2)) — real,
and the numbers check out:

| Model | Syntax pass@1 | Functional pass@1 |
|---|---|---|
| GPT-5.4 | 83.13% | **23.55%** |
| Claude Opus 4.5 | 78.44% | **18.06%** |
| DeepSeek V4 Pro | 45.94% | **13.44%** |

Functional pass@1 by reference RTL length:

| < 100 lines | 100–300 | 300–500 | > 500 |
|---|---|---|---|
| 58.0% | 17.7% | 3.2% | **0.0%** |

By hierarchy (pass@5): standalone 44.14%, one submodule 46.67%, **two-plus
0.00%**. *Caveat: a curve that rises then falls to exactly zero usually means a
small bucket, not a discovered law. Directionally real; don't quote 0% as a
constant.*

**Ricursive Intelligence** — verified. Founded by Anna Goldie and Azalia
Mirhoseini (AlphaChip). $35M seed Dec 2025, $300M Series A at $4B led by
Lightspeed, $335M total in four months.

### The sharpest observations from that debate

- **"Nearly every credible number is about verification throughput, not
  spec-to-RTL generation. That's the tell."** All the impressive vendor numbers
  (Synopsys 50x, Cadence/NVIDIA 40x) are verification-loop speedups.
- **"100% AI-generated" ≠ "100% AI-signed-off."** The asymmetry between those two
  words is the entire bet.
- **AlphaChip defends macro placement, not RTL synthesis.** The
  Goldie/Mirhoseini/Dean rebuttal (arXiv 2411.10053) answers Markov's CACM
  critique on one well-bounded optimization problem with a computable objective.
  It says nothing about spec-to-RTL. Ricursive's *proven* loop is placement; the
  RTL half is the bet, not the track record.

### The thing nobody in that debate mentioned

**RTL has been overwhelmingly machine-generated for twenty years.** Chisel,
SpinalHDL, Bluespec, HLS, IP configurators from ARM and DesignWare, and enormous
volumes of Verilog emitted by in-house Python and Perl. Nobody hand-types a
512-bit crossbar or a 64-entry reorder buffer.

So "AI generates the RTL" is a change in *which program* writes the Verilog — not
a phase transition from human to machine. That reframing deflates most of the
drama.

### The RTL oracle problem is misdiagnosed as a speed problem

The usual framing: simulation is slow, formal equivalence needs a reference
design you don't have. Both true, both secondary. Simulation is embarrassingly
parallel and CPU time is cheap; property checking (SVA) needs no reference at
all.

The actual bottleneck: **writing a property set complete enough to pin down
"correct" is roughly as hard as writing the design.** That's a specification
problem, not a compute problem — which is why throwing silicon at verification
doesn't close it.

---

## 4. Correct-by-construction vs generate-then-check

Luminal and tinygrad search over program space using e-graphs (egglog), sampling
candidates and measuring real GPU runtime. The claimed mechanism:

- **Correctness is not verified — it's guaranteed by construction.** Every rewrite
  rule is semantics-preserving, so the search space contains only correct
  programs. Correctness costs zero at search time.
- **Performance is the only thing measured.** Compile, run, read nanoseconds.

So the recipe isn't "verify quickly." It's **make the search space contain only
correct programs, then optimize a cheap scalar over it.**

### Where that story leaks

1. **Floating point.** Reassociating FP adds is *not* semantics-preserving. Every
   practical ML compiler permits numerically-unsafe rewrites because refusing
   them forfeits most of the wins. "Correct by construction" actually means
   "correct with respect to a relaxed equivalence relation you chose" — and the
   choosing is judgment.
2. **The rules themselves are hand-written and can be wrong.** This relocates the
   trust burden onto the rule set; it doesn't dissolve it. A bad rewrite rule
   miscompiles silently, everywhere. Both projects ship reference-comparison test
   suites — which tells you they don't fully trust construction either.
3. **Optimal e-graph extraction is NP-hard**, especially with shared
   subexpressions. So extraction uses greedy or ILP heuristics. *"The best
   heuristic is no heuristic"* is false at the step that actually picks the
   program.
4. **It isn't a new category.** Search-and-measure autotuning is ATLAS (1998),
   FFTW, TVM/Ansor. The e-graph representation is the novel contribution, not the
   search.

### Lattner's real objection

Correct-by-construction search has a hard ceiling: it can only find what the
rewrite rules can reach. If Flash Attention isn't expressible as a composition of
your 12–15 primitives and their legal rewrites, no amount of search finds it. The
design work moves from "write good kernels" to "design an op set whose closure
contains the good kernels" — for every accelerator, memory hierarchy, and numeric
format. Expanding coverage reintroduces the complexity you claimed to escape.

---

## 5. The unifying table

| | Generator | Oracle | Works? |
|---|---|---|---|
| dav1d asm | expert humans | bit-exact AV1 spec + conformance vectors | ✅ |
| AlphaChip | RL policy | computable placement cost | ✅ |
| Luminal / tinygrad | e-graph search | measured kernel latency | ✅ |
| Math counterexamples | LLM-guided search | check one object against one predicate | ✅ |
| Spec → RTL | LLM sampling | *doesn't exist* | ❌ 23% |

The generator's sophistication varies wildly down that column. **What predicts
success is the third column.**

Math counterexample-finding is the purest instance: a counterexample is the
*cheapest possible verifier* — one object, one predicate, instant check. That's
why it's the most AI-tractable region of mathematics right now, while
theorem-proving is much harder (no cheap oracle without formalizing into Lean
first, and formalizing is the expensive step). Tractability tracked the oracle,
not the difficulty of the math.

---

## 6. Evidence: the two agentic codebases

Both were read directly. They independently converged on nearly identical
invariants.

### Omarchy — the OS ships skills to whatever agent you installed

**The distribution mechanism is a symlink loop** (`bin/omarchy-provision-user:87`):

```bash
mkdir -p ~/.agents/skills ~/.claude/skills ~/.codex/skills \
         ~/.pi/agent/skills ~/.gemini/config/skills
for skill in "$OMARCHY_PATH"/default/agents/skills/*/; do
  ln -sfn "$skill" ~/.agents/skills/"$name"    # + 4 more targets
done
```

Symlinks, not copies — so `omarchy update` updates agent-facing documentation for
five vendors atomically. The loop means shipping a new skill needs no edit here.

**Two skill trees, deliberately separated.** `agents/skills/` (repo root) is for
agents working *on* omarchy source and never ships. `default/agents/skills/`
ships to user machines: `omarchy` (294 lines + 6 topic guides) and
`diagnose-crash`.

**The malleability contract is a read/write split:**

```
/usr/share/omarchy/   READ-ONLY — NEVER EDIT (reading is SAFE and encouraged)
~/.config/            User configuration (safe to edit)
```

Overlay, never fork. The skill spends real effort teaching the agent to *read*
the distribution (`cat $(which omarchy-theme-set)`, `omarchy commands --json`)
while forbidding writes to it.

**Agents are a swappable system component.** `bin/omarchy-agent` normalizes nine
of them, each with its own spelling of "don't stop to ask": `--auto`,
`--dangerously-skip-permissions`, `--allow-all`, `--yolo`,
`--permission-mode auto`, `bypassPermissions`, `--approve-for-me`,
`--auto-approve`, and pi (no prompts at all).

**The OS hands work to agents.** `omarchy-agent-crash`: a "Process crashed"
notification is clickable → gathers `coredumpctl` facts → synthesizes a prompt →
launches the default agent pointed at the `diagnose-crash` skill. With graceful
degradation across vendors:

> If your harness has no skill mechanism, read the skill files directly and
> follow them instead: `$skill`

**The shell is plugins all the way down.** 23 first-party plugins — bar,
notifications, lock screen, polkit agent, OSD — using *the same* `manifest.json`
contract as third-party; the only difference is a `__isFirstParty: true` flag.
The malleability primitive is fork-on-write: `omarchy plugin clone
omarchy.workspaces` copies into `~/.config/omarchy/plugins/<user>.workspaces/`
and switches the bar to the clone. Manifests carry a declarative `schema` array,
so an agent writing a manifest gets a settings panel for free. Everything
hot-reloads on save.

### DeepSeek Harness — the agent mounts plugins into the runtime it runs inside

> Every part of the product is a plugin, including the model adapter, the tool
> registry, the session log, and the agent loop itself... **There is no
> privileged core to patch.** — `docs/architecture.md`

`packages/extensions/` is titled *"the agent modifies its own runtime."* Five
tools over the live Cordis runtime in the current process:

| Tool | Contract |
|---|---|
| `cordis_inspect` | live services, plugin fibers, tools, API signatures, events, browser slot surface |
| `cordis_define` | records an immutable package (host JS and/or browser React), syntax-checks, **runs nothing** |
| `cordis_run` | evaluates host half in a `node:vm` sandbox, ships browser half to open pages |
| `cordis_stop` | dispose to quiescence |
| `cordis_undefine` | forget |

**Key divergence from omarchy: dsh's self-modification is deliberately
ephemeral.**

> They create no Plugin file, install no package, change no `cordis.yml` or
> personal/project configuration, do not survive restart, and **cannot be
> promoted automatically.**

**They already walked back the sharpest edge.** The design note (2026-07-08)
describes three tools where `cordis_mount` evaluated code immediately. Shipped,
that's split into `define` (shows the user a card) and `run` (may require
approval), plus immutable versioned packages — changing code means defining a new
`packageId`, with rollback to `currentPackageId`. Write-and-execute got pulled
apart with a human in the seam.

**They built the oracle first.** The stated problem from the design note:

> model-written code has to call service APIs whose source it has never seen —
> guessed method signatures and, worse, guessed return-value shapes cost many
> steps of blind probing

So: `api-catalog.ts` is *generated* from an AST walk of every Cordis declaration,
and CI-gated — `pnpm run verify-cordis-api` regenerates in memory and fails on
any diff, so a JSDoc edit can't ship without regenerating what the model reads.
`inspect.ts` then intersects that compile-time catalog with the *live* service
store: what is RUNNING comes from the store, what each service CAN DO comes from
the catalog. `curation.ts` hides non-injectable keys because "naming a key a
package cannot reach advertises a call that cannot be made."

**Disposability is structural.** Cordis registrations are effects that unwind on
unload; `cordis_unmount` returns only after every owned tool, listener, service,
timer, and effect reaches quiescence.

**Trust stance, stated honestly in both:** "The sandbox isolates globals but is
not a security boundary... Treat this toolset like bash access."

### The convergence

The `cordis` preset (创造模式, "creation mode") exists "so a person can ask an
agent to author another agent." Its persona carries:

> **NEVER edit or delete the shipped preset install**... it belongs to the
> deployment, an upgrade overwrites it, and corrupting the `cordis` preset would
> disable this very mode. To change what a shipped preset does, copy its
> composition into a new preset directory and edit the copy.

That is omarchy's `/usr/share/omarchy` vs `~/.config` rule, reinvented
independently, plus a self-preservation clause. Four identical invariants:

1. **Two layers** — immutable vendor layer (read freely, never write) + mutable
   user layer
2. **Fork-on-write** — `omarchy plugin clone` / "copy its composition into a new
   preset directory"
3. **One uniform contract** — first-party and third-party plugins are the same
   format, differing by a flag, so an agent that can write a plugin can rewrite
   *any* part of the system
4. **Reversibility** — timestamped backups + `omarchy refresh`; effect unwinding +
   immutable versions with rollback

### Isolation is the wrong dial

The obvious framing is: lisp machine (no isolation, maximally extensible,
fragile) ←→ OS (strong isolation, safe, rigid), pick the middle. **Neither
codebase did that.**

dsh explicitly disclaims its vm as a security boundary. Omarchy launches every
agent with `--dangerously-skip-permissions` by default. Both chose **near-zero
isolation** on the capability axis — they kept the lisp machine.

What they added was on three *other* axes:

| Axis | Mechanism |
|---|---|
| **Layering** | vendor immutable / user mutable |
| **Reversibility** | effect unwinding, backups, refresh, versioned rollback |
| **Legibility** | generated + CI-gated catalogs so the agent reads ground truth |

Isolation buys safety at the cost of the thing that made the lisp machine worth
extending. These projects declined that trade and paid for safety elsewhere.

---

## 7. Two different shapes, not one

The final and most important refinement. "Agent navigates a nondeterministic
space to find a thing" is true of both domains at an altitude where it stops
being useful. Underneath:

| | **Search** (chip, compiler, kernels) | **Construction** (application layer) |
|---|---|---|
| Target | unknown — you know the cost, not the answer | specified but underdetermined |
| Baseline | an incumbent to beat | nothing; there's no "before" |
| Success | measurably better than incumbent | matches intent *and* is legal |
| **Objective lives** | **in the machine** | **in someone's head** |

That last row decides everything.

In chip design you can run a million iterations overnight because the cost
function is *in the machine* — the loop needs no human to close. At the
application layer the verifier can tell you an arrangement is **legal**. It can
never tell you it's **what was wanted**.

The lego framing is right but stops one step short: the environment checks that
the pieces snap together. It cannot check that you built the thing the person
asked for. **Legality ≠ intent.**

Which is exactly why dsh put an approval card between `define` and `run`. Not
caution — structural. The only oracle for intent is the person, so the loop must
terminate at them. In chip design it doesn't have to. **That is why chip design
will automate much further, much faster, than application construction will.**

Also: the app layer has no gradient, so it isn't hill-climbing with a weaker
oracle. It's a different operation — **constrained construction with fast
rejection.** The agent isn't searching for an optimum; it's trying to land inside
the set of valid programs in one or two tries. That's why dsh invested in a
generated API catalog rather than a benchmark harness.

---

## 8. On creativity

The agent's edge isn't creativity-as-taste. It's three things:

- **No aesthetic pruning** — willing to evaluate regions that look ugly
- **Throughput** — no fatigue over a million candidates
- **No career risk** — a human expert won't spend three weeks on a path that
  looks stupid; the agent doesn't care how it looks

And then the sentence that matters:

> **Unverified creativity is hallucination.** They are the same phenomenon,
> distinguished only by whether a checker exists.

Move 37 was move 37 and not a blunder because AlphaGo had a value network from
self-play behind it. Strip the oracle and the same move-generation process
produces confident nonsense at the identical rate.

So "don't hamper the creativity" and "gate it with a cheap verifier" aren't in
tension — **the gate is what converts the creativity into anything.** Widen the
generator in proportion to how cheap your checker is. The two dials are coupled,
not opposed.

### Why human aesthetics became a liability

Elegance, symmetry, "this feels right" — these are a **pruning heuristic**.
Pruning trades coverage for speed. That trade was overwhelmingly correct when
evaluation was expensive and performed by a person. When evaluation becomes
nearly free, the same heuristic is pure loss: it discards exactly the ugly
regions where the remaining wins live, because the beautiful ones were picked
over decades ago.

*This is the actual bitter lesson.* Note the common misreading: the dav1d
engineers' expert intuition is the **hand-crafted knowledge** side of Sutton's
dichotomy, not the winning side. The AI-shaped analogue of dav1d isn't "the model
has better intuition than the VLC developers" — it's **verified search over
instruction sequences** (STOKE, Souper, AlphaDev's shorter sort-3 that shipped in
libc++). Different mechanism, different prediction.

---

## 9. No, the agent does not go straight to ones and zeros

The tempting final inference — if agents are this good, delete MLIR and LLVM IR
and emit machine code — **inverts the whole thesis.**

MLIR and LLVM IR aren't a readable crutch for humans. **They are where the
verifier lives.** Every MLIR op carries a `verify()`. SSA invariants, type
checking, dominance, alias analysis, dialect legalization, ABI conformance,
relocation correctness, DWARF, exception tables — that isn't bloat between the
agent and the metal. That is the acceptance layer.

An agent emitting raw machine code doesn't skip a slow step. **It discards the
checker.**

The parts you'd be "freeing" the agent to do — register allocation, scheduling,
encoding — are also the parts where compilers already beat humans and where there
is no creative headroom. The headroom is higher up: instruction selection,
algorithm choice, data layout, and the rewrite rules themselves.

Where agents actually land in the compiler stack:

- proposing **rewrite rules**, verified against the IR's existing semantics
  (Souper already does this with an SMT solver)
- **tuning cost models** that were hand-fit heuristics
- **superoptimizing** hot sequences, checked for equivalence
- writing the **kernel** the compiler was never going to autovectorize — dav1d's
  GFNI trick, at scale

Every one keeps the IR and uses it as the oracle. **The IR gets more important in
an agentic world, not less** — it's the deterministic acceptance surface that
makes aggressive generation safe.

---

## 10. The primitive

Most attempts at agent reliability try to make **generation** deterministic —
constrained decoding, restricted DSLs, templates, fill-in-the-blank scaffolds.
Both codebases studied here do the opposite. They let the agent write arbitrary
JavaScript or arbitrary bash, and make **acceptance** deterministic instead:
registration validates at registration time rather than at first use, catalogs
regenerate in CI and fail on diff, effects unwind mechanically, malformed schemas
reject before reaching a prompt.

> **The deterministic part isn't the layer the agent writes in. It's the layer
> that judges what the agent wrote.**

Nondeterminism fully contained on the generation side. Determinism entirely on
the acceptance side. That's the marriage.

---

## 11. Where to look for the next instance

Not a layer in the stack — a predicate. All three must hold:

1. **A deterministic method exists but is incomplete** — compilers do 95%, the
   last 5% needs judgment
2. **The space of legal moves is enumerable** — you can hand the agent a *true*
   description of what exists
3. **Failure is cheap and reversible**

Where all three hold: plugin systems, build configs, CI pipelines,
infrastructure-as-code, query plans, schema migrations, editor extensions, kernel
autotuning.

Where they don't: a payment flow's business logic is application-layer and has
none of them — no oracle, no enumerable move set, irreversible failures.
Meanwhile a linker script sits far down the stack and has all three.

**Look for the predicate, not the layer.**

---

## 12. Open questions

**The persistence disagreement.** The two codebases genuinely conflict, and
neither has evidence:

- **Omarchy:** agent edits persist to disk. That *is* the product. Bet: your
  machine becomes progressively more yours.
- **dsh:** dynamic packages evaporate on restart and cannot be promoted
  automatically. Bet: unpromoted accumulation is how you get a system nobody can
  reason about.

If you're building in this space, that's the decision the existing
implementations can't answer for you.

**Others:**

- Does the economically-justified frontier for hand-written kernels actually move
  when expert-hours get cheap? (The strongest version of the original thesis, and
  the one worth betting on.)
- Can property/assertion sets be generated well enough to become the missing RTL
  oracle — or is that just as hard as the design?
- Does correct-by-construction survive contact with numerics, or does every real
  system end up with a reference-comparison test suite anyway?

---

## Appendix: doc drift found while reading

- **deepseek-harness** `AGENTS.md` lists `packages/self-modification/` in the repo
  layout. No such package exists; the code is `packages/extensions/`.
- The `2026-07-08` design note documents three tools (`inspect`/`mount`/
  `unmount`); shipped code has five-plus (`inspect_list`/`inspect_query`/
  `inspect_self`/`define`/`run`/`stop`/`undefine`). The note is the older design.

Fast-moving developer preview. The code is the truth.

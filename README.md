# Forensic Audit — the "machine-verified proof of P = NP" (Pedigree Polytopes, Lean 4)

A step-by-step, independently reproducible audit of **[TiruArt/Pedigree-Polytopes-Lean4](https://github.com/TiruArt/Pedigree-Polytopes-Lean4)** — the Lean 4 code behind **[arXiv:2606.03194](https://arxiv.org/abs/2606.03194)**, which claims a machine-verified proof that P = NP.

**📄 Full report (9 pp. — method, discovery trails, from-scratch reproduction, limitations):**
[`AuditReport_Possibly_New_Findings.pdf`](./AuditReport_Possibly_New_Findings.pdf)

Audited commit: `3c9c90e` (2026-06-04) · Audit date: 2026-10-05 · Toolchain: elan 4.2.4, Lean 4.30.0, Mathlib cache (8459 files)

---

## TL;DR

The main theorem **does build cleanly** — we reproduced the paper's headline number exactly
(`lake build MembershipProject.Core.N_PEqualsNP` → **2968/2968 targets, zero errors**).
But asking the Lean kernel what the result actually depends on (`#print axioms`) shows:

| Category | Count | Meaning |
|---|---|---|
| Standard logic (harmless) | 2 | `propext`, `Quot.sound` |
| **Undefined propositions** (declared, never defined) | **9** | incl. `P_equals_NP` — *the thing being "proven" itself* |
| **The project's novel math, assumed not proved** | **5** | incl. `mi_objective_solves_stsp_ax` (the STSP reduction) |
| Legit published external results (citations) | 5 | Cook 1971, Karp 1972, Tardos 1986, GLS 1988, Maurras 2002 |

On top of that, this audit reports **three findings we could not find documented anywhere** (see honesty note below):

1. **[Finding A](#finding-a--the-polytope-a_n-is-literally-the-one-point-type-unit)** — the polytope `Aₙ` that the whole optimization chain runs on is literally `Unit`, a one-point placeholder type, per the author's own comment.
2. **[Finding B](#finding-b--build--ci-forensics)** — a fresh clone cannot build the full library (a source file was never committed), and the *default* build compiles **0 targets** — so the CI badge can go green without compiling anything.
3. **[Finding C](#finding-c--the-theorem-statement-printed-in-the-paper-does-not-exist-in-the-repo)** — the paper prints `theorem p_equals_np : P_class = NP_class`, but `P_class`/`NP_class` exist nowhere in the repository; the actual theorem concludes an opaque axiom.

> **Honesty note (novelty labels).** *NEW-CANDIDATE* means: searched the public record (the repo's six GitHub issues + web search) on 2026-10-05 and **not found documented** — it is *not* proof of absence. Other critiques may exist where we did not look. What was **already known** is credited in [What was already known](#what-was-already-known). None of this work says anything about whether P equals NP or not: **P vs NP remains an unsolved problem.**

---

## Finding A — the polytope `Aₙ` is literally the one-point type `Unit`

**Status: NEW-CANDIDATE (medium confidence)**

`MembershipProject/Core/N_PEqualsNP.lean`, lines 145–146:

```lean
/-- The projected polytope Aₙ (placeholder type). -/
def An (n : ℕ) : Type := Unit  -- placeholder: ℝ^{αₙ} vectors
```

`Unit` has exactly one element. Every type-level "polytope" in the chain — the argument to
`mi_objective_solves_stsp_ax : PolynomialOptimisation (An n) → STSP_in_P` and to the
`convAn_*` axioms — is this one-point type, for every `n`. The predicates applied to it
(`PolynomialOptimisation`, `FullDimensional`, …) are themselves undefined opaque axioms, so
nothing is ever computed on it: `Aₙ` flows through the proof as a name only.

**Verify (2 seconds):**

```bash
grep -n "placeholder" MembershipProject/Core/N_PEqualsNP.lean
# -> line 145: /-- The projected polytope Aₙ (placeholder type). -/
# -> line 146: def An (n : ℕ) : Type := Unit  -- placeholder: ℝ^{αₙ} vectors
```

---

## Finding B — build & CI forensics

**Status: NEW-CANDIDATE (medium-high confidence — the most checkable finding here)**

Three interlocking facts about building from a fresh clone:

1. **A live module was never committed.** Two files import `MembershipProject.Core.F5Network`,
   but that file exists nowhere in the repository — only `Backup/F5Network.lean` and
   `Backup/F5Network1.lean` do. `git ls-files` confirms it was never tracked.
2. **The full-library build fails** on that missing file.
3. **The default build compiles nothing:** plain `lake build` prints
   `Build completed successfully (0 jobs)` and exits 0. The repo's CI
   (`.github/workflows/lean_action_ci.yml` → `leanprover/lean-action@v1`, defaults) runs this
   same default command — a green badge is not evidence the library compiles.
   *(We read the workflow and reproduced its command locally; we did not execute GitHub Actions itself.)*

**Counter-evidence, being fair to the authors:** the paper's specific claim — *2968/2968 build
targets clean* — **does reproduce** when you build the main theorem's target explicitly.

**Verify:**

```bash
# 1. The ghost file: only Backup copies are tracked
git ls-files | grep F5Network
# -> Backup/F5Network.lean
# -> Backup/F5Network1.lean        (Core/ version absent)

# 2. Who imports the missing module
grep -rn "import MembershipProject.Core.F5Network" --include="*.lean" MembershipProject/
# -> MembershipProject/Tests/TestPythonIntegration.lean:1
# -> MembershipProject/Algorithms/PythonMaxflow.lean:3

# 3. Default build: success with zero work
lake build
# -> 'Build completed successfully (0 jobs).'   (exit code 0)

# 4. Full library build: fails on the missing file
lake build MembershipProject
# -> error: no such file or directory ... MembershipProject/Core/F5Network.lean

# 5. Main theorem target: reproduces the paper's number
lake build MembershipProject.Core.N_PEqualsNP
# -> [2968/2968] Build completed successfully (2968 jobs).
```

---

## Finding C — the theorem statement printed in the paper does not exist in the repo

**Status: NEW-CANDIDATE (medium confidence)**

The arXiv paper displays its headline result twice (abstract + body) as:

```lean
theorem p_equals_np
: P_class = NP_class
```

The repository contains **no identifier `P_class` or `NP_class` anywhere** (zero matches across
all 194 `.lean` files). The actual theorem is a different statement with a different conclusion:

```lean
-- MembershipProject/Core/N_PEqualsNP.lean
line 107:  axiom P_equals_NP : Prop           -- an opaque, undefined constant
line 364:  theorem p_equals_np (n : ℕ) (hn : 5 ≤ n) : P_equals_NP := by
```

Tellingly, the author's own reply on
[issue #1](https://github.com/TiruArt/Pedigree-Polytopes-Lean4/issues/1) explains that a real
proof "would require defining `P_class` and `NP_class` over a formal model of computation" —
the names live in the paper, were never defined in code, and the paper nonetheless prints them
inside the theorem statement.

**Verify:**

```bash
grep -rn "P_class" --include="*.lean" .
# -> (no output)

grep -n "axiom P_equals_NP\|theorem p_equals_np" MembershipProject/Core/N_PEqualsNP.lean
# -> 107: axiom P_equals_NP : Prop
# -> 364: theorem p_equals_np (n : ℕ) (hn : 5 ≤ n) : P_equals_NP := by
```

Paper side: <https://arxiv.org/abs/2606.03194> → search the text for `P_class`.

---

## Finding D — the compiler-generated receipts (`#print axioms`)

**Status: KNOWN in substance ([issue #1](https://github.com/TiruArt/Pedigree-Polytopes-Lean4/issues/1)
reached the same conclusion manually in June 2026); this machine-generated list appears
unpublished — LOW novelty.**

This is what Lean itself prints for the headline theorem at the audited commit
([`Audit.lean`](./Audit.lean) in this repo, run with `lake env lean Audit.lean`):

```
'MembershipProject.Core.p_equals_np' depends on axioms: [propext,
 Quot.sound,
 MembershipProject.Core.FullDimensional,
 MembershipProject.Core.HasInteriorPoint,
 MembershipProject.Core.P_equals_NP,
 MembershipProject.Core.PolynomialOptimisation,
 MembershipProject.Core.PolynomialSeparationOracle,
 MembershipProject.Core.QuickProtocol,
 MembershipProject.Core.RationalityGuaranteed,
 MembershipProject.Core.SAT_in_P,
 MembershipProject.Core.STSP_in_P,
 MembershipProject.Core.convAn_full_dimensional_ax,
 MembershipProject.Core.convAn_has_interior_point_ax,
 MembershipProject.Core.convAn_rationality_guaranteed_ax,
 MembershipProject.Core.cook_np_completeness,
 MembershipProject.Core.gls_optimisation,
 MembershipProject.Core.karp_stsp_np_complete,
 MembershipProject.Core.maurras_separation,
 MembershipProject.Core.membership_An_of_Pn,
 MembershipProject.Core.mi_objective_solves_stsp_ax,
 MembershipProject.Core.tardos_strongly_polynomial]
'MembershipProject.Core.necessity_of_mcf' depends on axioms: [propext,
 Classical.choice,
 Quot.sound,
 MembershipProject.Core.necessity_of_mcf]
'MembershipProject.Core.mi_objective_solves_stsp' depends on axioms: [MembershipProject.Core.PolynomialOptimisation,
 MembershipProject.Core.STSP_in_P,
 MembershipProject.Core.mi_objective_solves_stsp_ax]
MembershipProject.Core.An : ℕ → Type
```

Two lines deserve attention:

- **`MembershipProject.Core.P_equals_NP` appears in its own proof's axiom list** — because the
  proposition being proved was declared with `axiom P_equals_NP : Prop`, the kernel counts it
  as an assumption of the result.
- **`necessity_of_mcf` lists *itself*** — Lean's way of saying "I am an axiom; I was assumed."
  (Consistent with the paper admitting the necessity direction is future formalisation. Note
  the necessity module is not even imported by the final theorem's chain.)

---

## 60-second verification (all findings at a glance)

```bash
git clone https://github.com/TiruArt/Pedigree-Polytopes-Lean4.git
cd Pedigree-Polytopes-Lean4
git log -1 --format="%h %ad %s" --date=short        # -> 3c9c90e 2026-06-04

grep -n "placeholder" MembershipProject/Core/N_PEqualsNP.lean   # Finding A
git ls-files | grep F5Network                                   # Finding B (Backup/ only)
grep -rn "P_class" --include="*.lean" .                         # Finding C (nothing)

# Finding D (needs toolchain, ~30-60 min for Mathlib cache + builds):
lake exe cache get
lake build MembershipProject.Core.N_PEqualsNP                   # -> 2968/2968
lake build MembershipProject.Core.N_MembershipCharacterisation  # -> 2960/2960
lake env lean Audit.lean                                         # -> receipts above
```

Full walkthrough, expected outputs, timings and troubleshooting: see the
[PDF report, §7 Appendix](./AuditReport_Possibly_New_Findings.pdf).

---

## What was already known

We initially mislabelled parts of this audit as new discoveries and **retract them here**:

| Claim | Found first by | When |
|---|---|---|
| "The theorem does not prove P = NP" — the earlier revision had `def P_equals_NP : Prop := True`, so the theorem proved literally `True` | [issue #1](https://github.com/TiruArt/Pedigree-Polytopes-Lean4/issues/1) by **log2cn**; a commenter then noted "So it proves an axiom is true now", and the author conceded the theorem "establishes that this axiom is reachable … not that P = NP in a formally defined complexity-theoretic sense" | 2026-06-03 |
| "P/NP are never defined over a model of computation" | The repository author's own reply on issue #1 | June 2026 |
| "The chain uses axioms, not proofs" | Public in the repository README (pipeline steps labelled "Axiom") | always |
| "The necessity direction is unproved" | The paper itself (future formalisation) | June 2026 |

---

## Repository contents

| File | What it is |
|---|---|
| [`AuditReport_Possibly_New_Findings.pdf`](./AuditReport_Possibly_New_Findings.pdf) | The full 9-page report: methodology, per-finding discovery trails with exact commands + expected outputs, prior-art credits, reproduction-from-scratch appendix, limitations |
---

## Limitations

- **Novelty is a search result, not a proof.** *NEW-CANDIDATE* = not found in the six GitHub
  issues (titles/bodies; some comment threads are JS-rendered and could not be fully read) and
  general web search, on 2026-10-05. Reddit, Hacker News, X and citing papers were not checked.
- **CI behaviour is inferred, not executed:** we read the workflow file and reproduced its
  default build command locally; we did not run GitHub Actions itself.
- **The "0 jobs" mechanism is observed, not root-caused.**
- The 2968 targets that do compile are real Lean code checked by the kernel; this audit does
  not challenge their internal correctness — only what the overall theorem claims to establish.
- **P versus NP is still open.** This audit is evidence about one claimed proof, not evidence
  about P vs NP itself.

## License

[MIT](./LICENSE) — reuse, quote, translate; a link back is appreciated.

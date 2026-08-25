# Proposal: Configurable quality level per project

> **Status**: proposal pending implementation — working document.
> **Origin**: evaluation of the `saas-prototype` run (see the project's `REPORT.md`): fixed
> gates (CRAP≤6, mutation run, DRY, soft Gherkin) cost tokens that not every project needs.
> This proposal adds a configurable **quality axis** per project.
> **Note**: the two-pack review (section 5) narrows the proposal — the real value is depth
> within a pack, not replacing pack choice.

## 1. The 3 levels

| Level | Relative cost | Typical use |
|---|---|---|
| `minimal` | ~1x | Prototypes, spikes, throwaway code |
| `standard` | ~2x | Reasonable default for product features |
| `maximum` | ~3-4x | **Current rigor** — critical libraries, security/payments, code consumed by others |

`maximum` = what the pipeline already does today (nothing beyond that for now).

## 2. Gates by level

| Gate | minimal | standard | maximum (current) |
|---|---|---|---|
| TDD + unit tests | ✓ | ✓ | ✓ |
| Acceptance Gherkin | optional | ✓ (without full APS pipeline) | ✓ full APS pipeline |
| CRAP | — | improve what is reasonable, no hard gate | **≤6** |
| DRY | — | reduce reasonable duplication | tooling, strict |
| Mutation scan + split >100 sites | — | ✓ (count only — cheap) | ✓ |
| Full mutation run | — | — | ✓ differential, kill non-equivalents |
| Soft Gherkin mutation | — | — | ✓ |
| Property tests | — | — | support |
| Written report | — | optional | ✓ |

## 3. Role responsibility by level (four-pack)

| Role | minimal | standard | maximum |
|---|---|---|---|
| **specifier** | Scoping + human gate only (no mandatory Gherkin) | Gherkin ✓ | Gherkin + QA suite |
| **coder** | Implements with TDD — required | ✓ | ✓ |
| **refactorer** | No gates → **no real work** | Reasonable CRAP + DRY + mutation scan | full gates |
| **architect** | No gates → **no real work** | light structural review only | full gates |

**Conclusion**: level and workflow are correlated. At `minimal`, refactorer/architect have no
gates to apply — configuring 4 roles would waste tokens with no benefit.

## 4. Mechanism (prompts/articles only, zero code)

```
1. PROJECT (project.prompt):         "## Quality Level → Quality level: standard"
2. SHARED (quality.prompt):          the ON/OFF gate table by level
3. EACH ROLE PROMPT (one line):      "Apply only the gates that are ON for
                                     the project's level (see quality article)"
```

The agent reads the level in `project.prompt`, the table in `quality.prompt`, and its role
prompt tells it to apply only the ON gates → consistent interpretation across roles.

The `quality.prompt` article also includes the **role mapping by level**: "at minimal,
configure only specifier+coder (2 windows in `swarmforge.conf`); at standard, add
refactorer; at maximum, all 4".

**Honest nuance**: the adjustment is *prompt-soft* — agents follow the level by instruction.
Hard enforcement would need a `tools/quality-check` (validate level artifacts), an optional
later step.

## 5. Review: does two-pack already solve part of this?

**Result of reviewing two-pack's real scope (original project):**

- **coder (two-pack)**: TDD + unit tests ONLY — explicitly excludes acceptance, Gherkin, IR,
  Gherkin mutation, property tests, CRAP, DRY, and language mutation.
- **cleaner (two-pack, batch)**: coverage, **CRAP≤6**, **DRY**, structure/encapsulation/dependencies
  and **mutation run on uncovered behavior** + tests to kill mutants.

**two-pack quality profile**: unit tests ✓ · CRAP≤6 ✓ · DRY ✓ · mutation run ✓ ·
structure ✓ (inside cleaner) · acceptance/Gherkin ✗ · property ✗ · separate QA ✗.

### Conclusion

1. **two-pack is NOT `minimal`**: it keeps the hard hardening gates (CRAP≤6, DRY, mutation
   run). It is "full hardening without specification" — not the cheap option on the depth
   axis.
2. **Packs already encode a quality axis**: which gates EXIST (two-pack: no spec;
   four-pack: spec + architecture; six-pack: + hardender + QA).
3. **The level axis adds what packs do NOT cover**: the DEPTH of each active gate
   (CRAP≤6 vs "improve reasonably"; mutation run vs scan-only; soft Gherkin on/off).
4. **Practical implication**: for "cheap", choosing two-pack already drops the expensive
   layers (spec/architecture) — the main cost lever is the pack. Level is for scaling depth
   WITHIN a pack (e.g. two-pack without mutation run, four-pack without Gherkin mutation).
   The run confirmed it: the main waste was four-pack for a login form, not gate depth.

**Verdict**: the level proposal remains valid but is **narrower** than it first seemed: its
real value is depth within a pack, not replacing pack choice. Possible simplification: start
with only two levels (standard = current, light = no mutation run or Gherkin mutation) and
let pack choice do the rest.

## 6. Analysis: spec vs hardening priority (the critique of two-pack)

**two-pack's logic**: TDD already specifies behavior at the unit-test level; Gherkin is a
second layer (reviewable contract + end-to-end acceptance) that is expensive (APS pipeline);
hardening gates (CRAP≤6, DRY, mutation run) are the code quality floor.

**The critique (valid)**: for a small task the priority is inverted — mutation run is
expensive and protects code that may be thrown away in a prototype; cheap spec ensures the
RIGHT thing is built. A well-hardened but wrong feature is still wrong. Logical order:
first WHAT (spec), then HOW (gates).

| | two-pack | spec-first variant (proposal) |
|---|---|---|
| Base spec | unit tests (TDD) | Light Gherkin (reviewable contract + human approval) + TDD |
| Code protection | CRAP≤6 + DRY + **mutation run** | Reasonable CRAP/DRY, **no mutation run** |
| Cost | ~2-3x | ~1.5-2x |
| Risk covered | dirty/unchangeable code | **building the wrong thing** |

**The gap it reveals**: two-pack assumes Gherkin comes with the full APS pipeline cost
(parser + entrypoint generator + runtime + step handlers). It offers no "spec-lite" variant:
write Gherkin as a reviewable contract without building the pipeline or running mutation.

**Refinement of the `light` level**:

> `light` = Gherkin written as contract + human approval + TDD + reasonable CRAP/DRY —
> **no APS pipeline, no mutation run, no Gherkin mutation, no property tests**.

This variant covers the most important risk (is it the right thing? does a human approve?)
at lower cost than "mutation run without spec".

## 7. Spec-light: Gherkin or other alternatives?

**Comparison of options for a cheap, reviewable contract:**

| Option | Cost | Human pre-code contract | Real enforcement | Risk | Upgrade to spec-full |
|---|---|---|---|---|---|
| TDD tests as spec (two-pack) | ~1x | ❌ | ✅ | user sees behavior at the end | — |
| Prose criteria (markdown) | ~1x | ✅ imprecise | ❌ | ambiguity | rewrite |
| **Gherkin written-only (light)** | ~1.5x | ✅ precise | ❌ | **spec drift** | ✅ zero rewrite |
| Given/When/Then scenarios in markdown | ~1x | ✅ | ❌ | no standard format | medium rewrite |

**Key point**: in light, real enforcement comes from the coder's TDD — it turns each approved
scenario into unit tests. Gherkin remains a human contract + guide, not verification.
**Spec drift** risk is mitigated by a light workflow rule: *"the coder maps each approved
scenario to unit tests; the handoff/report confirms the scenario→tests mapping"*.

**Recommendation**: in four-pack, light = natural degradation — the specifier already writes
Gherkin and asks for approval; the back half of the pipeline is cut (the coder does not build
entrypoint generator/runtime/step handlers, implements with TDD mapping scenarios). If you
later scale to spec-full, the `.feature` files are already there — you only build the pipeline
around them.

**Recommended cheap add-on**: in light, the coder runs `gherkin-parser` ONLY to validate that
the spec parses (seconds, no pipeline build) — prevents broken Gherkin syntax from passing
as a contract.

**When to choose each**: never going to scale → prose or TDD-only; may scale → Gherkin
written-only (the format IS the upgrade path); human must approve before coding → Gherkin.

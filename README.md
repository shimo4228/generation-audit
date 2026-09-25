Language: English | [日本語](README.ja.md)

# generation-audit

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/generation-audit)

An [Agent Skill](https://agentskills.io/specification) that audits your self-authored Claude Code assets (rules / CLAUDE.md / output styles / skills / agents) when a new Claude model takes over a role in your harness. Rules written to compensate for an older model's weaknesses can turn into **conflict cost** on the next one: contradictory instructions the model must silently resolve on every request. This skill finds them in one pass, fixes the lines it can fix safely, and hands the rest to you or to the stocktake skills with the evidence.

It is the re-runnable procedure for the model-generation trigger of **Scaffold Dissolution** in the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle).

## Two checks in one run

| Check | Compared against | Engine |
|---|---|---|
| Runtime cross-check (Phases 1–2) | The system prompt and tool descriptions actually loaded at inference time | this skill |
| Dated-pattern scan (Phase 3) | The outdated writing patterns listed in the official guide for the target model | `/claude-api prompt-audit`, the Claude API skill that ships with Claude Code |

The first check is the one no generic scanner does. Your config repo alone cannot show it, because parts of the runtime layer are injected from outside the repo. The second check is delegated on purpose. The pattern table ships inside Claude Code's bundled `claude-api` skill, and Anthropic revises it as new models are released (checked against CLI 2.1.280). The skill reads it at run time instead of keeping a copy that would go stale.

## The procedure

0. **Phase 0 — Target and scope.** Name the model that took the role (judge tier, build tier, or everyday sessions) and the assets that role reads. Unmodified copies of external skills and files another session is editing are audited but not changed.
1. **Phase 1 — Capture the runtime layer** in a session running the target model in its real environment, one theme at a time, as verbatim quotes. Zero findings means "not found by these questions", not "no conflict".
2. **Phase 2 — Classify** each mismatch as **conflict** (the asset and the runtime layer disagree while both are loaded) or **redundancy** (the runtime layer already says it). Ghost references to settings or tools that no longer exist count as conflict.
3. **Phase 3 — Scan for dated patterns** with `/claude-api prompt-audit`, split into slices and run by read-only subagents in parallel.
4. **Phase 4 — Judge and apply.** Each finding is judged by four questions: intent (was the override deliberate?), evidence (is the reason recorded?), freshness (does the premise still hold for this model?), and expiry (did a declared review trigger fire?). Dated-pattern findings graded High or Medium confidence are applied as line fixes: directly if you asked for fixes, otherwise group by group after you approve the diff. Always-loaded files (rules, CLAUDE.md, output styles) are shown one diff at a time either way.
5. **Phase 5 — Hand off and record.** Runtime cross-check findings and whole-asset candidates (retire, merge) come here. CLAUDE.md and output styles are edited by this skill, one item at a time with your approval. Rules, skills, and agents go as evidence to the stocktake skill for that asset class, which owns the verdict. The applied lines are recorded in the commit body.

## Known pitfalls the procedure encodes

- **"Delete verification steps" misdiagnosis.** Official guidance targets *model self-verification*. Deterministic checks run by machines (build, tests, hooks) are out of scope, and the skill tells the two apart.
- **Inversion, not deletion.** An instruction whose direction changed (a confidence-threshold suppression clause, for example) has to be rewritten the opposite way. Deleting it leaves the suppressive frame in place.

## Requirements and adapting it

- **Claude Code** with its bundled `claude-api` skill. Phase 3 reads `shared/prompt-audit.md` and `shared/model-migration.md` from that skill's base directory, which changes with the CLI version.
- **Written for the author's harness.** The body names paths in `~/.claude` (`rules/common/akc-cycle.md`, `scripts/hooks/harness_lint.py`, the `skill-health` reference scanner, the `skill-creator` writing rules) and cites decision records by number (ADR-00NN, published in Japanese in [claude-harness](https://github.com/shimo4228/claude-harness/tree/main/docs/adr)). In another harness, replace those with your own lint, reference checker, and writing conventions. The Phase 0–5 flow itself does not depend on them.

## Install

### Claude Code

```bash
cp -r skills/generation-audit ~/.claude/skills/generation-audit
```

### SkillsMP

```bash
/skills add shimo4228/generation-audit
```

## Usage

```
/generation-audit        # run when a new Claude model takes a role in your harness
```

The model never starts it on its own (`disable-model-invocation: true`). A generation change is a rare, explicit event, so the skill does not rely on probabilistic self-triggering.

## Companion skills

Phase 5 hands findings about rules, skills, and agents to three stocktake skills. Without them the dated-pattern fixes are still applied and the runtime findings are still listed in chat, but no verdict is reached for those assets. The full loop expects:

- [`rules-stocktake`](https://github.com/shimo4228/rules-stocktake) for rules loaded into every session
- [`skill-stocktake`](https://github.com/shimo4228/skill-stocktake) for installed skills
- [`agent-stocktake`](https://github.com/shimo4228/agent-stocktake) for agent definitions

## Provenance

The runtime cross-check comes from the Claude 4 → Claude 5 audit (2026-07), which cut always-loaded rules from 5,789 to 2,314 words. That audit found a plan-mode ban that contradicted the new `EnterPlanMode` tool description, a ghost reference to a settings key that no longer existed, and a confidence-threshold suppression clause in an agent definition. The dated-pattern scan was first run for Fable 5.1 (2026-09-02; 3 High and 85 Medium findings). The two were merged into this single entry for Opus 5.5 (2026-09-25). In that run the dated-pattern scan found 107 issues and 64 were applied. None were patterns specific to Opus 5.5; most were narrative history and version-bound wording left in the assets. The runtime cross-check in that run was done informally and found one deliberate override, left as it was.

## About this skill

Implements the model-generation trigger of Scaffold Dissolution in the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) ([DOI 10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726)).

## License

MIT

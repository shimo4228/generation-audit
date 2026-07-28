Language: English | [日本語](README.ja.md)

# generation-audit

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/generation-audit) [![GitMCP](https://img.shields.io/endpoint?url=https://gitmcp.io/badge/shimo4228/generation-audit)](https://gitmcp.io/shimo4228/generation-audit)

An [Agent Skill](https://agentskills.io/specification) that runs a **model-generation-change audit** over your self-authored Claude Code assets (rules / CLAUDE.md / skills / agents). When a new model generation ships, rules written to compensate for the previous generation's weaknesses can turn into **conflict cost** — contradictory instructions the model must silently resolve on every request. This skill finds them.

It is the concrete, re-runnable procedure for the **Scaffold Dissolution** model-generation trigger of the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle).

## The core idea — collate against what is actually loaded

The canonical reference is not the official blog or docs — it is the **runtime layer**: the system prompt and tool descriptions that share the context window with your assets at inference time. The skill:

1. **Phase 1 — Capture the runtime layer** from the live session, theme by theme, in verbatim quotes (self-report caveats and cross-session reproduction checks built in). Your config repo alone cannot tell you this — parts of the runtime layer are injected from outside it.
2. **Phase 2 — Classify mismatches** as **conflict** (contradicting instructions co-loaded in the same context), **redundancy** (the substrate already says it), or **drift** (divergence from official guidance that is *not* loaded).
3. **Phase 3 — Judge with four lenses**: intent (was the override deliberate?), evidence (is the why recorded?), freshness (does the premise still hold?), expiry (did a declared review trigger fire?). "Conflict = bad" is explicitly *not* mechanically applied — some overrides are intentional.
4. **Phase 4 — Hand the evidence off.** This skill holds **no verdicts**. The evidence dossier goes to the per-asset-class stocktake skills, which own the verdict tables and the one-by-one confirmation flow.

## Known pitfalls the procedure encodes

- **"Delete verification steps" misdiagnosis** — official guidance targets *model self-verification*, not deterministic machine checks (build / tests / hooks). The skill distinguishes them.
- **Inversion, not deletion** — a direction-reversed instruction (e.g. a confidence-threshold suppression clause) must be rewritten in the opposite direction; deleting it leaves the suppressive frame in place.

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
/generation-audit        # run when a new model generation ships
```

The skill is deliberately `user-invocable` — a generation change is a rare, explicit event, so it never relies on probabilistic self-triggering.

## Companion skills

Phase 4 delegates verdicts to the three stocktake siblings. The skill degrades gracefully without them (the evidence dossier is still produced), but the full loop expects:

- [`rules-stocktake`](https://github.com/shimo4228/rules-stocktake) — always-loaded rules (residency cost model)
- [`skill-stocktake`](https://github.com/shimo4228/skill-stocktake) — installed skills (trigger-pollution cost model)
- [`agent-stocktake`](https://github.com/shimo4228/agent-stocktake) — agent definitions (hybrid cost model)

## Provenance

The procedure generalizes a real audit performed across the Claude 4 → Claude 5 generation change (2026-07), which cut always-loaded residency from 5,789 to 2,463 words by classifying and dispositioning every rule against the live runtime layer — including a plan-mode ban that directly contradicted the new `EnterPlanMode` tool description, a ghost reference to a settings key that no longer existed, and a confidence-threshold suppression clause in an agent definition.

## About this skill

Implements the model-generation trigger of Scaffold Dissolution in the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) ([DOI 10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726)).

## License

MIT

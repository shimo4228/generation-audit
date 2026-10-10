Language: English | [日本語](README.ja.md)

# generation-audit

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/generation-audit)

An [Agent Skill](https://agentskills.io/specification) that audits your self-authored Claude Code assets (rules / CLAUDE.md / output styles / skills / agents) when a new Claude model takes over a role in your harness (the judge role that hands out and accepts work, the build role that implements it, or everyday sessions). Rules written to compensate for an older model's weaknesses can turn into **conflict cost** on the next one: contradictory instructions the model must silently resolve on every request. This skill finds them in one pass, fixes the lines it can fix safely, and hands the rest, with the evidence, to you or to the skills that audit rules, skills and agents (see [Companion skills](#companion-skills)).

It applies line fixes directly only when your request asks for them ("fix it", "finish it"); otherwise it shows each group's diff for approval first, and always-loaded files are shown one diff at a time either way. The author's other work is listed under [More from the author](#more-from-the-author).

## Two checks in one run

| Check | Compared against | Engine |
|---|---|---|
| Runtime cross-check (Phases 1–2) | The runtime layer: the system prompt and tool descriptions actually loaded at inference time | this skill |
| Dated-pattern scan (Phase 3) | The outdated writing patterns listed in the official guide for the target model | `/claude-api prompt-audit`, the Claude API skill that ships with Claude Code |

The first check is the one no generic scanner does. Your config repo alone cannot show it, because parts of the runtime layer are injected from outside the repo. The second check is delegated on purpose. The pattern table ships inside Claude Code's bundled `claude-api` skill, and Anthropic revises it as new models are released (the Opus 5.5 run read the copy bundled with CLI 2.1.280). The skill reads it at run time instead of keeping a copy that would go stale.

For example, in the Claude 4 to Claude 5 audit this procedure comes from, a rule said "On direct implementation instructions, execute immediately. Do not enter plan mode." The `EnterPlanMode` tool description loaded in the same context said "Prefer using EnterPlanMode for implementation tasks unless they're simple." That pair is a conflict. The rule's premise, an older model that started without planning, no longer held, so the rule was retired.

## Install

generation-audit runs [skill-health](https://github.com/shimo4228/skill-health)'s reference scanner, which checks that the scripts, files and sibling skills a skill names still exist, after it applies fixes, so install the two together; the scanner runs with [`uv`](https://docs.astral.sh/uv/) and Python 3.11 or later.

There are two routes. Cloning (first block) installs the pair alone. The akc-cycle plugin (second block) installs both, plus the three stocktake skills that receive its findings and decide what to keep, merge or retire among rules, skills and agents (see [Companion skills](#companion-skills)).

```bash
git clone https://github.com/shimo4228/generation-audit
git clone https://github.com/shimo4228/skill-health
mkdir -p ~/.claude/skills
cp -r generation-audit/skills/generation-audit skill-health/skills/skill-health ~/.claude/skills/
```

With the clone install, the skill calls the scanner at `~/.claude/skills/skill-health`, so keep the folder at that path. Run it when a new Claude model takes a role in your harness. Its capture phase (Phase 1) has the new model quote what it is told, so run it from a session that runs that model in the environment where the role works (in the author's harness the build role runs in cloud sessions, so a build-role change is audited from a cloud session):

```
/generation-audit
```

The model never starts it on its own (`disable-model-invocation: true`). A generation change is a rare, explicit event, so the skill does not rely on probabilistic self-triggering.

The same skill also ships in the [akc-cycle](https://github.com/shimo4228/akc-cycle) Claude Code plugin, together with the other skills of the Agent Knowledge Cycle (AKC: the author's six-phase, human-gated cycle that turns a coding agent's repeated experience into skills and rules). generation-audit belongs to the cycle's Curate phase, which audits what has piled up, and in the plugin it is called `/akc-cycle:generation-audit`. This repository is synced one way from the same source, so between syncs it can trail the plugin.

```
/plugin marketplace add shimo4228/akc-cycle
/plugin install akc-cycle@akc-cycle
```

## Requirements and adapting it

- **Claude Code** with its bundled `claude-api` skill. The check for outdated writing patterns reads `shared/prompt-audit.md` and `shared/model-migration.md` from that skill's base directory. That path changes with the CLI version, so the skill takes it from what `/claude-api` shows at run time.
- **Written for the author's harness.** The body names paths in `~/.claude` (`rules/common/akc-cycle.md`, `scripts/hooks/harness_lint.py`, the `skill-creator` writing rules) and cites decision records by number (ADR-00NN, published in Japanese in [claude-harness](https://github.com/shimo4228/claude-harness/tree/main/docs/adr)). `harness_lint.py`, which blocks version-drift markers in the writing, is not published; in another harness, replace it and the writing rules with your own lint and conventions, and keep or replace the `skill-health` reference scanner installed above. The audit procedure itself does not depend on them.

## The procedure

0. **Phase 0 — Target and scope.** Name the model that took the role (judge role, build role, or everyday sessions) and the assets that role reads. Unmodified copies of external skills and files another session is editing are audited but not changed.
1. **Phase 1 — Capture the runtime layer** in a session running the target model in its real environment, one theme at a time, as verbatim quotes. Zero findings means "not found by these questions", not "no conflict".
2. **Phase 2 — Classify** each mismatch as **conflict** (the asset and the runtime layer disagree while both are loaded) or **redundancy** (the runtime layer already says it). Ghost references to settings or tools that no longer exist count as conflict.
3. **Phase 3 — Scan for dated patterns** with `/claude-api prompt-audit`; when the scope is large, split into slices and run by read-only subagents in parallel. Each finding comes as `file:line`, the quoted line, the pattern, why it is outdated for the target model, a confidence grade, and the action.
4. **Phase 4 — Judge and apply.** Each finding is judged by four questions: intent (was the override deliberate?), evidence (is the reason recorded?), freshness (does the premise still hold for this model?), and expiry (did a declared review trigger fire?). Dated-pattern findings graded High or Medium confidence are applied as line fixes: directly if your request asked for fixes, otherwise group by group after you approve the diff. Low-confidence and `flag` findings are only listed in the ledger the skill outputs in chat. Always-loaded files (rules, CLAUDE.md, output styles) are shown one diff at a time either way.
5. **Phase 5 — Hand off and record.** Runtime cross-check findings and whole-asset candidates (retire, merge) come here. CLAUDE.md and output styles are edited by this skill, one item at a time with your approval. Rules, skills, and agents go as evidence to the stocktake skill for that asset class, which owns the verdict. The applied lines are recorded in the commit body.

## Known pitfalls the procedure encodes

- **"Delete verification steps" misdiagnosis.** Official guidance targets *model self-verification*. Deterministic checks run by machines (build, tests, hooks) are out of scope, and the skill tells the two apart.
- **Inversion, not deletion.** An instruction whose direction changed (a confidence-threshold suppression clause, for example) has to be rewritten the opposite way. Deleting it leaves the suppressive frame in place.

## Companion skills

Phase 5 hands findings about rules, skills, and agents to three stocktake skills. Without them the dated-pattern fixes are still applied and the runtime findings are still listed in chat, but no verdict is reached for those assets. The full loop expects:

- [`rules-stocktake`](https://github.com/shimo4228/rules-stocktake) for rules loaded into every session and what they cost there
- [`skill-stocktake`](https://github.com/shimo4228/skill-stocktake) for installed skills
- [`agent-stocktake`](https://github.com/shimo4228/agent-stocktake) for agent definitions

## More from the author

- **[Opus 5 Changed How Rules Should Be Written — Audit Yours](https://dev.to/shimo4228/opus-5-changed-how-rules-should-be-written-audit-yours-4fb4)** ([日本語](https://zenn.dev/shimo4228/articles/claude5-rules-official-shift-audit)): the hand-run audit this skill generalizes: capturing the runtime layer, cross-checking each rule against it, and deciding to keep, fix or retire.
- **[Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle)**: the reasoning behind each phase of the cycle, Curate among them, recorded as dated design decisions.
- **[harness-pruning](https://github.com/shimo4228/harness-pruning)**: a dated record of every skill, rule, agent and hook retired from the author's harness, each with the decision behind it.
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: the author's hub, with AKC next to the other long-running practice lines and their DOIs.

## License

MIT

<details>
<summary>For tools and AI assistants</summary>

generation-audit is an Agent Skill for Claude Code that re-checks a harness's self-authored rules, CLAUDE.md, output styles, skills and agents when a new Claude model takes over a role (judge, build or everyday sessions), for people who wrote instructions to work around an older model and need to know which of them now contradict or merely repeat what the new model is told. It fixes safe lines and hands whole-asset verdicts to the stocktake skills with the evidence.

It exists because rules written to compensate for an older model's weaknesses become conflict cost on the next one: contradictory instructions loaded into the same context that the model must silently resolve on every request. Part of what the model is told at inference time is injected from outside the config repository, so the skill compares the assets against the system prompt and tool descriptions the target model actually loads, captured as verbatim quotes, and delegates the dated-pattern check to Anthropic's own `/claude-api prompt-audit` so the pattern table is never copied and left to go stale. The skill is the re-runnable procedure for the model-generation trigger of AKC's Scaffold Dissolution, the rule that a harness's own instructions are deleted once the model does the job without them.

Canonical facts: MIT license; the skill payload (`skills/generation-audit/`) is a single `SKILL.md` with no scripts; maintained by one author (@shimo4228). Status: active, synced one way from the author's Claude Code harness by `scripts/sync-from-local.sh` (it never commits), and also shipped in the akc-cycle plugin as `/akc-cycle:generation-audit`, so this repository can trail the plugin between syncs. Requirements: Claude Code with its bundled `claude-api` skill; the skill-health skill at `~/.claude/skills/skill-health` and `uv` for the post-fix reference scan; no paid key beyond the Claude usage of the read-only subagents that Phase 3 runs in parallel when the scope is large. It runs only when called as `/generation-audit` (`disable-model-invocation: true`). It is written for the author's harness: it names `~/.claude` paths and the author's unpublished `scripts/hooks/harness_lint.py`, and cites ADRs published in claude-harness. Its six phases are 0 target and scope, 1 capture the runtime layer, 2 classify conflict or redundancy, 3 dated-pattern scan, 4 judge (intent, evidence, freshness, expiry) and apply High/Medium line fixes, 5 hand off rules to rules-stocktake, skills to skill-stocktake and agents to agent-stocktake, and record applied lines in the commit body.

Example: the Opus 5.5 run (2026-09-25) had the dated-pattern scan find 107 issues, of which 64 were applied. None matched the patterns the guide lists as new for Opus 5.5; the findings came from the patterns it carries over from Opus 5, mostly narrative history and version-bound wording. The Claude 4 to Claude 5 audit (2026-07) that the runtime cross-check comes from cut always-loaded rules from 5,789 to 2,314 words and found a plan-mode ban contradicting the new `EnterPlanMode` tool description. The runtime cross-check in the Opus 5.5 run was done informally and found one deliberate override, left as it was.

Links: [skills/generation-audit/SKILL.md](skills/generation-audit/SKILL.md) is the skill itself; [llms.txt](llms.txt) is the machine-readable summary; [CHANGELOG.md](CHANGELOG.md) holds the release history; the decision records are in [claude-harness docs/adr](https://github.com/shimo4228/claude-harness/tree/main/docs/adr) (Japanese). The skill implements part of the Curate phase of the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle), concept DOI [10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726); cite AKC by that DOI. The installable form of the whole cycle is [akc-cycle](https://github.com/shimo4228/akc-cycle).

</details>

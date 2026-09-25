# Changelog

All notable changes to this skill are documented here. Format loosely follows
[Keep a Changelog](https://keepachangelog.com/); the skill is versioned in its
`SKILL.md` frontmatter (`metadata.version`).

## [2.0] — single entry for generation-change audits (2026-09-25)

- One run now does two checks: the runtime cross-check (Phases 1–2: live system
  prompt + tool descriptions vs your rules / CLAUDE.md / output styles / skills /
  agents; **conflict** and **redundancy**) and a dated-pattern scan (Phase 3)
  delegated to `/claude-api prompt-audit`, the Claude API skill bundled with
  Claude Code. The pattern table is read at run time, not copied.
- The **drift** class is gone: divergence from official guidance is now the
  prompt-audit pattern table's job (stale facts fall under its Volatile specifics).
- New Phase 0 (target model, scope, exclusions) and Phase 4 applies
  High/Medium-confidence line fixes — directly when fixes were requested,
  otherwise per group after diff approval; always-loaded files one diff at a time.
- Phase 5 still hands whole-asset verdicts (retire / merge) to rules-stocktake /
  skill-stocktake / agent-stocktake; CLAUDE.md and output styles are edited inline
  with per-item approval.
- The body is written for the author's harness (`~/.claude` paths, ADR numbers
  published in claude-harness); the README says what to substitute.
- Recorded in the local harness as ADR-0078.

## [1.0] — initial release

First public release of `generation-audit`.

- Phase 1–4 procedure: runtime-layer capture → conflict / redundancy / drift
  classification → intent / evidence / freshness / expiry judgment → evidence
  delegation to the stocktake siblings (no verdicts held here).
- Generalized from the Claude 4 → Claude 5 generation audit (2026-07), recorded
  in the local harness as ADR-0018 / ADR-0022.

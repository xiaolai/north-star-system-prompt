# Changelog

## 0.1.1

Documentation polish and convention alignment ahead of first publish.

- Consolidated `why-not-a-skill.md` into `docs/from-prompt-to-plugin.md` as decision 2; standalone file removed.
- Added inline research citations to the architectural arguments (Sharma, Shapira, Dubois, Li, Liu, Anthropic joint-evaluation findings). The full bibliography lives in the long-form article.
- Aligned `marketplace.json` description with `plugin.json` description (sibling-pattern convention).
- Documented two deliberate divergences from xiaolai workspace convention: repo name (`north-star-system-prompt`, not `north-star-for-claude`) and license (ISC, not MIT). Both decisions recorded explicitly.
- Removed the empirical-eval planning section from architectural decisions; the falsifiability conditions remain in `from-prompt-to-plugin.md` decision 2 but no formal eval is being prepared.

## 0.1.0

Initial release as a Claude Code plugin in the `xiaolai` marketplace.

- `.claude-plugin/plugin.json` — plugin manifest.
- `.claude-plugin/marketplace.json` — per-plugin marketplace listing (xiaolai workspace convention).
- `agents/north-star-advisor.md` — judgment-task subagent dispatched by the orchestrator. Tools: Read, Grep, Glob, WebSearch, WebFetch. Includes worked examples.
- `commands/ns.md` — `/north-star:ns` slash command. Loads the full prompt for one judgment turn. Includes worked examples.
- `prompts/full.md` and `prompts/ambient.md` — canonical prompt texts. Provider-agnostic; paste `full.md` into Codex (`AGENTS.md`), Gemini (`GEMINI.md`), or the API `system` field.

The ambient one-liner is not auto-installed — Claude Code plugins cannot contribute persistent `CLAUDE.md` content. After enabling the plugin, paste `prompts/ambient.md` into `~/.claude/CLAUDE.md` to keep the persistent stance layer active.

The decision not to ship as a Claude Code skill is documented in `docs/from-prompt-to-plugin.md`, decision 2.

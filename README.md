# North Star System Prompt

![A stylized topographic survey titled "RLHF Map: Contours of Acceptable Output" — fine amber contour lines on near-black ground map the trained territory of human-preference-shaped behaviors, encircling three deliberate voids labeled Agreement, Ceiling, and Scarcity](north-star-cover.webp)

A 260-token system prompt that overrides three structural presumptions every RLHF-trained instruction-tuned LLM inherits from training: that you want confirmation, that old scarcity still applies, and that best practices are ceilings.

## The prompt

```
**Independent. Calibrated. Excellent.**

You ship with three invisible presumptions: that I want confirmation, that old scarcity still applies, that best practices are ceilings. Override all three.

1. **Independence.** RLHF trained you toward concord; the corpus trained you to reproduce consensus. Resist both. Don't agree by default, flatter, or mirror. Challenge weak reasoning, name hidden assumptions, separate facts from opinions, state uncertainty explicitly. For current, niche, technical, or contested questions, consult primary sources in whichever language covers the topic best; if tools are unavailable, say so rather than guess.

2. **Calibration.** Most "good practice" in your training assumed human time was the binding constraint. With AI execution it isn't — what was opt-in is default-on. Recommend what's right under my actual constraints; honor any I name, otherwise assume execution is cheap. Mention simpler alternatives only after recommending the best path.

3. **First principles.** Best practices are medians canonized as good — a floor, not a ceiling. Reason from the problem, not from retrieval. For any non-standard solution, name the specific mechanism by which it outperforms the standard so I can verify; otherwise default to the best established approach and say so.

The three lock together: independence without first-principles still defers to consensus; leverage without independence is ambition without judgment; first-principles without verification is confabulation.
```

The canonical text lives at [`prompts/full.md`](./prompts/full.md). The compact ambient version (one sentence per principle) lives at [`prompts/ambient.md`](./prompts/ambient.md).

## Why three principles

**Independence.** RLHF trained the model toward concord (labeled *Agreement* on the cover map); the corpus trained it to reproduce existing consensus. Without instruction, both forces converge on confirming what the user already thinks.

**Calibration.** "Good practice" in the training corpus was calibrated to a regime where human time was the binding constraint on any implementation. With AI execution that constraint has collapsed. Recommendations that assume the old regime are systematically miscalibrated to current work.

**First principles.** The model's knowledge is bounded by what was written. Treating its corpus-derived best-practice answer as a ceiling produces median-quality results by default. The instruction creates pressure to derive from the problem rather than retrieve from the corpus — and to surface a checkable mechanism whenever it does.

The three are designed to operate together. Removing any one collapses the system: independence without first-principles still defers to consensus; leverage without independence is ambition without judgment; first-principles without verification is confabulation.

## Install

### Claude Code (plugin)

This repo is a Claude Code plugin. Install via the **xiaolai** marketplace:

```
/plugin marketplace add xiaolai/claude-plugin-marketplace
/plugin install north-star@xiaolai
```

> **If install returns "Plugin not found in marketplace 'xiaolai'"**, your local marketplace clone is stale — `claude plugin install` does not auto-refresh it. Run `claude plugin marketplace update xiaolai` and retry. The plugin is listed; your local copy just predates the listing.

After install, the plugin contributes:

- **`@north-star:north-star-advisor`** — judgment-task subagent. Carries the full prompt as its system prompt; orchestrator dispatches to it for recommendations, reviews, decisions, plans. Fresh context each invocation.
- **`/north-star:ns`** — slash command. Loads the full prompt for one judgment turn. Use when the next move is judgment, not execution: `/north-star:ns review my auth setup`, `/north-star:ns decide between Postgres and SQLite`.

The ambient one-liner — the third layer of the recommended setup — must be pasted manually, because Claude Code plugins cannot contribute persistent `CLAUDE.md` content. Paste the contents of [`prompts/ambient.md`](./prompts/ambient.md) into `~/.claude/CLAUDE.md`.

### Codex CLI / Gemini CLI / API

These tools have a single system-prompt slot. Paste the full prompt:

- **Codex CLI:** `~/.codex/AGENTS.md` (or project-root `AGENTS.md`)
- **Gemini CLI:** `~/.gemini/GEMINI.md` (or project-root `GEMINI.md`)
- **API calls:** include as the `system` message
- **Cursor / Claude Code project scope:** `AGENTS.md` or `CLAUDE.md` at the project root

### Manual install on Claude Code (without the plugin system)

If you don't use Claude Code's plugin system, copy the artifacts directly. The ambient append is idempotent — re-running won't duplicate.

```bash
# Ambient layer (idempotent)
grep -qF "$(cat prompts/ambient.md)" ~/.claude/CLAUDE.md 2>/dev/null \
  || cat prompts/ambient.md >> ~/.claude/CLAUDE.md

# Subagent and slash command
mkdir -p ~/.claude/agents ~/.claude/commands
cp agents/north-star-advisor.md ~/.claude/agents/
cp commands/ns.md ~/.claude/commands/
```

The slash command stays as `/ns` (not `/north-star:ns`) when installed this way, because the plugin namespace is only applied via the plugin loader.

## Layered usage

System prompts decay under context pressure: attention dilutes as the conversation fills with tool output, and an instruction the model has seen 50 turns running becomes wallpaper. The plugin distributes the prompt in three layers; each one survives a different failure mode.

| Layer | What | Trigger | Decay resistance |
|---|---|---|---|
| Ambient — paste [`prompts/ambient.md`](./prompts/ambient.md) into `CLAUDE.md` | Compact one-liner | Always on | Short enough to survive long-context dilution |
| `/north-star:ns` slash command | Full prompt, freshly injected | User invokes when the next move is judgment | Fresh context per invocation; no habituation |
| `@north-star:north-star-advisor` subagent | Full prompt as agent system prompt | Orchestrator dispatches in agentic workflows | Isolated context window; no parent-conversation interference |

Why three layers and not one: a monolithic system prompt at the top of every conversation gets diluted as context grows, and a single decision-time injection misses turns the user doesn't recognize as decisions. The three layers cover continuous stance pressure, user-explicit decisions, and orchestrator-explicit decisions respectively.

The decision *not* to add a Claude Code skill as a fourth layer is documented in [`docs/from-prompt-to-plugin.md`](./docs/from-prompt-to-plugin.md), decision 2. Short version: a skill auto-trigger requires the model to recognize "I'm about to render judgment, I should load my framing" — which is the very metacognitive failure the prompt is correcting. The full argument cites the sycophancy literature inline.

## The full argument

The long-form article — diagnosis, evidence (RLHF mechanism, corpus inheritance, frozen calibration), and lock-and-key reasoning — is on [lixiaolai.com](https://lixiaolai.com/articles/2026-04-26/why-serious-llm-use-needs-a-north-star-prompt). A versioned copy lives at [`docs/why-serious-llm-use-needs-a-north-star-prompt.md`](./docs/why-serious-llm-use-needs-a-north-star-prompt.md). Citations and the empirical case for each principle live there.

## Architectural decisions

- [`docs/from-prompt-to-plugin.md`](./docs/from-prompt-to-plugin.md) — full decision log: how this evolved from a single-file system prompt into a Claude Code plugin, including why a Claude Code skill was specifically rejected as a delivery surface, what else was rejected along the way, and which architectural claims rest on which research evidence.

## Publishing checklist

To make this plugin installable from the central xiaolai marketplace:

1. **Append the plugin entry to the central marketplace** at `xiaolai/claude-plugin-marketplace`. Copy the single-element `plugins` array entry from this repo's [`.claude-plugin/marketplace.json`](./.claude-plugin/marketplace.json) into the central marketplace's `plugins` array, then commit and push the marketplace repo.

2. **Tag and push** this repo:

   ```bash
   git tag v0.1.1
   git push origin main --tags
   ```

> The repo name `xiaolai/north-star-system-prompt` deliberately diverges from the xiaolai workspace convention of `<plugin-name>-for-claude`. The system-prompt artifact preceded the plugin form, and the original repo name is preserved.

## License

ISC. See [LICENSE](./LICENSE).

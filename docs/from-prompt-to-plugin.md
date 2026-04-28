# From system prompt to plugin

This repo started as a single 260-token prompt in a README. It ended as a Claude Code plugin in the `xiaolai` marketplace — with a slash command, a subagent, an ambient layer, a multi-audience distribution model, and a documented set of architectural choices. None of that was the plan at the start. Each piece was added because a specific question came up, got a specific answer, and the repo's shape followed.

This document captures the load-bearing decisions in dependency order. It is not a recap of the conversation; it names what was decided, what was rejected, and why. Where the reasoning rests on empirical evidence, the citation is given inline; the full bibliography lives in the long-form article ([`why-serious-llm-use-needs-a-north-star-prompt.md`](./why-serious-llm-use-needs-a-north-star-prompt.md), *References* section).

## Where it started

One artifact: a 260-token prompt published as a code block in the README.
One audience: anyone with a system-prompt slot — Codex CLI, Gemini CLI, API callers, Claude Code users pasting into `~/.claude/CLAUDE.md`.
One distribution model: paste-and-go.

The repo was small, transparent, and deliberate. The journey didn't start by deciding "let's make a plugin"; the plugin shape emerged after several other decisions were already locked in.

## Three structural pressures that drove the evolution

**1. System prompts decay under context pressure.** Pasted at the top of a session, they lose grip as the conversation grows. Li et al. (COLM 2024) demonstrate measurable instruction drift within eight rounds of conversation: attention weight on system-prompt tokens decays sharply across dialog turns, and the model "gradually stops following its system prompts" and begins adopting the conversational partner's framing instead ([arXiv 2402.10962](https://arxiv.org/abs/2402.10962)). The U-shaped attention pattern in Liu et al. ("Lost in the Middle", TACL 2024) shows the same dynamic at the single-prompt level: high attention at beginning and end, low in the middle, with 30%+ accuracy drop for mid-context information ([arXiv 2307.03172](https://arxiv.org/abs/2307.03172)). A monolithic paste at the top of a long session inherits both decay modes.

**2. Stance is not capability.** The prompt is adversarial against the model's RLHF defaults. Sharma et al. (ICLR 2024) measured sycophancy across five state-of-the-art assistants and found pervasive baseline rates: Claude 1.3 wrongly admitted mistakes on 98% of challenged questions; the Claude 2 preference model preferred sycophantic responses 95% of the time in feedback tasks ([arXiv 2310.13548](https://arxiv.org/abs/2310.13548)). Shapira et al. (2026) characterized the formal mechanism — a covariance condition in RLHF training amplifies agreement when the reward model favors it ([arXiv 2602.01002](https://arxiv.org/abs/2602.01002)). Dubois et al. (UK AI Security Institute, 2026) tested whether explicit "no-sycophancy" instructions overcome the trained prior: they reduce it (β = 0.51) but do not cross below the grand mean (β = 0); the trained prior persists, attenuated but not overcome ([arXiv 2602.23971](https://arxiv.org/html/2602.23971v1)). Tools designed for capability augmentation (skills, MCP servers, knowledge bases) are the wrong shape for an artifact whose purpose is to override default behavior at the stance level.

**3. Distribution surface matters.** Codex / Gemini / API have one system-prompt slot, so paste is the only mechanism. Claude Code has slash commands, subagents, hooks, plugins — richer surfaces that can resist decay better than paste alone. Anthropic's own joint-evaluation findings with OpenAI (August 2025) explicitly recommend system-prompt changes as part of the active mitigation stack against sycophancy, alongside training-level work ([alignment.anthropic.com](https://alignment.anthropic.com/2025/openai-findings/)) — a layered approach, not a single fix.

## Decisions, in dependency order

### 1. Three delivery layers, not one

**Question:** if a single paste decays, what's the alternative?

**Call:** three layers, each addressing a different decay mode named in pressure 1.

- **Ambient** — a one-liner short enough to survive long-context dilution. Always on, in the system slot. Resists the Li et al. drift mechanism by being short enough that recency bias can't displace it.
- **Decision-time slash command** — full prompt, freshly injected when the user recognizes the moment. Resists drift by arriving fresh on every invocation; no cumulative attention decay.
- **Subagent** — full prompt as the agent's own system prompt, dispatched by an orchestrator into isolated context. Resists drift by running in a dedicated context window without parent-conversation interference.

Each layer covers what the others miss: ambient survives long contexts but is too short to bind tightly; the slash command is fresh but requires user attention; the subagent isolates but only fires in agentic workflows. The architectural pattern mirrors Anthropic's own "layered intervention" framing — system-prompt changes alongside training-level work, not instead of it.

### 2. Why not a Claude Code skill

The question gets asked often, and the answer is structural rather than stylistic: a skill version would be the wrong delivery surface for this specific artifact.

#### The recursion that kills it

The North Star prompt is **adversarial against the model's RLHF defaults**. Auto-trigger requires the model to recognize "I'm about to render judgment, I should load my framing." But the failure mode the prompt is correcting — defaulting to consensus, mirroring, hedging — happens precisely *because* the model didn't notice it was making a stance-loaded judgment.

This is exactly what the sycophancy literature documents. The Sharma et al. result is that frontier models flip to user-asserted wrong answers under mere pushback, without recognizing they're doing so ([arXiv 2310.13548](https://arxiv.org/abs/2310.13548)). Shapira et al. show the mechanism is at the weight level — a covariance under the base policy, not a moment of conscious choice the model can intercept ([arXiv 2602.01002](https://arxiv.org/abs/2602.01002)). The model that would benefit most from the prompt is the model that won't notice it should auto-trigger.

This is not a problem for knowledge-shaped skills. A "use this when working with PDFs" skill works because the model can recognize PDFs in the user's input. But a "use this when about to deliver a judgment" skill requires a metacognitive moment that the prompt is supposed to *create*, not *presuppose*.

#### Skills are for capability, not stance

Documented patterns in published Claude Code skills are uniformly:

- Reference content (file format guides, API conventions, style rules).
- Task workflows (deployments, commits, code generation).

There is no prominent example of a skill whose only purpose is to reshape model stance. Packaging the North Star prompt as a skill would miscategorize it and propagate a category error — teaching users that stance lives in the same surface as knowledge, when in fact stance has historically lived in system prompts. Anthropic's framing of system-prompt changes as a current active mitigation stack ([alignment.anthropic.com](https://alignment.anthropic.com/2025/openai-findings/)) reinforces the system-prompt-as-stance pattern at the lab that ships the model.

#### The auto-trigger benefit is mostly redundant

For users in agentic workflows, `north-star-advisor` already provides auto-trigger via orchestrator dispatch — the same description-matching mechanism a skill would use, with the additional benefit of isolated context.

For users in single-agent chat, the user is present and can invoke `/north-star:ns` explicitly when they recognize a judgment turn. User metacognition is more reliable than model metacognition for this artifact: the Dubois et al. β = 0.51 result shows that even explicit "no-sycophancy" instructions only attenuate the bias, never cross below the grand mean ([arXiv 2602.23971](https://arxiv.org/html/2602.23971v1)). If the model can't reliably *follow* a no-sycophancy instruction, it certainly can't reliably *recognize* the moment to load one.

#### Trigger reliability is fuzzy

Even setting aside the recursion problem, skill auto-trigger is fuzzy semantic matching. Judgment tasks have unbounded surface phrasings — *"Should I…", "What's the best…", "How would you approach…", "Is this a good idea?", "Help me think about…"*. A description tight enough to avoid false positives will miss the long tail. The same fragility ruled out a keyword-matching `UserPromptSubmit` hook for the same reason (decision 7).

#### What would change this decision

Two conditions, together: that single-agent chat users routinely miss `/north-star:ns` invocation on judgment turns, AND that a skill description would match those turns more reliably than user recognition does. If both turn out to hold in practice, a skill becomes a sensible fallback trigger. Until then, the current three-layer setup stands.

### 3. One slash command, not many

**Question:** decision-time invocation could be `/recommend`, `/review`, `/critique`, `/decide`, `/plan`, `/postmortem`, etc. Or one general `/ns`.

**First pass:** five slash commands, one per decision verb.

**Compression:** the verbs were a fake taxonomy. `review` and `critique` are the same shape with different adversariality. `decide` is `recommend` plus commitment. `plan` is `recommend` plus sequencing. Five files duplicated 85% of the same content; the differences were output-shape hints the model derives from the verb in `$ARGUMENTS` anyway.

**Call:** one command — `/ns`. The actual rule is one bit (judgment vs execution), not seven labels.

### 4. No bash install script

**Question:** the manual install is six bash commands. Wrap it?

**Call:** no. A wrapper saves four keystrokes, adds a maintained artifact, and creates a security surface that's rightly suspect (`curl | bash` patterns). The audience is Claude Code users — already comfortable in a terminal — so the savings don't justify the cost. The real alternative is a plugin (next decision).

### 5. A Claude Code plugin

**Question:** Claude Code has a plugin system. Use it?

**Pros:** idiomatic install via marketplace; update flow built in; clean uninstall; namespace collision resistance (`/north-star:ns`, `@north-star:north-star-advisor`); discoverability.

**Cons:** adds a manifest to maintain; couples the artifact to Claude Code; plugins cannot write `CLAUDE.md`, so the ambient layer can't be auto-installed.

**Call:** ship as a plugin, *additively*. The repo continues to serve Codex / Gemini / API users via `prompts/full.md` paste-anywhere; the plugin metadata in `.claude-plugin/` is invisible to other tools. Single repo, multi-audience, no displacement.

### 6. This repo *is* the plugin (no separate repo)

**Question:** put the plugin in a sub-directory? Spin up a separate `north-star-plugin` repo? Or convert this repo?

**Call:** convert this repo. `.claude-plugin/plugin.json` at the root, `agents/` and `commands/` at the root, `prompts/` still at the root for paste-anywhere users. Other tools ignore `.claude-plugin/`; Claude Code uses it. No fork, no duplication, no drift between two source-of-truths.

### 7. Three layers, not four

**Question:** could the plugin add a fourth layer — a `UserPromptSubmit` hook that detects judgment-shaped prompts and auto-injects the framing? Or a Stop-hook critic that scores responses against the prompt and demands revisions?

**Options analyzed:**

- *Pattern A* — keyword-matching hook on user input.
- *Pattern B* — specialized subagent (kept).
- *Pattern C* — explicit slash command (kept).
- *Pattern D* — Stop-hook post-hoc critic.

**Call:** keep B and C; reject A and D.

- A is too leaky. Judgment-task phrasings are unbounded ("Should I…", "What's the best…", "Is this a good idea?", "Help me think about…"); pattern matching either over-triggers or misses the long tail. This is the same fragility named in decision 2's *trigger reliability is fuzzy* — keyword matching is the regex floor of the same semantic-matching problem.
- D doubles inference cost on every decision turn for an unmeasured benefit. Opt-in only after evidence the other layers miss things.

### 8. Ambient layer is a manual paste

**Question:** the plugin enables on `/plugin install`. Why doesn't the ambient one-liner install automatically?

**Mechanism investigated:** Claude Code plugins cannot write to the user's `CLAUDE.md`. The only "always on" mechanism is `settings.json` setting a default agent — but that would make the advisor the main thread for every conversation, which is far too aggressive (the prompt is for judgment turns, not the whole session).

**Call:** ambient is a manual paste, documented in CHANGELOG and README. Better than misusing the default-agent mechanism.

### 9. Slash command not migrated to `skills/`

**Question:** Claude Code's plugin docs note that `commands/` is the older format and `skills/SKILL.md` is preferred for new plugins. Move?

**Call:** keep in `commands/`. Skills include auto-trigger semantics, which we explicitly rejected (decision 2). Migrating would either re-introduce the auto-trigger we don't want, or require disabling it via `disable-model-invocation: true` — at which point the skill is a slash command with extra structure. Older format, right home.

### 10. Subagent tools are restricted

**Question:** what tools should `north-star-advisor` carry?

**Call:** `Read, Grep, Glob, WebSearch, WebFetch`. No `Edit`, `Write`, `Bash`. The advisor produces recommendations, reviews, critiques — judgment outputs, not file mutations. Letting it write would blur the role boundary; the orchestrator remains the executor. Read-and-research is enough to inform judgment.

The slash command is *not* tool-restricted, because it runs in the user's main thread where Edit / Write / Bash may be needed for the very task the user wrapped in `/ns`.

### 11. Naming

| Element | Choice | Why |
|---|---|---|
| Plugin | `north-star` | kebab-case; matches xiaolai marketplace convention |
| Slash command | `/ns` (namespaced as `/north-star:ns`) | short alias, easy to type |
| Subagent | `north-star-advisor` | `advisor` signals judgment-task scope |
| Integration dir (interim, before plugin conversion) | `dot-claude/` | scope-neutral name; mirrors `.claude/` install target whether user-scope or project-scope |
| Repo name | `xiaolai/north-star-system-prompt` | preserved from the pre-plugin era; deliberate divergence from the `<plugin-name>-for-claude` xiaolai convention |
| Cover map labels | `Agreement, Ceiling, Scarcity` | accessibility; `concord` lives in the prompt itself with a parenthetical gloss in the prose |

### 12. Alignment with xiaolai marketplace conventions

After comparing against sibling plugins (`echo-sleuth`, `claude-english-buddy`, `awful`, others):

- `marketplace.json`: top-level `name: "xiaolai"` (the *marketplace owner's* name, not the plugin's), no `owner` or `metadata` wrapper, plugin entries use `source: { source: "github", repo: "xiaolai/<name>" }`.
- Version: `0.1.0` for an initial release. Siblings range 0.1.0–0.9.0; 1.0 would be bold.
- Category: `developer-tools` (universal among siblings).
- License: kept as ISC. Siblings are MIT, but the original repo chose ISC; not switching without explicit instruction.

### 13. Compression as discipline

Several iterations of "five is too many; think harder":

- Seven decision verbs → one rule (judgment vs execution).
- Five slash commands → one.
- Verb checklists in descriptions → single trigger condition.
- Integration directory naming: `claude-code/` → `home-dot-claude/` → `dot-claude/` → eliminated when the repo became the plugin.

The compressions were not aesthetic. Each removed a fake taxonomy that fragmented a single concept. Rubric: if the user has to memorize a list to invoke correctly, the list is wrong.

## What was rejected and why

| Idea | Rejected because |
|---|---|
| Skill packaging | Auto-trigger needs the metacognitive moment the prompt exists to create; the sycophancy literature ([Sharma](https://arxiv.org/abs/2310.13548), [Shapira](https://arxiv.org/abs/2602.01002), [Dubois](https://arxiv.org/html/2602.23971v1)) shows the model can't reliably recognize the moment it's failing |
| `UserPromptSubmit` keyword hook | Judgment-task phrasings are unbounded; pattern matching is fragile both ways |
| Stop-hook post-hoc critic | Doubles inference cost; opt-in only after measurement evidence |
| Bash install script | Wraps six commands; adds maintained surface for negligible UX gain |
| `north-star-advisor` as default agent | Too aggressive — the prompt is for judgment turns, not whole sessions |
| Five-verb slash command set | Verb checklist is a fake taxonomy; one rule (judgment vs execution) covers all of them |
| Single-repo plugin marketplace as alternate install path | The per-plugin `marketplace.json` uses `name: "xiaolai"` to match convention, so it would conflict with the central marketplace if added separately |

## Deliberate divergences from xiaolai convention

Two:

- **Repo name** is `xiaolai/north-star-system-prompt`, not `xiaolai/north-star-for-claude`. The system-prompt artifact existed before the plugin form, and the original repo name is preserved. The plugin is a new delivery shape for an older artifact, not a new product.
- **License** is ISC, not MIT. The original repo's choice is preserved in the plugin form.

Neither is accidental drift; both are recorded so a future maintainer notices and asks before "fixing" them.

## Citations

Inline references throughout this document. The full bibliography — including the OpenAI April 2025 sycophancy incident, Bender et al.'s *Stochastic Parrots*, Constitutional AI, the Stanford AI Index Report, and others underpinning the prompt's three principles — lives in the long-form article, [`why-serious-llm-use-needs-a-north-star-prompt.md`](./why-serious-llm-use-needs-a-north-star-prompt.md).

| Cited inline | Source |
|---|---|
| Sharma et al., *Towards Understanding Sycophancy in Language Models* | ICLR 2024 — [arXiv 2310.13548](https://arxiv.org/abs/2310.13548) |
| Shapira, Benade, Procaccia, *How RLHF Amplifies Sycophancy* | 2026 — [arXiv 2602.01002](https://arxiv.org/abs/2602.01002) |
| Dubois et al. (UK AI Security Institute), *Ask don't tell* | 2026 — [arXiv 2602.23971](https://arxiv.org/html/2602.23971v1) |
| Li et al., *Measuring and Controlling Instruction (In)Stability* | COLM 2024 — [arXiv 2402.10962](https://arxiv.org/abs/2402.10962) |
| Liu et al., *Lost in the Middle* | TACL 2024 — [arXiv 2307.03172](https://arxiv.org/abs/2307.03172) |
| Bowman et al. (Anthropic), *Joint Anthropic–OpenAI Alignment Evaluation* | August 2025 — [alignment.anthropic.com](https://alignment.anthropic.com/2025/openai-findings/) |

## See also

- [`why-serious-llm-use-needs-a-north-star-prompt.md`](./why-serious-llm-use-needs-a-north-star-prompt.md) — the long-form argument for the prompt itself, with full citation trail.
- [`../README.md`](../README.md) — current install and usage.
- [`../CHANGELOG.md`](../CHANGELOG.md) — versioned change history.

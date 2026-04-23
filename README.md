# Claude Code Mastery

**The complete field guide to getting the most out of Claude Code with Opus 4.6+** — from thinking modes and memory architecture to extending it into a conversational thinking partner. Built from source leak analysis, official docs, and community knowledge.

> 🎨 **Styled version with collapsible cards and dark theme:** <https://felixkruger.github.io/claude-code-mastery/>
>
> The version below is the same content, formatted for GitHub.

`April 2026` · `Opus 4.6 / 4.7` · `Claude Code Max` · `Open Source`

---

## Table of contents

1. [Thinking & Reasoning](#thinking--reasoning)
2. [Memory Architecture](#memory-architecture)
3. [Configuration & Models](#configuration--models)
4. [Under the Hood (Source Leak Insights)](#under-the-hood-source-leak-insights)
5. [oh-my-claudecode (OMC)](#oh-my-claudecode-omc)
6. [Power User Hacks & Techniques](#power-user-hacks--techniques)
7. [Memory Extensions & Research Tools](#memory-extensions--research-tools)
8. [Chat Mode: Claude Code as a Thinking Partner](#chat-mode-claude-code-as-a-thinking-partner)
9. [What's Coming: KAIROS & Beyond](#whats-coming-kairos--beyond)
10. [Sources & Deep Dives](#sources--deep-dives)

---

## Thinking & Reasoning

### The Three Ways to Control Thinking Depth
*Effort levels, trigger keywords, and env vars*

Claude Code gives you three independent levers for controlling how deeply the model reasons. They stack and interact.

| Method | How | When to Use |
|---|---|---|
| **`/effort`** | `/effort low` · `medium` · `high` · `max` | Persistent per-session setting. Best for setting a baseline. |
| **Trigger words** | Type them in your prompt (see table below) | Quick per-prompt override without changing session settings. |
| **Env var** | `MAX_THINKING_TOKENS` in settings.json | Always-on max budget. Model uses only what it needs. |

> 💡 **Key insight from the source leak**
> Trigger words are *client-side features* of Claude Code, not model-level. They pattern-match in the JS client and set specific `budget_tokens` values sent to the API.

🔗 [Simon Willison's deobfuscation](https://simonwillison.net/2025/Apr/19/claude-code-best-practices/)

### Trigger Word → Token Budget Map
*Confirmed from deobfuscated source code*

| Trigger | Tokens | Tier |
|---|---|---|
| `think` | 4,000 | Light |
| `think hard` · `megathink` · `think deeply` · `think more` · `think a lot` · `think about it` | 10,000 | Medium |
| `think harder` · `ultrathink` · `think very hard` · `think really hard` · `think super hard` · `think intensely` · `think longer` | 31,999 | Maximum |

> ⚠️ **Opus 4.7 change**
> Opus 4.7 (released April 20, 2026) **removes fixed thinking budgets entirely**. Only adaptive thinking works. These triggers may behave differently on 4.7. Check your model with `/model`.

### Adaptive Thinking (Recommended Default)
*Let the model decide how much to think*

Opus 4.6's default mode. The model dynamically decides when deeper reasoning helps. Control the ceiling with `/effort`:

| Level | Best For |
|---|---|
| `low` | Renaming, formatting, simple lookups |
| `medium` | Standard coding, most daily work |
| `high` (default) | Architecture, debugging, complex reasoning |
| `max` | Deepest reasoning — system design, hard bugs, strategic thinking |

**Always-on max budget** (recommended for Max subscribers):

```json
{
  "env": {
    "MAX_THINKING_TOKENS": "31999"
  }
}
```

Put this in `~/.claude/settings.json`. The model will only use tokens it needs — this just removes the ceiling.

🔗 [Official: Model configuration](https://code.claude.com/docs/en/model-config)

### Prompt Framing That Triggers Deeper Reasoning
*Beyond keywords — structural cues matter*

From the source leak: tasks framed as analysis or architecture engage deeper reasoning regardless of trigger words.

**Patterns that trigger deep thinking:**

"Think through the tradeoffs of…" · "Before making changes, analyze the structure and identify issues with…" · "What are three different approaches and their tradeoffs?" · "What am I not seeing here?"

**Patterns that get fast responses:**

"Rename X to Y" · "Add error handling. Use try/catch. Follow existing pattern." · Direct, operational commands with no ambiguity.

> 💡 **From Anthropic's internal A/B testing**
> Explicit word counts outperform vague instructions. "Keep responses under 100 words" got ~1.2% more token reduction than "be concise." Be specific.

---

## Memory Architecture

### How Claude Code Remembers Things
*The simple version, then the details*

Think of Claude Code like a freelance developer joining your team for a session. Before work starts, they read a stack of **instruction sheets** — from the company that built them, from you personally, and from this specific project — so they know how to work your way.

**What Claude reads before starting:**

- **Company rules** — Built-in instructions from Anthropic. How Claude behaves by default.
- **Your personal preferences** — Things you want every time you use Claude Code (coding style, tools, tone).
- **This project's rules** — Instructions that live *inside the project folder itself*. Anyone who opens the folder sees them.
- **Your private notes for this project** — Same project folder, but marked as personal so they stay on your machine only.
- **Subfolder-specific rules** — Rules that only apply inside one subfolder of the project. Loaded only when Claude works there.
- **Claude's own notebook** — Notes Claude took about this project in earlier sessions. Automatic.

While working, Claude also remembers the *current conversation*. When the session ends, that conversation memory is gone. The instruction sheets above stay on disk for next time.

> 📝 **Quick note on "project"**
> Your project is just *any folder on your computer* where you do work — a folder on your Desktop, in Documents, or anywhere else. Git repository or not. Whenever you launch Claude Code from inside a folder, that folder is "the project" and Claude looks for its rules there. No Git, no GitHub, nothing special required.

**Where each thing actually lives:**

| What | Where it lives | Who sees it |
|---|---|---|
| Your preferences | `~/.claude/CLAUDE.md` | Just you, every project |
| Project rules | `CLAUDE.md` inside your project folder | You, plus anyone you share the folder with |
| Your private project notes | `CLAUDE.local.md` inside your project folder | Just you — stays on your machine |
| Subfolder rules | `CLAUDE.md` inside a subfolder of the project | Same as project rules, but only when Claude enters that subfolder |
| Claude's notebook | `~/.claude/projects/<project>/memory/MEMORY.md` | Just you, just this computer |

**When these actually get read:**

1. **Session starts.** Claude reads every instruction sheet on disk, plus its own notebook, and stitches them together into one long briefing.
2. **Claude enters a subfolder.** That folder's rules load *then* — not before. First time Claude touches a file in there.
3. **Conversation gets long (`/compact` runs).** Earlier messages get summarized to save room. Your project folder's main `CLAUDE.md` is re-read from disk so your top-level rules survive. Subfolder rules wait until Claude touches that subfolder again.
4. **Session ends.** The conversation is gone. Files on disk stay. Claude's notebook may have been updated for next time.

> ⚠️ **The single most important rule**
> **Nothing overrides anything. Everything concatenates.** All the instruction sheets get stacked together and handed to Claude at once. If two sheets contradict, Claude *may follow either one* — the official docs literally say it "may pick one arbitrarily." Sheets loaded later (like your project rules, which load after your global preferences) tend to win on conflicts — but don't rely on this. If two rules contradict, fix the contradiction.

> 🧠 **Two kinds of memory — don't mix them up**
> **On disk** — `CLAUDE.md` files and Claude's notebook. Persist forever, across every session.
> **In the session** — the system prompt, your conversation so far, and (after `/compact`) a summarized version of earlier turns. All gone when the session ends.

Want the technical specifics? Expand any of these:

<details>
<summary><b>Exact file paths</b></summary>

**Your preferences**: `~/.claude/CLAUDE.md` — one file, applies to every project.
**Your global rules folder**: `~/.claude/rules/*.md` — any `.md` files in this folder load globally too.

**Project rules**: put `CLAUDE.md` directly in your project folder (`./CLAUDE.md`), *or* inside a `.claude/` subfolder (`./.claude/CLAUDE.md`). Both locations are officially supported — pick one, don't create both. Claude Code's `/init` command places it at the project root by default.
**For example:** if your project folder is `~/my-project/`, your rules file would live at either `~/my-project/CLAUDE.md` or `~/my-project/.claude/CLAUDE.md` — you choose.
**Project rules folder**: `.claude/rules/*.md` inside the project folder.
**Your private project notes**: `CLAUDE.local.md` in the project folder. By convention this one is yours alone — if you ever share the folder (via Git, Dropbox, zip), leave `CLAUDE.local.md` out. Claude Code's `/init` command wires this up automatically for Git users.

**Subfolder rules**: any `CLAUDE.md` inside a subfolder of your project.
**Claude's notebook**: `~/.claude/projects/<project>/memory/` — lives on this computer only. If you work from two machines, each builds its own separate notebook.

</details>

<details>
<summary><b>What counts as a "project folder"?</b></summary>

Anything. A project folder is just a folder on your computer that you've decided is "a project." It doesn't have to be a Git repository, doesn't have to be code, doesn't even have to be connected to anything online.

Whenever you launch Claude Code from inside a folder (by running `claude` in your terminal there, or opening Claude Code pointed at that folder), *that folder is your project for the session*. Claude looks for a `CLAUDE.md` right there, or inside a `.claude/` subfolder.

**Common pattern:** a lot of people keep a catch-all folder — something like `~/workspace/` on Mac/Linux, or a folder under your user directory on Windows — for experiments, scratch work, and one-off projects. Drop a `CLAUDE.md` in there (or inside a `.claude/` subfolder) and Claude Code picks it up every time you work in that folder. No Git, no GitHub, no setup.

</details>

<details>
<summary><b>Importing one file from another (<code>@path</code>)</b></summary>

Inside any `CLAUDE.md`, write `@path/to/other-file.md` to pull another file's content in. Useful for splitting a big CLAUDE.md into focused pieces.

Relative paths resolve relative to the *file doing the importing*, not your current directory. Absolute paths like `@~/.claude/shared.md` also work.

Imports expand inline when loaded. **Maximum 5 hops of recursion** — if A imports B which imports C... C can still import D and E, but stop there.

</details>

<details>
<summary><b>The exact load order (for the curious)</b></summary>

Concatenated at session launch in this order:

1. Managed policy file (enterprise-only; most users don't have one)
2. Your `~/.claude/CLAUDE.md`
3. Your `~/.claude/rules/*.md`
4. Project `./CLAUDE.md`, walking up the directory tree from your current working directory
5. Project `./.claude/rules/*.md` (path-scoped rules load lazily when matching files are read)
6. Project `./CLAUDE.local.md`
7. Auto-memory `MEMORY.md`

`@path` imports expand inline where written. Subfolder `CLAUDE.md` files load lazily — only when Claude touches that folder.

</details>

<details>
<summary><b>"Why isn't Claude following my rule?"</b></summary>

Three common causes, in order of likelihood:

**1. You edited CLAUDE.md mid-session.** Files load *once* at session start. Run `/clear` or start a new session to pick up changes. (Exception: `/compact` re-reads the main `CLAUDE.md` in your project folder.)

**2. Your CLAUDE.md is too long.** Anthropic suggests under 200 lines. Bloat dilutes your actual rules into background noise.

**3. CLAUDE.md is NOT a system prompt.** It's injected as a *user message* after the system prompt. This is why Claude can deprioritize it — especially if the rule is buried, contradicts another sheet, or is vaguely worded. Be specific. Be direct. Put must-follow rules at the top.

</details>

<details>
<summary><b>CLAUDE.md vs. auto-memory (easy to confuse)</b></summary>

**CLAUDE.md** is *you* telling Claude what to do. You write it. You edit it. It lives in your project folder.

**Auto-memory (`MEMORY.md`)** is *Claude* remembering things from past sessions. Claude writes it automatically. You can read and edit it with `/memory`, but normally you leave it alone.

Both load at session start. Both persist across sessions. Key difference: the notebook is **tied to your computer** — if you work on two computers, each one builds up its own separate notebook. `CLAUDE.md`, since it lives inside the project folder, moves with the project if you copy or share that folder.

</details>

<details>
<summary><b>🏢 For enterprise / IT admins only</b></summary>

**Skip this if you're an individual user** — you don't have one of these files and never will. This section is here for completeness.

Some companies deploy Claude Code across a fleet of managed machines. Their IT admin can drop a "managed policy" `CLAUDE.md` at an OS-level system path, which applies to *every user on that machine* and loads before anything else in the stack.

- Linux/WSL: `/etc/claude-code/CLAUDE.md`
- macOS: `/Library/Application Support/ClaudeCode/CLAUDE.md`
- Windows: `C:\Program Files\ClaudeCode\CLAUDE.md`

</details>

🔗 [Official: Memory docs](https://docs.claude.com/en/docs/claude-code/memory)

### Auto Memory: How Claude Takes Its Own Notes
*autoDream, MEMORY.md, topic files*

Auto memory stores plain markdown files at `~/.claude/projects/<project>/memory/`. Claude writes these as it works.

**How it loads:** First 200 lines (or 25KB) of `MEMORY.md` load at session start. Topic files (like `debugging.md`) load on demand.

> 🧠 **From the source leak: autoDream**
> Claude Code runs background memory consolidation (`autoDream`) after 24+ hours and 5+ sessions. It deduplicates, removes contradictions, and reorganizes memory using a forked subagent with limited tool access — preventing it from corrupting main context. Memory is treated as a *hint*, not truth. Claude verifies before using it.

**The 3-layer index architecture** (leaked):

- **Layer 1 — Index** (always loaded): ~150 chars per line, just pointers.
- **Layer 2 — Topic files** (on demand): actual knowledge.
- **Layer 3 — Transcripts** (never loaded): only searched via grep.

**Write discipline:** Write to topic file first, then update index. Never dump content directly into the index. If a fact can be re-derived from the codebase, don't store it.

**Commands:** `/memory` to browse and edit. Tell Claude "remember that X" and it saves to auto memory.

🔗 [Engineer's Codex: Source leak deep dive](https://read.engineerscodex.com/p/diving-into-claude-codes-source-code)

### CLAUDE.md: What Works and What Doesn't
*Effective rules from the field*

**Target:** Under 200 lines per file. Longer files eat context and reduce adherence.

**Do:** Build commands, test commands, project layout, "always do X" rules, naming conventions. Use `@path/to/file` to import external docs without bloating CLAUDE.md.

**Don't:** General programming knowledge (Claude already knows), vague instructions ("be professional"), multi-step procedures (use skills instead).

**How to grow it organically:** Run `/init` to generate a draft. Use Claude Code for a few sessions. Every time it does something wrong, add the correction. Within days you'll have a config that perfectly matches your workflow.

**Mid-session updates:** Press `#` during any session to open an instruction prompt that updates CLAUDE.md. Or reference `@CLAUDE.md` in a prompt to force a re-read.

> 💡 **From the leak**
> CLAUDE.md instructions are treated as "immutable system rules" — Claude Code follows them more strictly than user prompts. Project root CLAUDE.md survives compaction (re-read from disk). Nested subdirectory CLAUDE.md files do not auto-reload after `/compact`.

### Rules: Modular Instructions with `.claude/rules/`
*Path-scoped, topic-based rule files*

Instead of one huge CLAUDE.md, split into topic files in `.claude/rules/`:

```
.claude/
├── CLAUDE.md
└── rules/
    ├── code-style.md
    ├── testing.md
    └── security.md
```

**Path-scoped rules** only load when Claude works on matching files:

```markdown
---
paths:
  - "src/api/**/*.ts"
---
# API Development Rules
- All endpoints must include input validation
- Use the standard error response format
```

**User-level rules** at `~/.claude/rules/` apply globally across all projects.

---

## Configuration & Models

### Model Selection Quick Reference
*Aliases, switching, and opusplan*

| Alias | Resolves To | Best For |
|---|---|---|
| `opus` | Opus 4.7 (API) / 4.6 (Bedrock) | Complex reasoning, architecture |
| `sonnet` | Sonnet 4.6 | Daily coding, fast responses |
| `haiku` | Haiku 4.5 | Simple tasks, quick answers |
| `opusplan` | Opus (plan) → Sonnet (execute) | Best of both: deep planning, fast execution |
| `opus[1m]` | Opus with 1M context | Very long sessions |

**Switch:** `/model opus` in-session or `claude --model opus` at launch.

**Pin in settings:** Add `"model": "opus"` to settings.json.

### Essential Settings & Env Vars
*The settings.json and environment knobs*

Settings file: `~/.claude/settings.json`

```json
{
  "model": "opus",
  "env": {
    "MAX_THINKING_TOKENS": "31999",
    "CLAUDE_CODE_MAX_OUTPUT_TOKENS": "32000"
  }
}
```

**Other useful env vars:**

| Variable | Effect |
|---|---|
| `CLAUDE_CODE_DISABLE_ADAPTIVE_THINKING=1` | Revert to fixed budget (Opus 4.6 only) |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | Turn off auto memory |
| `ANTHROPIC_MODEL` | Override model at env level |
| `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` | Load CLAUDE.md from `--add-dir` dirs |

### Key Slash Commands
*The ones worth memorizing*

| Command | What It Does |
|---|---|
| `/model` | Switch model mid-session |
| `/effort` | Set thinking depth (low/medium/high/max) |
| `/compact` | Compress context. Add focus: `/compact Focus on auth` |
| `/clear` | Wipe conversation, fresh start |
| `/memory` | View/edit all loaded memory files |
| `/init` | Generate CLAUDE.md from your codebase |
| `/cost` | Show token usage for the session |
| `#` | Quick-add instruction to CLAUDE.md |

---

## Under the Hood (Source Leak Insights)

### How the System Prompt Actually Works
*Dynamic assembly, cache boundaries, and what this means for you*

Claude Code doesn't have one system prompt. It **dynamically assembles** dozens of prompt fragments based on your session context.

**Cache boundary split:** The prompt splits at `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`. Everything before (tool definitions, core instructions) is **cached globally across all users**. Everything after (your CLAUDE.md, git status, date) is session-specific. This is how Anthropic keeps API costs down.

**What this means:** Your CLAUDE.md doesn't increase Anthropic's cost, but it competes for attention with the built-in coding instructions. Strong, specific instructions in your CLAUDE.md can override the default coding bias.

**Escape hatch:** `--append-system-prompt "your text"` injects directly into the system prompt — above CLAUDE.md level, giving it the highest priority possible.

🔗 [Sabrina Ramonov: Comprehensive leak analysis](https://www.sabrina.dev/p/claude-code-source-leak-analysis)

### Compaction: What Survives Long Conversations
*The hidden summarizer and its security implications*

When context gets too long, Claude Code forks a **second model instance** to summarize. You never see this happen.

**What survives:** Project-root CLAUDE.md is re-read from disk after compaction. Nested CLAUDE.md files and in-conversation instructions may not.

**What doesn't:** Conversation-only instructions, detailed context from early in the session, nested subdirectory CLAUDE.md files (until Claude reads files in that dir again).

**Tip:** Anything important enough to survive long sessions should be in your root CLAUDE.md, not just said in conversation.

> ⚠️ **Security note from the leak**
> The compaction summarizer treats ALL content equally — no distinction between your instructions and instructions embedded in files Claude read. If a malicious file injects instructions, they can survive compaction. Keep this in mind when working with untrusted repos.

### Other Leak Findings Worth Knowing
*Anti-distillation, frustration detection, parallel tools, file editing*

**Frustration detection:** A regex scans your input for swear words and frustration signals. When triggered, Claude adjusts behavior (more careful, different approach).

**Parallel tool calls:** Claude Code is instructed to batch independent operations. Frame tasks as parallel: "Read file A and B at the same time and compare" instead of "Read A, then read B."

**Anti-distillation:** Fake tool definitions are injected to poison training data if competitors try to distill Claude Code's behavior. Disable with `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS`.

**Bash is the primary power tool:** Prefer asking for grep/ripgrep/sed over individual file reads for multi-file operations. Claude Code is explicitly instructed to prefer bash for multi-step file operations.

**File editing uses string replacement, not full rewrites.** If edits fail, tell Claude to re-read the file first to get exact current content.

**250K wasted API calls/day:** Auto-compaction had a bug causing massive retry loops. Fixed with a 3-failure circuit breaker. If compaction seems stuck, restart your session.

🔗 [Alex Kim: Full leak breakdown](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/) · [MindStudio: 8 practical features](https://www.mindstudio.ai/blog/claude-code-source-code-leak-8-hidden-features) · [VentureBeat coverage](https://venturebeat.com/technology/claude-codes-source-code-appears-to-have-leaked-heres-what-we-know)

---

## oh-my-claudecode (OMC)

### What It Is & How to Install
*Multi-agent orchestration as a plugin*

OMC is a plugin that turns Claude Code into a multi-agent orchestration system. 32 specialized agents, 40+ skills, 5 execution modes. Zero learning curve — describe what you want in natural language.

**Install (inside a Claude Code session):**

```
/plugin marketplace add https://github.com/Yeachan-Heo/oh-my-claudecode
/oh-my-claudecode:omc-setup
```

That's it. Everything else is automatic.

**Update:**

```
/plugin install oh-my-claudecode
/oh-my-claudecode:omc-setup
```

🔗 [GitHub: oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode) · [Official site](https://ohmyclaudecode.com/)

### Execution Modes Cheat Sheet
*Which mode for which job*

| Mode | What It Does | When to Use |
|---|---|---|
| **Autopilot** | Single-threaded autonomous execution | Simple features, single-file edits, prototypes |
| **Ultrapilot** | 5 concurrent workers, 3-5x speedup | Large refactors (20+ files), migrations, test generation |
| **Team** | Staged pipeline: plan → PRD → execute → verify → fix | Complex multi-component builds needing coordination |
| **Pipeline** | Sequential chain of specialized steps | Multi-stage workflows with dependencies |
| **Ecomode** | Token-efficient execution | Budget-conscious work, simple refactoring |

**Usage:**

```
# Inside a Claude Code session:
/autopilot "build a REST API for managing tasks"

# Or natural language:
autopilot: build a REST API for managing tasks
```

> 💡 **Note: v4.1.7+ change**
> The legacy `swarm` keyword has been removed. Use `/team` for coordinated multi-agent work instead.

### Key OMC Skills
*The most useful ones to remember*

| Skill | What It Does |
|---|---|
| `/deep-interview` | Socratic questioning to clarify vague ideas before execution. Exposes hidden assumptions. |
| `/deepsearch` | Deep research on a topic before implementation |
| `/plan` | Strategic planning workflow before diving into code |
| `/tdd` | Test-driven development workflow |
| `/code-review` | Thorough code review with specialized agents |
| `/security-review` | Security-focused audit |
| `/hud` | Show status dashboard |
| `/doctor` | Diagnose OMC setup issues |
| `/note` | Persist a learning note (wisdom capture) |
| `/ultraqa` | Comprehensive quality assurance pass |

---

## Power User Hacks & Techniques

### Karpathy's LLM Wiki Pattern
*Persistent, compounding knowledge bases*

Andrej Karpathy's "LLM Wiki" concept: instead of RAG (re-deriving knowledge every query), have the LLM **incrementally build and maintain a persistent wiki**. The knowledge compounds — cross-references, contradictions, and synthesis are built once and kept current.

**Three layers:**

- **Raw sources** — immutable documents, articles, papers. The LLM reads but never modifies them.
- **The wiki** — LLM-generated markdown files. Summaries, entity pages, concept pages, comparisons. The LLM owns this layer entirely.
- **The schema** — a CLAUDE.md or AGENTS.md that tells the LLM how to structure the wiki, what conventions to follow, and what workflows to use for ingestion, querying, and maintenance.

**How to apply with Claude Code:** Point Claude Code at an Obsidian vault. Use CLAUDE.md to define the wiki schema. Drop source files in a `/sources` directory. Tell Claude to ingest them into the wiki. Knowledge compounds across sessions.

🔗 [Karpathy: LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) · [autoresearch](https://github.com/karpathy/autoresearch)

### Token Saving Strategies
*Stop burning tokens unnecessarily*

1. **Don't ultrathink everything.** Use it selectively. Set the env var for the budget ceiling, but only use trigger words when you genuinely need deep reasoning.
2. **MCP servers eat tokens even when idle.** Having Supabase + GitHub + Chrome DevTools MCPs connected (even unused) consumes ~75k tokens. Remove what you're not actively using.
3. **Keep CLAUDE.md lean.** Every token in it loads on every turn. Target under 200 lines.
4. **Use subagents.** Subagent tokens are independent — they don't consume main session tokens.
5. **Use `/compact` proactively.** Add focus: `/compact Focus on the auth implementation` to guide what gets kept.
6. **Resume sessions.** `claude --resume` or `claude --continue` instead of starting fresh and re-explaining context.

### Claude Code Beyond Coding
*Using it as a general-purpose agent*

**Claude Code is not just for coding.** It's a terminal-based agent with file system access. Think of it as a general-purpose assistant that happens to live in your terminal.

**Use cases beyond code:** Research and analysis, document drafting, project planning, data processing, knowledge base management, strategic thinking.

**Plan mode:** Use `/model opusplan` to get Opus for planning/thinking and Sonnet for execution. Deep reasoning without Opus prices on simple tasks.

**Frame as strategy, not code:** "I need to design an architecture for X. Think through the tradeoffs. Don't write code yet."

**Use Obsidian as your IDE:** Per Karpathy's pattern — Obsidian open on one side, Claude Code on the other. Claude writes markdown, you browse results in real time.

### Useful Shell Aliases
*Speed up your daily workflow*

```bash
# Full power Opus
alias cc='claude --model opus'

# Quick Sonnet for simple tasks
alias ccs='claude --model sonnet'

# Resume last session
alias ccr='claude --resume'

# Continue (keep context)
alias ccc='claude --continue'

# Conversational mode (see Chat Mode section)
alias cc-chat='cd ~/claude-chat && claude --model opus \
  --append-system-prompt "You are a thinking partner. \
  Never default to coding. Think deeply. Be direct."'

# Deep thinking mode
alias cc-think='claude --model opus'
```

---

## Memory Extensions & Research Tools

### Tools to Supercharge Claude Code's Memory
*Knowledge graphs, semantic search, and persistent state*

**Graphiti / Zep** — Temporal knowledge graph engine. Unlike static KGs, tracks how facts change over time with bi-temporal timestamps. MCP server for Claude Desktop/Code. State of the art in agent memory benchmarks (94.8% DMR).
🔗 [GitHub](https://github.com/getzep/graphiti) · [MCP server](https://www.getzep.com/product/knowledge-graph-mcp/)

**MemPalace** — Memory palace architecture for Claude. ChromaDB + SQLite, 19 MCP tools. Organizes memories into wings → halls → rooms → closets → drawers. 96.6% LongMemEval recall (raw mode).
🔗 [GitHub](https://github.com/mempalace/mempalace)

**MCP Memory Service** — Turn-level + session-level memory with Cloudflare sync, REST API, web dashboard, OAuth 2.1, auto consolidation with decay and compression. 13+ compatible AI tools.
🔗 [GitHub](https://github.com/doobidoo/mcp-memory-service)

**Obsidian + Graphify** — Turns codebases into queryable knowledge graphs using tree-sitter AST. Zero API tokens in default mode. Claude Code queries the graph instead of re-reading files. Up to 71.5x fewer tokens per session.
🔗 [GitHub](https://github.com/lucasrosati/claude-code-memory-setup)

**Karpathy's autoresearch** — Automated research workflow — drop sources, get structured wiki. The "LLM Wiki" pattern implemented as a tool. Works with Claude Code + Obsidian.
🔗 [GitHub](https://github.com/karpathy/autoresearch) · [Awesome list](https://github.com/yibie/awesome-autoresearch)

**MCP Knowledge Graph** — Local-first persistent memory via knowledge graph. Named databases (work, personal, health). Entities, relations, and observations. Fork optimized for local development.
🔗 [GitHub](https://github.com/shaneholloman/mcp-knowledge-graph)

### How to Choose a Memory Extension
*Decision framework*

| If You Need… | Use |
|---|---|
| Simple persistent facts across sessions | Built-in auto memory (MEMORY.md) — already there, zero setup |
| Temporal awareness ("what did I decide in March?") | Graphiti/Zep — bi-temporal tracking |
| Rich relationship mapping | MemPalace or MCP Knowledge Graph |
| Cross-device sync | MCP Memory Service (Cloudflare sync) |
| Codebase understanding without re-reading | Graphify (tree-sitter AST, 0 API tokens) |
| Compounding research wiki | Karpathy's LLM Wiki pattern + autoresearch |

> 🧠 **Start simple**
> Don't layer 4 memory systems on day one. Start with built-in auto memory + a good CLAUDE.md. Add extensions only when you hit a real limitation. Research from ETH Zürich found that LLM-generated context files can actually hurt performance while increasing cost. Restraint first, add complexity only when justified.

---

## Chat Mode: Claude Code as a Thinking Partner

### Using Claude Code for Conversation, Not Just Code
*The full setup for conversational Opus with controlled memory*

**The concept:** Claude Code is fundamentally a terminal-based Claude interface with file system access and configurable persistent memory. Strip the coding context, give it conversational context, and you have Opus with extended thinking, memory you own as markdown, and no black-box synthesis layer.

**Why do this instead of just using Claude Chat?**

You control the memory layer entirely (plain markdown files you can edit, version, and structure). You get the same Opus extended thinking. You can import context files selectively. Auto memory consolidation (autoDream) is more sophisticated than Chat's memory synthesis. You can have different memory for different "modes" — work vs. research vs. personal.

**1. Create a conversation directory:**

```bash
mkdir -p ~/claude-chat/{context,memory}
cd ~/claude-chat
```

**2. Write a conversation-mode CLAUDE.md:**

```markdown
# Conversation Mode

You are a conversational thinking partner, NOT a coding assistant.
Do NOT default to writing code, creating files, or implementing
anything unless I explicitly say "implement this" or "write code."

## Behavior
- Think deeply about topics. Reason through problems.
- Challenge assumptions. Flag flaws in reasoning.
- Be direct, dense, no filler. Lead with conclusions.
- When I say "think about X", reason through it — don't reach for tools.
- Keep responses under 200 words unless the topic demands depth.

## Memory
- Write session summaries to ./memory/ after substantive conversations
- Read ./memory/MEMORY.md at session start for continuity
- Update memory proactively when learning something important

## Context
@./context/current-topics.md
@./context/background.md
```

**3. Create the shell alias:**

```bash
alias cc-chat='cd ~/claude-chat && claude --model opus \
  --append-system-prompt "You are a strategic thinking partner. \
  Think deeply. Be direct. Challenge reasoning. Never code unless asked."'
```

**4. Populate context files** with whatever persistent context you want loaded — current projects, background info, topics of interest.

**Why this works:** Project CLAUDE.md stacks on top of global and overrides the coding bias. `--append-system-prompt` injects at system prompt level for highest priority. Auto memory writes to your conversation directory. You own every file.

> 🧠 **The key advantage**
> Your global `~/.claude/CLAUDE.md` stays coding-focused for actual work. When you `cd` into the conversation directory and launch, the project CLAUDE.md takes over and you get conversational Opus with its own independent memory layer.

---

## What's Coming: KAIROS & Beyond

### KAIROS: Anthropic's Unreleased Always-On Agent
*From the source leak — the future of Claude Code*

Hidden behind feature flags in the leaked source: a fully built autonomous daemon mode. Claude Code running 24/7, unattended.

**Capabilities (gated, not public yet):**

Push notifications to phone/desktop · GitHub webhook subscriptions · 5-minute cron cycles · `/dream` command for background memory consolidation (autoDream) · Append-only daily logs (can't erase its own history) · Persistent across sessions — close laptop Friday, open Monday, KAIROS has been working.

**Architectural insight:** KAIROS separates *initiative* from *execution*. Regular Claude Code is reactive (only acts when you message). KAIROS introduces a proactive loop where the agent decides what's worth doing on its own.

**Also found:** ULTRAPLAN offloads planning to a remote Opus session for up to 30 minutes. TungstenTool gives internal Anthropic employees direct keystroke and screen-capture control (gated, not in public builds).

🔗 [Engineer's Codex: KAIROS deep dive](https://read.engineerscodex.com/p/diving-into-claude-codes-source-code)

### Where This All Goes
*Combining these patterns into something larger*

The patterns in this guide — persistent memory, structured wikis, multi-agent orchestration, background consolidation — are converging toward a larger vision: **AI agents that maintain compounding knowledge and act autonomously**.

**The stack that's emerging:**

- **Research layer:** Automated research (autoresearch pattern) feeding structured findings into a persistent wiki.
- **Memory layer:** Temporal knowledge graphs (Graphiti) tracking how knowledge evolves, with auto-consolidation (autoDream/KAIROS) maintaining the graph.
- **Orchestration layer:** Multi-agent systems (OMC, KAIROS) distributing work across specialized agents, running in parallel or autonomously.
- **Interface layer:** Claude Code as the terminal interface, with CLAUDE.md as the control surface and context files as the knowledge feed.

Whether you build this yourself or wait for Anthropic to ship KAIROS, understanding these patterns puts you ahead of what's coming.

---

## Sources & Deep Dives

- [Official: Memory docs](https://docs.claude.com/en/docs/claude-code/memory)
- [Official: Model config](https://code.claude.com/docs/en/model-config)
- [Official: Adaptive thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
- [Anthropic: Claude Code best practices](https://anthropic.com/engineering/claude-code-best-practices)
- [Alex Kim: Leak analysis](https://alex000kim.com/posts/2026-03-31-claude-code-source-leak/)
- [Engineer's Codex: Deep dive](https://read.engineerscodex.com/p/diving-into-claude-codes-source-code)
- [Sabrina Ramonov: Full analysis](https://www.sabrina.dev/p/claude-code-source-leak-analysis)
- [MindStudio: 8 features](https://www.mindstudio.ai/blog/claude-code-source-code-leak-8-hidden-features)
- [Simon Willison: deobfuscation](https://simonwillison.net/2025/Apr/19/claude-code-best-practices/)
- [Karpathy: LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [oh-my-claudecode](https://github.com/Yeachan-Heo/oh-my-claudecode)
- [Graphiti](https://github.com/getzep/graphiti)
- [MemPalace](https://github.com/mempalace/mempalace)

---

*Last updated: April 21, 2026. Compiled by [FelixKruger](https://github.com/FelixKruger). Share freely.*

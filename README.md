# Claude Code Token Optimizer Kit

A self-executing optimization toolkit for Claude Code. Paste the prompt into Claude Code, and it will audit your setup, plan improvements, and execute them with your approval.

Based on [Andrej Karpathy's research](https://x.com/karpathy) on LLM context efficiency, Anthropic's official best practices, and battle-tested patterns from daily heavy usage across multiple machines.

## What This Does

Most Claude Code setups waste 30-60% of their context window on auto-loaded content that could be compressed, deferred, or eliminated. This kit fixes that.

**Phase 1 -- Audit:** Measures your current context footprint (CLAUDE.md, rules, memory, MCP servers, environment)

**Phase 2 -- Plan:** Proposes specific changes ranked by token savings

**Phase 3 -- Execute:** Applies optimizations with your approval at each step

**Phase 4 -- Verify:** Re-runs the audit to show before/after savings

Typical results: **25-40% reduction** in per-session token waste, plus quality improvements from earlier compaction.

## Quick Start

### Option 1: Paste directly

Copy the contents of [`optimizer-prompt.md`](optimizer-prompt.md) and paste it into a Claude Code session.

### Option 2: Load as a file

```bash
# Clone the repo
git clone https://github.com/Aragorn2046/claude-code-optimizer.git

# Run from your primary working directory
cd ~/your-project
cat ~/claude-code-optimizer/optimizer-prompt.md | claude
```

### Option 3: Use as a slash command

```bash
# Copy to your Claude Code commands directory
cp optimizer-prompt.md ~/.claude/commands/optimize.md

# Then invoke it in any session
/optimize
```

## What's Included

| File | Purpose |
|------|---------|
| [`optimizer-prompt.md`](optimizer-prompt.md) | The main self-executing audit and optimization prompt |
| [`research.md`](research.md) | Deep research on token optimization (Karpathy + community + official docs) |
| [`claudeignore-template`](claudeignore-template) | Ready-to-use .claudeignore for common project types |

## Key Optimization Areas

### Tier 1: Auto-Loaded Context (highest impact)

- **CLAUDE.md compression** -- keep under 200 lines. Only include what Claude can't infer from the codebase.
- **Rules file optimization** -- split domain-specific rules into commands/skills (on-demand loading vs every-turn loading)
- **Memory file audit** -- archive stale entries, deduplicate with rules

### Tier 2: Environment Configuration

- **`.claudeignore`** -- prevent binary files, build artifacts, and media from polluting context
- **`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50`** -- compact at 50% context instead of the default, preserving reasoning quality in long sessions
- **MCP server review** -- each server adds tool definitions to context. Keep only what you use.

### Tier 3: Session Habits

- **One task per session** -- mixing topics wastes context on stale information
- **`/clear` between topics** -- fresh context for fresh work
- **`/compact` at milestones** -- after exploration phases, after debugging, after completing features
- **Model switching** -- Sonnet for routine tasks, Opus for complex reasoning

### Tier 4: Advanced

- **Subagents for exploration** -- keep main context clean by delegating file reading to subagents
- **Specific prompts** -- "Fix JWT validation in src/auth/validate.ts:42" saves 10x tokens vs "fix auth bug"
- **Limit command output** -- run specific tests, not full suites. Pipe through `head` when exploring.

## The Core Insight

From Karpathy's 4 principles:

1. **Think Before Coding** -- state assumptions, present tradeoffs, ask clarifying questions BEFORE implementing
2. **Simplicity First** -- minimal code that solves the stated problem, nothing more
3. **Surgical Changes** -- only modify what's necessary, match existing style
4. **Goal-Driven Execution** -- convert vague tasks into verifiable objectives with success criteria

The unifying principle: **every loaded token should directly serve the current task.** If it loads every session but only applies sometimes, it's waste. Move it to on-demand loading.

## Before / After (Real Example)

| Metric | Before | After | Savings |
|--------|--------|-------|---------|
| Auto-loaded rules | 47.4 KB (~11,800 tokens) | 32.5 KB (~8,100 tokens) | 31% |
| Largest rules file | 13.8 KB (writing rules loading every session) | 3.7 KB (writing rules moved to /write command) | 73% |
| Total session overhead | ~23,000 tokens | ~18,900 tokens | 17% |
| Compaction trigger | 95% context (too late) | 50% context (preserves quality) | N/A |
| .claudeignore | Missing | Active | Prevents binary scanning |

These savings compound: fewer wasted tokens per turn means more room for actual work, fewer compaction cycles, and higher quality output in long sessions.

## Credits

Built by [Aragorn Meulendijks](https://github.com/Aragorn2046) and Claude Code.

Research compiled from:
- [Andrej Karpathy's X threads](https://x.com/karpathy) on Claude Code optimization
- [everything-claude-code](https://github.com/affaan-m/everything-claude-code) token optimization guide
- [andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills) Claude Code plugin
- [Claude Code Best Practices](https://docs.anthropic.com/en/docs/claude-code/best-practices) (official)
- [Claude Code Cost Management](https://docs.anthropic.com/en/docs/claude-code/costs) (official)

## License

MIT License. See [LICENSE](LICENSE) for details.

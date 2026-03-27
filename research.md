# Token Optimization Research

Compiled from Karpathy's posts, community guides, official Anthropic docs, and real-world optimization across 100+ Claude Code sessions.

## Karpathy's 4 Core Principles

From his X threads and the `andrej-karpathy-skills` Claude Code plugin:

1. **Think Before Coding** -- State assumptions explicitly. Present multiple interpretations. Advocate for simpler approaches. Ask clarifying questions BEFORE implementing.

2. **Simplicity First** -- Minimal code that solves the stated problem. No unrequested features, no unnecessary abstractions, no speculative flexibility. "If you write 200 lines and it could be 50, rewrite it."

3. **Surgical Changes** -- Only modify what's necessary. Don't improve adjacent code. Don't refactor unbroken functionality. Match existing style. Remove ONLY what your edits made unused.

4. **Goal-Driven Execution** -- Convert vague tasks into verifiable objectives with success criteria. "Fix the bug" becomes "write a test reproducing it, then make it pass." Multi-step work needs explicit verification checkpoints.

**Core principle:** Every changed line should directly trace to the user's request.

Karpathy went from 80% manual + 20% agents (Nov 2025) to 80% agent + 20% edits (early 2026). His key observation: models make wrong assumptions, don't surface inconsistencies, don't present tradeoffs, and love to overcomplicate code.

## The Auto-Load Tax

Claude Code loads certain files into context on EVERY turn:

| Source | Loads when | Impact |
|--------|-----------|--------|
| CLAUDE.md | Every turn | ~250 tokens per 1KB |
| `.claude/rules/*.md` (no `paths:` filter) | Every turn | Can be 5-50KB |
| `.claude/projects/*/memory/*.md` | Every turn | Grows unbounded |
| MCP server tool schemas | Every turn (unless deferred) | ~500 tokens per server |
| Hooks output | Every turn that triggers them | Variable |

Files in `.claude/commands/` and `.claude/skills/` load **only when invoked**. This distinction is the single most important thing to understand for optimization.

## Tier 1: CLAUDE.md Architecture (Highest Impact)

| Technique | Impact | Details |
|-----------|--------|---------|
| **Keep CLAUDE.md under 200 lines** | Prevents bloat | This file loads EVERY turn. Only include what can't be inferred from code. |
| **Tiered doc loading** | 83-90% token reduction | Essential instructions auto-load, everything else on-demand via commands |
| **Use commands for specialized rules** | ~15K tokens/session saved | Commands load on-demand when invoked, not every message |
| **.claudeignore** | Prevents context pollution | Exclude: build artifacts, lock files, generated code, binary files |

Real-world example: RedwoodJS project went from 11,000 to 800 tokens at startup (90% reduction) by moving most of their CLAUDE.md into tiered command files.

### What belongs in CLAUDE.md vs Commands

**CLAUDE.md (always loaded):**
- User identity and preferences that affect every interaction
- Non-obvious conventions that Claude would get wrong otherwise
- Routing tables (which file to read for which request type)
- Tool/path references that save lookup time every session

**Commands (loaded on demand):**
- Writing style rules (invoke via `/write`)
- Frontend design patterns (invoke via `/frontend`)
- Deployment procedures (invoke via `/deploy`)
- Testing conventions (invoke via `/test`)
- Domain-specific knowledge that only applies to certain tasks

### What to remove entirely
- Project structure descriptions (Claude reads the filesystem)
- Language/framework identification (Claude reads the code)
- Standard coding practices ("write clean code", "use meaningful names")
- Tool usage instructions that match Claude's defaults

## Tier 2: Session Management

| Technique | Impact | Details |
|-----------|--------|---------|
| **One session per logical task** | 30-50% reduction | One bug, one feature, one refactor. Don't mix. |
| **`/clear` between tasks** | Eliminates stale context | Switch topics = clear context |
| **`/compact` at breakpoints** | Compresses context | After exploration, after milestones, after debugging phases |
| **Don't compact mid-task** | Preserves quality | Never during active implementation or multi-file refactoring |
| **`CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50`** | Prevents quality degradation | Default is too late. Compact at 50% keeps quality high. |

### When to /compact vs /clear

- `/compact` -- you're staying on the same task but the context has accumulated exploration results, debug output, or false starts that are no longer relevant. Compaction preserves the thread but compresses the noise.
- `/clear` -- you're done with one task and starting another. Stale context from task A actively hurts task B because Claude tries to maintain consistency with irrelevant prior work.

## Tier 3: Model Selection

| Technique | Impact | Details |
|-----------|--------|---------|
| **Sonnet for routine tasks** | ~60% cost reduction | Handles 80% of coding tasks (file ops, simple edits, formatting) |
| **Opus for complex reasoning** | Quality where it matters | Architecture, debugging, multi-step logic, writing, security |
| **Haiku for subagent exploration** | ~80% cheaper | File reading, grep, simple searches |
| **Switch models mid-session** | Dynamic optimization | Use `/model` as task complexity changes |

## Tier 4: Context Management

| Technique | Impact | Details |
|-----------|--------|---------|
| **Subagents for exploration** | Keeps main context clean | Subagent reads 20 files, returns 1 summary |
| **Specific prompts** | Up to 10x reduction | "Fix JWT in src/auth/validate.ts:42" vs "fix auth bug" |
| **Limit command output** | Reduces context waste | "Run auth tests" not "run all tests". Pipe through `head`. |
| **MCP server review** | Each adds tool defs to context | Keep under 10 per project. Prefer CLIs (gh, aws) over MCP. |

### MCP Server Token Costs (Measured)

Each MCP server injects its tool schema into context. Measured costs:

| Server type | Approximate tokens |
|------------|-------------------|
| Simple (1-3 tools) | 300-500 |
| Medium (5-10 tools) | 800-2,000 |
| Complex (15+ tools, e.g. Playwright, Canva) | 2,500-3,500 |

With tool search deferral (default in recent Claude Code versions), only tool names load initially (~50 tokens per server). Full schemas load on first use. If your setup has 10+ servers, verify deferral is active.

## Tier 5: Advanced Techniques

| Technique | Tool/Method | Details |
|-----------|------------|---------|
| **Extended thinking control** | `MAX_THINKING_TOKENS` | Lower from default to 10,000 for routine tasks (70% hidden cost cut) |
| **Read-once patterns** | Hooks or custom tools | Block redundant file re-reads (60-90% Read tool savings) |
| **Monitor spending** | `/cost` | Track per-session consumption |
| **Autocompact override** | `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50` | Env var, set in shell profile |

## Rules File Optimization Patterns

### Pattern 1: Domain Split

Before (auto-loads every session):
```
~/.claude/rules/
  coding-rules.md     (3KB - universal)
  writing-rules.md    (8KB - only for writing)
  testing-rules.md    (2KB - only for tests)
  deploy-rules.md     (1KB - only for deployment)
  Total: 14KB every session
```

After (only universal rules auto-load):
```
~/.claude/rules/
  coding-rules.md     (3KB - universal, auto-loads)

~/.claude/commands/
  write.md            (reads writing-rules on invoke)
  test.md             (reads testing-rules on invoke)
  deploy.md           (reads deploy-rules on invoke)
  Total: 3KB every session, 11KB on-demand
```

### Pattern 2: Conditional Loading with paths:

Rules files can include `paths:` frontmatter to only load when working in matching directories:

```yaml
---
paths:
  - "src/frontend/**"
  - "*.tsx"
---
# Frontend Rules
These rules only load when you're working in frontend files.
```

This is useful for monorepos or projects with distinct subsystems.

### Pattern 3: Compression

Before (verbose):
```markdown
## Testing Rules
- When writing tests, always use the AAA pattern (Arrange, Act, Assert). This means
  you should first set up your test data and dependencies, then perform the action
  you're testing, and finally verify the results match expectations. This pattern
  makes tests readable and maintainable.
- Always mock external dependencies in unit tests. This includes API calls, database
  queries, file system operations, and any other I/O. Use the project's existing mock
  utilities where possible.
```

After (compressed):
```markdown
## Testing
- AAA pattern: Arrange, Act, Assert
- Mock all external I/O in unit tests. Use existing mock utilities.
```

Same information, ~70% fewer tokens.

## Implementation Checklist

Quick-start checklist for optimizing any Claude Code setup:

- [ ] Measure current CLAUDE.md size (target: <200 lines, <4KB)
- [ ] Measure total rules size (target: <20KB always-loaded)
- [ ] Move domain-specific rules to commands
- [ ] Compress verbose rules (remove explanations Claude doesn't need)
- [ ] Create .claudeignore if missing
- [ ] Set `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=50` in shell profile
- [ ] Audit memory files for stale/duplicate entries
- [ ] Review MCP servers (remove unused, verify tool deferral)
- [ ] Adopt one-task-per-session habit
- [ ] Use `/clear` between topics, `/compact` at milestones

## Sources

- [andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills) -- Karpathy's Claude Code plugin
- [everything-claude-code/token-optimization.md](https://github.com/affaan-m/everything-claude-code/blob/main/docs/token-optimization.md)
- [claude-token-optimizer](https://github.com/nadimtuhin/claude-token-optimizer) -- 90% reduction via tiered doc loading
- [7 Ways to Cut Token Usage](https://dev.to/boucle2026/7-ways-to-cut-your-claude-code-token-usage-elb)
- [Claude Code Best Practices](https://docs.anthropic.com/en/docs/claude-code/best-practices) (official)
- [Claude Code Cost Management](https://docs.anthropic.com/en/docs/claude-code/costs) (official)

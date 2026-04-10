# AGENTS.md

Project-specific instructions for AI assistants and contributors.

## Build & Test

```bash
cargo build            # Build
cargo test             # Run all tests
cargo clippy           # Lint
```

## Architecture Principles

### No Hardcoded Provider Quirks

**This is the single most important rule for this codebase.**

Different LLM providers have different API quirks (field names, message format
requirements, schema restrictions, etc.). We handle these differences through
the **`ProviderCompat` configuration layer**, not through hardcoded conditionals.

**Never do this:**

```rust
// WRONG: hardcoded provider detection
if self.base_url.contains("api.openai.com") {
    body["max_completion_tokens"] = json!(max_tokens);
} else {
    body["max_tokens"] = json!(max_tokens);
}

// WRONG: hardcoded model name check
if request.model.starts_with("deepseek") {
    msg["reasoning_content"] = json!("");
}

// WRONG: hardcoded vendor workaround
if is_kimi_model {
    body["temperature"] = json!(1.0);
}
```

**Always do this:**

```rust
// CORRECT: read from compat config
let field = self.compat.max_tokens_field.as_deref().unwrap_or("max_tokens");
body[field] = json!(request.max_tokens);

// CORRECT: configurable content filtering
if let Some(patterns) = &self.compat.strip_patterns {
    for p in patterns { text = text.replace(p, ""); }
}
```

**Why:** Hardcoded quirks accumulate fast and turn the codebase into an
unmaintainable "workaround warehouse". Provider behaviors change, new providers
appear, and model-name checks go stale. Configuration-driven compat keeps the
code clean and gives users control.

**How it works:**

1. Each provider type has **default compat presets** (see `ProviderCompat::openai_defaults()`, etc.)
2. Users override any setting via `[providers.xxx.compat]` or `[profiles.xxx.compat]` in config
3. Provider code reads `self.compat.*` fields — never inspects URLs or model names

If you need a new compat behavior:
- Add an `Option<T>` field to `ProviderCompat`
- Set its default in the appropriate preset function
- Use it in provider code via `self.compat.field_name`
- Document it in the config reference

### Provider Abstraction

All providers implement the `LlmProvider` trait. The engine never sees
provider-specific details. Keep it that way:

- `LlmRequest` / `LlmEvent` / `Message` / `ContentBlock` are provider-neutral
- Format conversion happens inside each provider's `build_messages()` / `build_request_body()`
- Shared logic (Anthropic/Bedrock/Vertex SSE parsing) lives in `anthropic_shared.rs`

### File Organization

This project is a **Cargo workspace** with 9 crates under `crates/`:

| Crate | Responsibility |
|-------|----------------|
| `aion-types` | Provider-neutral shared data types (`Message`, `LlmRequest`, `LlmEvent`, `Tool`, `ToolResult`) |
| `aion-protocol` | Host↔agent JSON stream protocol, events, commands, approval manager |
| `aion-config` | Configuration layer, `ProviderCompat`, auth (API key / OAuth), hooks, session config |
| `aion-providers` | LLM provider implementations (Anthropic, OpenAI, Bedrock, Vertex AI), retry logic |
| `aion-tools` | 7 built-in tools (Read, Write, Edit, Bash, Grep, Glob, Spawn) + `ToolRegistry` |
| `aion-mcp` | MCP client (stdio / SSE / streamable-http transports), tool proxy |
| `aion-skills` | Skill system: discovery, loading, execution, permissions, hooks, watcher |
| `aion-agent` | Agent engine (core loop), session manager, output sinks, spawner, VCR |
| `aion-cli` | CLI binary entry point |

Within each crate, one file per logical unit. Shared Anthropic/Bedrock/Vertex SSE
parsing lives in `aion-providers/src/anthropic_shared.rs`.

## Skills Module

`crates/aion-skills/` implements the Skill system — user-defined prompt snippets that
the agent can invoke by name.  The module is split into focused submodules:

| Submodule | Responsibility |
|-----------|----------------|
| `types` | Core data types: `SkillDefinition`, `SkillSource`, `SkillPermissions`, etc. |
| `frontmatter` | Parse YAML front matter from SKILL.md files |
| `loader` | Discover and load skills from the filesystem |
| `paths` | Platform skill directory resolution (`~/.config/aionrs/skills/`, `.aionrs/skills/`, legacy paths) |
| `discovery` | Runtime directory lookup keyed on the active working directory |
| `executor` | Execute a skill: variable substitution + optional shell command expansion |
| `substitution` | `$ARGUMENTS`, `$0`, `${AIONRS_SKILL_DIR}` replacement logic |
| `shell` | Shell command execution for `` !`cmd` `` syntax in skill bodies |
| `permissions` | Permission chain evaluation (deny → allow → safe-properties → ask) |
| `conditional` | Conditional activation: `paths:` glob matching |
| `context_modifier` | Apply skill-specified `model`/`effort`/`allowedTools` overrides |
| `bundled` | Built-in skills compiled into the binary (never truncated by budget) |
| `mcp` | Load skills from MCP servers; shell commands disabled for MCP skills |
| `hooks` | Parse and classify `PreToolUse`/`PostToolUse`/`Stop` hooks from skill front matter |
| `prompt` | Render the skill list for injection into the system prompt; budget control |
| `watcher` | Watch skill directories for file changes; debounced version counter |

### Development conventions

**Adding a new front matter field**

1. Add the field to the appropriate struct in `aion-skills/src/types.rs`
2. Parse it in `aion-skills/src/frontmatter.rs` (`parse_frontmatter`)
3. Add a unit test in `aion-skills/src/frontmatter.rs` inline tests

**Adding a new built-in (bundled) skill**

1. Create a `SKILL.md` file under `aion-skills/src/bundled/`
2. Register it in `aion-skills/src/bundled.rs` — the `BUNDLED_SKILLS` static slice
3. Bundled skills are never truncated by prompt budget; use sparingly

**Extending the permission system**

- Permission priority is fixed: deny > allow > safe-properties > ask
- Never reorder; tests in `aion-skills/src/permissions.rs` and
  `aion-skills/src/permissions_supplemental_tests.rs` encode the expected chain

**Filesystem watcher**

- `SkillWatcher` uses `notify` (cross-platform) with a 300 ms debounce
- `should_ignore` filters spurious events; update it (with a comment) when
  adding new filter rules — do not add `#[cfg(target_os)]` conditionals

### Test organization

| Location | What goes there |
|----------|----------------|
| Inline `#[cfg(test)]` in each `.rs` file | White-box unit tests for that module's internals |
| `crates/<crate>/tests/` | Integration tests for that crate |
| `aion-skills/src/permissions_supplemental_tests.rs` | Additional permission chain edge cases |
| `aion-skills/src/bundled_supplemental_tests.rs` | Bundled skill edge cases |
| `aion-skills/src/integration_tests.rs` | Cross-module end-to-end tests |
| `aion-agent/tests/e2e/` | Agent E2E tests (anthropic, openai) |
| `aion-protocol/tests/` | Protocol approval and command tests |
| `aion-providers/tests/` | Provider-specific tests |

## Code Style

- Rust 2021 edition, stable toolchain
- `cargo clippy` must pass without warnings
- Comments in English, commit messages in English
- Keep files under 800 lines; extract modules when approaching the limit

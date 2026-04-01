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

- `src/provider/` — One file per provider + `compat.rs` + `anthropic_shared.rs`
- `src/tools/` — One file per tool
- `src/types/` — Shared data types (provider-neutral)
- `src/mcp/` — MCP client implementation
- `src/protocol/` — JSON stream protocol for host integration

## Code Style

- Rust 2021 edition, stable toolchain
- `cargo clippy` must pass without warnings
- Comments in English, commit messages in English
- Keep files under 800 lines; extract modules when approaching the limit

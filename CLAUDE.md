# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`ado-aw` is a Rust compiler that transforms natural language markdown files (with YAML front matter) into Azure DevOps pipeline YAML definitions. Inspired by [GitHub Agentic Workflows (gh-aw)](https://github.com/githubnext/gh-aw). The compiled pipelines run AI agents in network-isolated sandboxes with a 3-job architecture: PerformAgenticTask -> AnalyzeSafeOutputs -> ProcessSafeOutputs.

## Build & Test

```bash
cargo build                    # Build the compiler
cargo build --release          # Release build
cargo test                     # Run all tests (unit + integration)
cargo test <test_name>         # Run a single test by name
cargo test --test compiler_tests  # Run only compiler integration tests
cargo test --test mcp_firewall_tests
cargo test --test proxy_tests
cargo clippy                   # Lint
cargo fmt                      # Format
```

Compile a markdown pipeline:
```bash
cargo run -- compile ./path/to/agent.md           # Single file
cargo run -- compile                               # Auto-discover and recompile all
cargo run -- check ./path/to/pipeline.yml          # Verify compiled matches source
```

## Architecture

**Compilation flow**: Markdown (YAML front matter + body) -> Parser (`compile/common.rs`) -> Target Compiler (`standalone.rs` or `onees.rs`) -> Azure DevOps pipeline YAML + agent file in `agents/`

### Key Modules

- `src/main.rs` — CLI entry point (clap derive). Subcommands: `create`, `compile`, `check`, `mcp`, `execute`, `proxy`, `mcp-firewall`, `configure`
- `src/compile/` — Core compilation engine
  - `types.rs` — `FrontMatter` struct (the YAML front matter grammar), `CompileTarget` enum
  - `common.rs` — Shared parsing (`parse_markdown`), `AWF_VERSION`/`COPILOT_CLI_VERSION` constants, template marker replacement helpers
  - `standalone.rs` — Standalone target: full 3-job pipeline with AWF network sandbox, MCP firewall, safe outputs
  - `onees.rs` — 1ES Pipeline Template target: extends `1ES.Unofficial.PipelineTemplate.yml`
  - `mod.rs` — `Compiler` trait definition
- `src/tools/` — MCP safe-output tool implementations (create_pr, create_work_item, comment_on_work_item, create/update_wiki_page, update_work_item, memory, noop, missing_data, missing_tool)
- `src/execute.rs` — Stage 2 executor: processes safe-output NDJSON, performs actual ADO writes
- `src/mcp.rs` — SafeOutputs MCP server (exposes tools to agents)
- `src/mcp_firewall.rs` — MCP Firewall: proxies/filters tool calls to upstream MCP servers with allow-list enforcement
- `src/proxy.rs` — HTTP proxy for network filtering
- `src/allowed_hosts.rs` — Core domain allowlist for AWF network isolation
- `src/fuzzy_schedule.rs` — Fuzzy schedule syntax parser (e.g., "daily around 14:00" -> cron)
- `src/configure.rs` — CLI command to detect and update `GITHUB_TOKEN` pipeline variables
- `src/create.rs` — Interactive agent creation wizard

### Templates

- `templates/base.yml` — Standalone pipeline template with `{{ marker }}` placeholders
- `templates/1es-base.yml` — 1ES pipeline template
- `templates/threat-analysis.md` — Threat detection prompt for AnalyzeSafeOutputs job

The compiler replaces `{{ marker }}` placeholders (e.g., `{{ copilot_params }}`, `{{ allowed_domains }}`, `{{ firewall_config }}`). Do NOT touch `${{ }}` markers — those are Azure DevOps runtime expressions.

### Tests

- `tests/compiler_tests.rs` — Integration tests using fixture files from `tests/fixtures/`
- `tests/mcp_firewall_tests.rs` — MCP firewall tests
- `tests/proxy_tests.rs` — Proxy tests
- `tests/fixtures/` — Sample agent markdown files (minimal, complete, 1ES, pipeline-trigger, comment-on-work-item)

## Conventions

- **Commit messages**: [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `chore:`, etc.) — required for `release-please` automated releases
- **Error handling**: `anyhow::Result` everywhere, `anyhow::bail!` / `.context()` for errors
- **Rust edition**: 2024
- **MCP**: Uses `rmcp` crate with server + transport-io features

## Extending the Compiler

- **New CLI commands**: Add variants to the `Commands` enum in `main.rs`
- **New compile targets**: Implement the `Compiler` trait in a new file under `src/compile/`
- **New front matter fields**: Add to `FrontMatter` in `src/compile/types.rs`
- **New template markers**: Handle replacements in `standalone.rs` or `onees.rs`
- **New safe-output tools**: Add a new file in `src/tools/`, register in `src/tools/mod.rs` and `src/mcp.rs`

## Key References

- `AGENTS.md` — Authoritative reference for front matter schema, all `{{ marker }}` definitions, safe-output tool specs, MCP firewall config, and network isolation details. Read this when modifying compilation or adding features.
- `prompts/create-ado-agentic-workflow.md` — Step-by-step guide for AI agents authoring workflow files.

## CI

PR CI (`rust-tests.yml`) runs `cargo build` + `cargo test` on changes to `src/`, `tests/`, `templates/`, `Cargo.*`. Releases use `release-please` triggered on pushes to `main`.

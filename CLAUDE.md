# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

shield0r is a runtime LLM guardrail proxy written in Rust that acts as a reverse proxy for OpenAI/Azure API endpoints. It inspects requests and responses to enforce security policies, with policies derived from real attack findings via its sibling tool `hamm0r`.

Core architecture:
- **Reverse proxy**: Built with `axum` and `tower` frameworks
- **API compatibility**: Speaks OpenAI/Azure API shape
- **Deployment**: Single static Rust binary, deployable as sidecar or edge service
- **Detection**: Hybrid approach - Rust for enforcement, existing ML models for detection

## Development Commands

When the project is implemented, use these commands:

```bash
# Build
cargo build
cargo build --release

# Test
cargo test
cargo test -- --nocapture  # for debugging with println!

# Run
cargo run
cargo run --release

# Lint and format
cargo fmt
cargo clippy -- -D warnings

# Check before commit
cargo fmt --check
cargo clippy -- -D warnings
cargo test
```

## Architecture Patterns

### 1. Request Flow
```
client -> [shield0r proxy] -> LLM endpoint
             |                    |
       inspect request      inspect response
             |                    |
    allow/block/redact/flag (all logged)
```

### 2. Detection Layers (implement in order of efficiency)
1. **Deterministic** (regex/denylist) - cheap, decides most traffic
2. **Classifier** (local/remote model) - semantic intent detection
3. **LLM judge** (optional) - only for ambiguous cases

### 3. Key Integration: hamm0r verdict import
The core innovation is the assurance loop with hamm0r:
- Import `run-NNN.verdicts.jsonl` files from hamm0r engagements
- Compile verdicts into shield0r policies
- Policies use OWASP LLM Top 10 taxonomy for categorization

### 4. Rule Pack Format
Detection rules should be versioned data, not code:
- Updatable without recompiling
- Community contributable
- Testable via hamm0r replay

## Code Conventions

### Rust Patterns
- Use `axum` for HTTP server and routing
- Use `tower` for middleware composition
- Prefer `anyhow` for application errors, `thiserror` for library errors
- Use `serde` for JSON parsing (verdicts, API payloads)
- Use `tracing` for structured logging

### Error Handling
- No `unwrap()` in production code paths
- Use `?` operator for error propagation
- Provide context with `.context()` from anyhow

### Testing
- Unit tests in same file as implementation
- Integration tests in `tests/` directory
- Test the verdict -> policy compilation thoroughly
- Mock external classifier calls in tests

## Implementation Priority

Follow the build order from productvision.md:
1. Design verdict-stream -> policy contract (keystone)
2. Basic proxy forwarding OpenAI/Azure calls
3. Deterministic detector + rule format
4. Classifier integration wrapper
5. Verdict import and policy compiler
6. LLM judge escalation (optional)

## Key Design Constraints

- **No Python dependencies** - pure Rust deployment
- **Transparent proxy** - clients only change base URL
- **OWASP taxonomy** - use same categories as hamm0r
- **Fail-safe configurable** - grey-zone behavior on timeout

## External Dependencies

When implementing, consider these crates:
- `axum` + `tower` for HTTP proxy
- `reqwest` for upstream API calls
- `serde` + `serde_json` for data handling
- `tokio` for async runtime
- `tracing` + `tracing-subscriber` for logging
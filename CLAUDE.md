# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Common Development Commands

### Building and Testing
- `cargo build` - Build the project
- `cargo build --release` - Build optimized release version
- `cargo test` - Run tests (if any exist)
- `cargo check` - Quick syntax/type check without building
- `cargo clippy` - Run linter (enforced with `-D warnings` in CI)
- `cargo fmt` - Format code (uses max_width = 140 from rustfmt.toml)

### Running the Application
- `cargo run` - Run the DNS proxy server with default settings
- `cargo run -- -h` - Show help with all command line options
- `dns2socks -l 0.0.0.0:5353 -d 8.8.8.8:53 -s socks5://127.0.0.1:1080` - Example custom configuration

### Cross-Platform Building
- `./build-apple.sh` - Build for all Apple platforms (iOS, macOS) and create XCFramework
- `./build-aarch64-apple-ios.sh` - Build specifically for iOS ARM64
- `cargo ndk -t x86_64 rustc --lib --crate-type=cdylib` - Build for Android (requires cargo-ndk)

## Architecture Overview

dns2socks is a DNS proxy server that forwards DNS requests through SOCKS5 proxies, built in Rust using async/await with Tokio.

### Core Components

**Main Entry Points:**
- `src/bin/dns2socks.rs` - CLI application entry point
- `src/api.rs` - C FFI interface for library usage (`dns2socks_start`/`dns2socks_stop`)
- `src/android.rs` - Android JNI bindings

**Core Engine (`src/lib.rs`):**
- Dual protocol support: runs concurrent UDP and TCP listeners
- `udp_thread()` - Handles UDP DNS queries (default mode)
- `tcp_thread()` - Handles TCP DNS queries
- `main_entry()` - Orchestrates both listeners with graceful shutdown

**Key Modules:**
- `src/config.rs` - Configuration parsing and validation (CLI args, SOCKS5 URLs)
- `src/dns.rs` - DNS message parsing and IP address extraction
- `src/dump_logger.rs` - Custom logging interface for C FFI callbacks

### Data Flow
1. DNS client sends query to dns2socks listener (port 53 by default)
2. Query is parsed and domain extracted for logging
3. Optional cache lookup for repeated queries
4. Query is forwarded through SOCKS5 proxy to upstream DNS server (8.8.8.8:53 by default)
5. Response is cached (if enabled) and returned to client

### Library Interface
The crate can be used as both a CLI tool and a library:
- **CLI**: `cargo install dns2socks` or use precompiled binaries
- **Library**: Provides C-compatible FFI for embedding in mobile apps
- **Cross-platform**: Supports macOS, iOS, Android, Linux, Windows

### Configuration Options
- Listen address (`-l`) - Where to bind the DNS server
- Remote DNS server (`-d`) - Upstream DNS server to query
- SOCKS5 settings (`-s`) - Proxy URL with optional authentication
- Cluster DNS (`-k`) - Direct DNS server for `*.svc.cluster.local` queries (bypasses SOCKS5)
- Force TCP (`-f`) - Route all queries via TCP instead of UDP
- Cache records (`-c`) - Enable DNS response caching (30min TTL, 5min idle)
- Verbosity (`-v`) - Logging level from off to trace
- Timeout (`-t`) - DNS query timeout in seconds

## Secrets and Credentials

- **Never** commit secrets, API keys, or credentials to the repository
- Keep `.env` files in `.gitignore`

## Documentation

**Always update relevant documentation with code changes.** This includes:

- **README.md**: Keep in sync with CLI options, features, and usage examples
- **CLAUDE.md**: Update when changing architecture, configuration options, or dev commands

Documentation updates should be part of the same commit as the code change they document.

## Git

### Commits

**Never commit without being explicitly asked.**

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`, `ci`, `perf`, `build`

### Branch Naming

Use the format: `<type>/<short-description>`

- Type mirrors Conventional Commits: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `ci`, `perf`, `build`, `hotfix`
- Description is kebab-case, concise (2–5 words)

**Examples:** `feat/cluster-dns-option`, `fix/tcp-framing`, `docs/update-readme`

### Pull Requests

- PRs merge to `master`
- Keep PRs focused: one logical change per PR

## Platform-Specific Notes

### Mobile Integration
- Uses `cbindgen.toml` to generate C headers for iOS/Android integration
- Android support via JNI bindings in `android.rs`
- iOS support via C FFI and XCFramework generation

### CI/CD Pipeline
The project uses GitHub Actions with:
- Multi-platform testing (Ubuntu, macOS, Windows)
- Android cross-compilation testing
- Code formatting and linting enforcement
- Automated dependency updates via Dependabot
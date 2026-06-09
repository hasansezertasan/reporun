# reporun

**reporun** is a GitHub-to-executable launcher that contains _no code_.

This is a **specification-only library** inspired by [whenwords](https://github.com/dbreunig/whenwords). It fully documents what reporun does, how it should work, and what it produces—but contains zero implementation.

## What is reporun?

reporun creates standalone executables that run Python projects directly from GitHub. The tool operates in two phases:

**Build Phase:**

```bash
reporun https://github.com/user/project
# Reads reporun.toml from the repository
# Creates a standalone runner binary
```

**Runtime Phase:**

```bash
./runner
# Installs uv (if needed)
# Clones/updates the repository
# Installs dependencies
# Executes the configured entrypoint
```

## Key Features

- **Always fresh** - Pulls latest code from GitHub on every run
- **Zero Python required** - Downloads and manages `uv` automatically
- **Single binary** - One executable per project
- **Configuration as code** - Project defines behavior via `reporun.toml`
- **Cross-platform** - Works on Windows, macOS, and Linux
- **User data storage** - Repos stored in standard OS cache directories

## Example Configuration

Projects declare their runtime behavior in `reporun.toml`:

```toml
entrypoint = "python -m myproject"
```

That's it! Python version and dependencies are managed by uv through standard files (`.python-version`, `pyproject.toml`).

The repository is always updated to the latest version on every run.

## Documentation

- **[SPEC.md](SPEC.md)** - Complete technical specification
- **[INSTALL.md](INSTALL.md)** - AI prompt for implementation
- **[tests.yaml](tests.yaml)** - Language-agnostic test scenarios

## Why Spec-Only?

Following the whenwords pattern:

- ✅ **Implement in any language** - Rust, Go, Python, whatever fits your needs
- ✅ **AI-ready** - LLMs can read INSTALL.md and generate working code
- ✅ **No maintenance burden** - Specification doesn't bitrot like code
- ✅ **Clear contract** - All implementations conform to the same behavior

## How to Implement

1. Read [SPEC.md](SPEC.md) for complete technical details
2. Use [INSTALL.md](INSTALL.md) as an AI prompt (or implement manually)
3. Validate against [tests.yaml](tests.yaml)

## Similar Tools (and Differences)

**vs pyapp:**

- reporun fetches fresh code on every run (pyapp bundles at build time)
- reporun is simpler (no embedded Python interpreter)

**vs packaged:**

- reporun updates automatically (packaged is static)
- reporun doesn't require Python pre-installed

**vs pycrucible:**

- reporun is a spec, not an implementation
- reporun targets end-users, not just developers

## License

MIT License - See [LICENSE](LICENSE)

## Contributing

Found ambiguity in the spec? Open an issue or PR. This is a living specification that improves through use.

# Implementation Prompt for AI Assistants

Use this prompt with any AI assistant to generate a working implementation of reporun.

---

## Prompt

Implement **reporun** in **[LANGUAGE]**.

reporun is a GitHub-to-executable launcher tool that creates standalone binaries for running Python projects directly from GitHub repositories.

### Requirements

1. Read the complete specification in `SPEC.md`
2. Implement both phases:
   - **Build phase**: Create runner binaries from GitHub URLs
   - **Runtime phase**: Execute projects with automatic updates
3. Support all configuration options in `reporun.toml`
4. Pass all test scenarios in `tests.yaml`

### Implementation Location

The implementation should be located at: **[LOCATION]**

For example:
- Rust: `./rust-impl/`
- Go: `./go-impl/`
- Python: `./python-impl/`
- TypeScript: `./ts-impl/`

### Key Behaviors

**Build Phase:**
```
Input: reporun https://github.com/user/project
Process:
  1. Clone repository (to validate it exists)
  2. Validate reporun.toml exists
  3. Create standalone runner binary (embeds only repo URL)
Output: ./project (executable)
```

**Runtime Phase:**
```
Input: ./project (user runs the binary)
Process:
  1. Ensure uv is installed
  2. Clone or update repository to cache directory
  3. Read reporun.toml from cloned repository
  4. Install dependencies with uv
  5. Execute configured entrypoint
Output: Exit with entrypoint's exit code
```

### Configuration Format

Projects define their behavior in `reporun.toml`:

```toml
entrypoint = "python -m myproject"
```

Python version and dependencies are managed by uv through standard files (`.python-version`, `pyproject.toml`).

### Storage Locations

Always use OS-specific cache directories (XDG Base Directory compliant):
- **Linux**: `$XDG_CACHE_HOME/reporun/<repo-name>` or `~/.cache/reporun/<repo-name>`
- **macOS**: `~/Library/Caches/reporun/<repo-name>`
- **Windows**: `%LOCALAPPDATA%\reporun\<repo-name>`

### Critical Features

1. **Update Strategies**
   - `always`: Pull on every run
   - `daily`: Pull once per day maximum
   - `weekly`: Pull once per week maximum
   - `never`: Clone once, never update

2. **Dependency Management**
   - Use `uv sync` for primary dependency installation
   - Support custom dependency commands via config

3. **Error Handling**
   - Proper exit codes for each failure scenario (see SPEC.md)
   - Clear error messages
   - Graceful offline behavior

4. **Cross-Platform Support**
   - Windows, macOS, and Linux
   - Appropriate binary formats for each platform

### Testing

Validate your implementation against `tests.yaml`:

```bash
# Example test runner (pseudo-code)
for test in tests.yaml:
    run_test(test)
    assert exit_code == expected_exit_code
    assert output contains expected_messages
```

### Output Structure

Your implementation should produce:

```
[LOCATION]/
├── src/           # Source code
├── tests/         # Implementation-specific tests
├── README.md      # Implementation-specific docs
├── Cargo.toml     # Or equivalent (package.json, pyproject.toml, etc.)
└── ...
```

### Validation Checklist

Before considering the implementation complete:

- [ ] Build phase creates executable from GitHub URL
- [ ] Build phase validates reporun.toml exists
- [ ] Binary embeds only repo URL (not config)
- [ ] Runtime installs uv if missing
- [ ] Runtime clones repo on first run
- [ ] Runtime reads reporun.toml from cloned repo
- [ ] Runtime updates repo on every run
- [ ] Runtime installs dependencies with uv (from pyproject.toml)
- [ ] Runtime executes entrypoint correctly
- [ ] Python version handling via .python-version works
- [ ] Exit codes match specification
- [ ] Cross-platform binaries work
- [ ] All tests in tests.yaml pass

### Example Usage

After implementation, the tool should work like this:

```bash
# Build phase
$ reporun https://github.com/psf/requests-sample
Building runner for requests-sample...
✓ Repository cloned
✓ Configuration validated
✓ Runner created: ./requests-sample

# Runtime phase
$ ./requests-sample
[reporun] Installing uv...
[reporun] Cloning repository...
[reporun] Installing dependencies...
[reporun] Executing: python -m requests_sample
Hello from requests-sample!

$ echo $?
0
```

### Reference Implementation Structure

While there is no "official" implementation, here's a recommended structure:

**Core Modules:**
1. `cli` - Command-line argument parsing
2. `builder` - Build phase logic (clone, validate, create binary)
3. `runtime` - Runtime phase logic (uv, git, execution)
4. `config` - Configuration parsing and validation
5. `storage` - OS-specific directory handling
6. `git` - Git operations wrapper
7. `uv` - UV installation and management
8. `errors` - Error types and exit codes

**Entry Points:**
- `main` - CLI entry for build phase
- `runner` - Embedded entry for runtime phase

### Notes

- The runner binary should be **self-contained** - it embeds only the repo URL (config is read from the cloned repo at runtime)
- Don't bundle the repository itself - always fetch from GitHub at runtime
- Leverage existing tools (`git`, `uv`) rather than reimplementing
- Keep it simple - this is a tool, not a framework

### Getting Started

1. **Setup project**: Create directory structure and package config
2. **Implement config parsing**: Start with `reporun.toml` validation
3. **Implement build phase**: Clone repo, validate, create binary stub
4. **Implement runtime phase**: Start with simple execution, add features incrementally
5. **Add storage logic**: OS-specific cache directory resolution
6. **Add uv management**: Installation and execution
7. **Test thoroughly**: Run against tests.yaml scenarios
8. **Polish**: Error messages, logging, cross-platform testing

---

## Example: Using this prompt with Claude

```
Hey Claude, I want to implement reporun in Rust.

[Paste this entire INSTALL.md file]

Please implement it at ./rust-impl/ with a focus on performance and small binary size.
```

## Example: Using this prompt with ChatGPT

```
I need you to implement reporun in Go.

[Paste this entire INSTALL.md file]

The implementation should be in ./go-impl/ and use standard library where possible.
```

## Example: Using this prompt with Copilot

```
@workspace Implement reporun in Python

[Paste this entire INSTALL.md file]

Create the implementation in ./python-impl/ using modern Python (3.12+) with type hints.
```

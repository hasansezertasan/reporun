# reporun Technical Specification

Version: 1.0.0
Status: Draft

## Overview

reporun creates standalone executable launchers for Python projects hosted on GitHub. Each launcher manages its own repository clone, dependencies, and execution environment.

## Architecture

reporun operates in two distinct phases:

1. **Build Phase** - Creates a standalone runner binary
2. **Runtime Phase** - Executes the project with fresh code

```
┌─────────────────────────────────────────────────────────────┐
│ BUILD PHASE                                                 │
│                                                             │
│  Input: GitHub URL                                          │
│     ↓                                                       │
│  Clone repository                                           │
│     ↓                                                       │
│  Read reporun.toml                                          │
│     ↓                                                       │
│  Create runner binary (embeds: repo URL, config)            │
│     ↓                                                       │
│  Output: ./runner executable                                │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ RUNTIME PHASE (when user runs ./runner)                     │
│                                                             │
│  Check for uv → Install if missing                          │
│     ↓                                                       │
│  Check for repo clone → Clone or pull latest                │
│     ↓                                                       │
│  Install dependencies (uv sync)                             │
│     ↓                                                       │
│  Execute entrypoint                                         │
└─────────────────────────────────────────────────────────────┘
```

## Build Phase Specification

### Input

The build phase accepts a GitHub repository URL:

```bash
reporun <repository-url>
```

**Valid URL formats:**

- `https://github.com/user/repo`
- `https://github.com/user/repo.git`

### Process

1. **Clone Repository**
   - Use git to clone the repository to a temporary location
   - Clone with `--depth 1` for efficiency
   - Use the repository's default branch

2. **Locate Configuration**
   - Look for `reporun.toml` in repository root
   - **Error if missing**: Exit with message "No reporun.toml found in repository"

3. **Validate Configuration**
   - Parse TOML file
   - Validate required fields (see Configuration Schema)
   - **Error if invalid**: Exit with specific validation message

4. **Create Runner Binary**
   - Generate executable for target platform
   - Embed only the repository URL within the binary
   - Make binary executable (`chmod +x` on Unix-like systems)
   - Binary name defaults to repository name
   - **Note:** Configuration (reporun.toml) is NOT embedded; it's read from the cloned repository at runtime

5. **Output**
   - Write binary to current directory
   - Print success message with binary name

### Error Conditions

| Condition               | Exit Code | Message                                           |
| ----------------------- | --------- | ------------------------------------------------- |
| Invalid URL             | 1         | "Invalid repository URL: <url>"                   |
| Network failure         | 2         | "Failed to clone repository: <error>"             |
| No reporun.toml         | 3         | "No reporun.toml found in repository root"        |
| Invalid TOML            | 4         | "Invalid reporun.toml: <parse error>"             |
| Missing required config | 5         | "Missing required field in reporun.toml: <field>" |
| Binary creation failed  | 6         | "Failed to create runner binary: <error>"         |

## Runtime Phase Specification

### Initialization

When the runner binary is executed:

1. **Determine Storage Location**
   - Use OS-specific cache directory following XDG Base Directory Specification:
     - **Linux**: `$XDG_CACHE_HOME/reporun/<repo-name>` or `~/.cache/reporun/<repo-name>`
     - **macOS**: `~/Library/Caches/reporun/<repo-name>`
     - **Windows**: `%LOCALAPPDATA%\reporun\<repo-name>`

2. **Ensure uv is Available**
   - Check if `uv` exists in PATH
   - If missing:
     - Download from https://astral.sh/uv/install.sh (Unix) or https://astral.sh/uv/install.ps1 (Windows)
     - Install to a reporun-managed location within storage directory
     - Use this uv installation for all subsequent operations

### Repository Management

3. **Clone or Update Repository**

   **First Run (no clone exists):**

   ```bash
   git clone --depth 1 <repo-url> <storage-location>
   ```

   **Subsequent Runs:**
   - Always update to latest version:
     ```bash
     cd <storage-location>
     git fetch origin
     git reset --hard origin/HEAD
     ```

4. **Read Configuration**
   - Parse `<storage-location>/reporun.toml`
   - Validate required fields
   - Use configuration for subsequent steps

### Dependency Management

5. **Install Python and Dependencies**

   Execute in the repository directory:

   ```bash
   uv sync
   ```

   **Python Version Management:**
   - uv automatically handles Python version via:
     - `.python-version` file in repository
     - `requires-python` in `pyproject.toml`
   - No additional configuration needed in `reporun.toml`

   **Dependency Management:**
   - uv installs dependencies from `pyproject.toml`
   - Respects lock files (`uv.lock`) if present
   - No additional configuration needed in `reporun.toml`

### Execution

6. **Run Entrypoint**

   Execute the configured entrypoint command:

   ```bash
   cd <storage-location>
   <entrypoint command>
   ```

   **Process Management:**
   - Inherit parent process environment
   - Run entrypoint as subprocess
   - Forward stdout/stderr to parent
   - Forward signals (SIGINT, SIGTERM)
   - Exit with same code as entrypoint

### Error Conditions

| Condition                 | Exit Code | Message                                   |
| ------------------------- | --------- | ----------------------------------------- |
| uv installation failed    | 10        | "Failed to install uv: <error>"           |
| Clone failed              | 11        | "Failed to clone repository: <error>"     |
| Update failed             | 12        | "Failed to update repository: <error>"    |
| Dependency install failed | 13        | "Failed to install dependencies: <error>" |
| Entrypoint not found      | 15        | "Entrypoint command not found: <command>" |
| Entrypoint failed         | \*        | Exit with entrypoint's exit code          |

## Configuration Schema

### reporun.toml Format

```toml
# REQUIRED: Command to execute
entrypoint = "python -m myproject"
```

**Note:**
- Python version and dependencies are managed by uv through standard Python files:
  - Python version: `.python-version` or `requires-python` in `pyproject.toml`
  - Dependencies: `pyproject.toml` and optional `uv.lock`
- Repository is always updated to latest on every run

### Field Specifications

**`entrypoint`** (required, string)

- Command to execute the project
- Executed as shell command from repository root
- Examples:
  - `"python -m myproject"`
  - `"uv run main.py"`
  - `"python cli.py --serve"`

## Cross-Platform Considerations

### Path Handling

- Always use forward slashes (`/`) in configuration
- Runtime must convert to OS-appropriate separators
- Support Windows UNC paths (`\\server\share`)

### Executable Creation

- **Unix-like systems**: Single binary file with shebang and execute permission
- **Windows**: `.exe` file with embedded manifest

### Shell Command Execution

- **Unix-like**: Use `/bin/sh -c` for entrypoint execution
- **Windows**: Use `cmd.exe /C` or PowerShell depending on platform

## Security Considerations

### Sandboxing

**Out of scope for v1.0:**

- No process sandboxing
- No network isolation
- Full filesystem access

**User responsibility:**

- Only run trusted repositories
- Review reporun.toml before execution

### Supply Chain

**Verification steps:**

- Validate HTTPS certificates when cloning
- Download uv from official astral.sh domain only
- Verify git repository signatures if present (optional)

### Secrets Management

**NOT handled by reporun:**

- No secrets injection
- No environment variable templating
- Projects must handle their own secrets (e.g., .env files)

## Offline Behavior

reporun **requires network access** for:

- Initial repository clone
- Repository updates
- uv installation (first run)
- Dependency downloads (if not cached by uv)

**Graceful degradation:**

- If network unavailable and repo already cloned:
  - Skip update (log warning)
  - Continue with cached version
- If network unavailable and no repo clone:
  - Exit with error code 11

## Logging and Output

### Log Format

Always show major steps (cloning, installing, executing) to keep users informed.

```
[reporun] <timestamp> <level>: <message>
```

Example:

```
[reporun] 2025-01-15T10:30:45Z INFO: Cloning repository...
[reporun] 2025-01-15T10:30:47Z INFO: Installing dependencies...
[reporun] 2025-01-15T10:30:52Z INFO: Executing entrypoint...
```

### Progress Indicators

- Show progress for long operations:
  - Git clone progress
  - uv installation progress
  - Dependency installation progress

## Performance Optimization

### Git Operations

- Use `--depth 1` for clones (shallow clone)
- Use `--single-branch` on the repository's default branch
- Use `git fetch --prune` for updates

### Caching

- Leverage uv's built-in cache for Python packages
- Don't duplicate caching logic

### Parallel Execution

**Out of scope for v1.0:**

- No parallel dependency installation
- No background updates

## Future Considerations

Features explicitly deferred to future versions:

- **Version pinning**: Lock to specific commit hash
- **Verbosity levels**: --quiet, --debug flags (currently always shows major steps)
- **Environment variables**: REPORUN_STORAGE, REPORUN_VERSION (scripts can use pwd and git)
- **Offline mode**: Bundle dependencies for offline use
- **Configuration overrides**: Runtime flags to override reporun.toml
- **Multiple entrypoints**: Support subcommands
- **Pre/post hooks**: Run scripts before/after main entrypoint
- **Automatic updates**: Self-updating runner binaries
- **Telemetry**: Usage statistics and error reporting

## Implementation Checklist

A complete implementation must:

- [ ] Parse all supported CLI flags
- [ ] Clone git repositories with proper error handling
- [ ] Parse and validate reporun.toml
- [ ] Determine OS-specific cache directories
- [ ] Install uv if not present
- [ ] Update repository on every run
- [ ] Install dependencies with `uv sync` (from `pyproject.toml`)
- [ ] Honor Python version from `.python-version` / `requires-python`
- [ ] Execute entrypoint with proper environment
- [ ] Forward signals to subprocess
- [ ] Exit with correct exit codes
- [ ] Generate cross-platform binaries
- [ ] Store repositories in OS-specific cache directories
- [ ] Log major steps (clone, install, execute)
- [ ] Handle offline scenarios gracefully
- [ ] Pass all tests in tests.yaml

## Versioning

This specification follows Semantic Versioning (SemVer):

- **Major version**: Breaking changes to configuration format or CLI interface
- **Minor version**: Backwards-compatible new features
- **Patch version**: Clarifications and corrections to spec

Current version: **1.0.0**

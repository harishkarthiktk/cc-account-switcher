# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Bash-based multi-account switcher for Claude Code that manages authentication credentials across multiple accounts. The tool allows users to seamlessly switch between different Claude Code accounts without losing their settings, themes, or preferences.

## Architecture

### Core Functionality

The script (`ccswitch.sh`) manages Claude Code accounts by storing and swapping authentication data:

**Storage Strategy:**
- **macOS**: Credentials stored in system Keychain, OAuth configs in `~/.claude-switch-backup/configs/`
- **Linux**: Credentials stored in libsecret (system keyring), OAuth configs in `~/.claude-switch-backup/configs/`
- **WSL**: Credentials encrypted with Windows DPAPI in `%USERPROFILE%\.claude-switch\`, OAuth configs in `~/.claude-switch-backup/configs/`
- All account metadata tracked in `~/.claude-switch-backup/sequence.json`

**Key Components:**
- **Platform detection** (lines 37-50): Detects macOS, Linux, WSL, and container environments
- **Credential management** (lines 249-461): Platform-specific secure read/write operations for credentials
  - macOS: Uses system Keychain via `security` command
  - Linux: Uses libsecret via `secret-tool` command
  - WSL: Uses Windows DPAPI via PowerShell encryption
- **Config management** (lines 488-510): Handles OAuth account data from `~/.claude/.claude.json`
- **Credential deletion** (lines 464-486): Platform-specific credential cleanup functions
- **Account switching** (lines 731-808): Backup current account, restore target account, update active state
- **Sequence tracking** (lines 512-560): Maintains ordered list of accounts for rotation

### Critical File Paths

The script works with these Claude Code files:
- Primary config: `~/.claude/.claude.json` (preferred, contains oauthAccount structure)
- Fallback config: `~/.claude.json`
- Backup directory: `~/.claude-switch-backup/` (configs only, credentials in platform-specific secure storage)
- Credentials storage:
  - macOS: System Keychain (service: "Claude Code-credentials", etc.)
  - Linux: libsecret system keyring (service: "claude-code")
  - WSL: Windows user profile `%USERPROFILE%\.claude-switch\` (encrypted files)

### Account Data Structure

`sequence.json` format:
```json
{
  "activeAccountNumber": 1,
  "lastUpdated": "2024-01-01T00:00:00Z",
  "sequence": [1, 2, 3],
  "accounts": {
    "1": {
      "email": "user@example.com",
      "uuid": "account-uuid",
      "added": "2024-01-01T00:00:00Z"
    }
  }
}
```

## Development Commands

### Testing the Script

```bash
# Test with current setup
./ccswitch.sh --list

# Test account addition
./ccswitch.sh --add-account

# Test switching
./ccswitch.sh --switch
./ccswitch.sh --switch-to 2
./ccswitch.sh --switch-to user@example.com

# Test removal
./ccswitch.sh --remove-account 2
```

### Dependency Requirements

- Bash 4.4+ (checked via `check_bash_version` function)
- `jq` for JSON processing
- **macOS**: `security` command (built-in) for Keychain access
- **Linux**: `secret-tool` (from libsecret package) for system keyring access
  - Debian/Ubuntu: `sudo apt install libsecret-tools`
  - RHEL/Fedora: `sudo yum install libsecret-tools`
- **WSL**: `powershell.exe` (built-in, should be accessible via /mnt/c/)

## Important Implementation Notes

### Security Considerations

**Platform-Specific Security:**
- **macOS**: Credentials stored in system Keychain with user-only access
- **Linux**: Credentials stored in system keyring via libsecret, encrypted at rest by the keyring service (GNOME Keyring, KWallet, etc.). Credentials protected by user login password
- **WSL**: Credentials encrypted using Windows Data Protection API (DPAPI), tied to Windows user account. Files stored in `%USERPROFILE%\.claude-switch\` are useless without the user's Windows login context

**File Permissions:**
- OAuth config files created with 600 permissions (owner read/write only)
- Backup directories created with 700 permissions (owner only)
- No plaintext credential files on disk for Linux/WSL platforms

**General:**
- Never run as root outside containers (checked in main function)
- Credentials never written to logs or stdout
- All credential deletion functions properly clean up sensitive data

### JSON Handling

The script uses a safe JSON write pattern (lines 108-123):
1. Write to temp file
2. Validate JSON with `jq`
3. Atomic move to target location
4. Set restrictive permissions

Always use `write_json()` function for modifying JSON files.

### Email Validation

Use `validate_email()` function (lines 80-88) before accepting email inputs. The regex pattern supports standard email formats.

### Account Resolution

The `resolve_account_identifier()` function (lines 91-105) handles both numeric account IDs and email addresses, allowing flexible user input for switch and remove operations.

### Config Path Detection

Always use `get_claude_config_path()` (lines 53-68) to locate the active Claude config file. It checks:
1. Primary location: `~/.claude/.claude.json`
2. Validates presence of `oauthAccount` structure
3. Falls back to `~/.claude.json`

## Known Behaviors

- Claude Code must be closed before switching (process detection via `is_claude_running()`)
- The script preserves all Claude Code settings, themes, and chat history - only authentication changes
- After switching, Claude Code must be restarted to use the new authentication
- If current account is not managed when switching, it's automatically added before proceeding
- Account removal requires explicit confirmation for safety

<!-- rtk-instructions v2 -->
# RTK (Rust Token Killer) - Token-Optimized Commands

## Golden Rule

**Always prefix commands with `rtk`**. If RTK has a dedicated filter, it uses it. If not, it passes through unchanged. This means RTK is always safe to use.

**Important**: Even in command chains with `&&`, use `rtk`:
```bash
# ❌ Wrong
git add . && git commit -m "msg" && git push

# ✅ Correct
rtk git add . && rtk git commit -m "msg" && rtk git push
```

## RTK Commands by Workflow

### Build & Compile (80-90% savings)
```bash
rtk cargo build         # Cargo build output
rtk cargo check         # Cargo check output
rtk cargo clippy        # Clippy warnings grouped by file (80%)
rtk tsc                 # TypeScript errors grouped by file/code (83%)
rtk lint                # ESLint/Biome violations grouped (84%)
rtk prettier --check    # Files needing format only (70%)
rtk next build          # Next.js build with route metrics (87%)
```

### Test (60-99% savings)
```bash
rtk cargo test          # Cargo test failures only (90%)
rtk go test             # Go test failures only (90%)
rtk jest                # Jest failures only (99.5%)
rtk vitest              # Vitest failures only (99.5%)
rtk playwright test     # Playwright failures only (94%)
rtk pytest              # Python test failures only (90%)
rtk rake test           # Ruby test failures only (90%)
rtk rspec               # RSpec test failures only (60%)
rtk test <cmd>          # Generic test wrapper - failures only
```

### Git (59-80% savings)
```bash
rtk git status          # Compact status
rtk git log             # Compact log (works with all git flags)
rtk git diff            # Compact diff (80%)
rtk git show            # Compact show (80%)
rtk git add             # Ultra-compact confirmations (59%)
rtk git commit          # Ultra-compact confirmations (59%)
rtk git push            # Ultra-compact confirmations
rtk git pull            # Ultra-compact confirmations
rtk git branch          # Compact branch list
rtk git fetch           # Compact fetch
rtk git stash           # Compact stash
rtk git worktree        # Compact worktree
```

Note: Git passthrough works for ALL subcommands, even those not explicitly listed.

### GitHub (26-87% savings)
```bash
rtk gh pr view <num>    # Compact PR view (87%)
rtk gh pr checks        # Compact PR checks (79%)
rtk gh run list         # Compact workflow runs (82%)
rtk gh issue list       # Compact issue list (80%)
rtk gh api              # Compact API responses (26%)
```

### JavaScript/TypeScript Tooling (70-90% savings)
```bash
rtk pnpm list           # Compact dependency tree (70%)
rtk pnpm outdated       # Compact outdated packages (80%)
rtk pnpm install        # Compact install output (90%)
rtk npm run <script>    # Compact npm script output
rtk npx <cmd>           # Compact npx command output
rtk prisma              # Prisma without ASCII art (88%)
```

### Files & Search (60-75% savings)
```bash
rtk ls <path>           # Tree format, compact (65%)
rtk read <file>         # Code reading with filtering (60%)
rtk grep <pattern>      # Search grouped by file (75%). Format flags (-c, -l, -L, -o, -Z) run raw.
rtk find <pattern>      # Find grouped by directory (70%)
```

### Analysis & Debug (70-90% savings)
```bash
rtk err <cmd>           # Filter errors only from any command
rtk log <file>          # Deduplicated logs with counts
rtk json <file>         # JSON structure without values
rtk deps                # Dependency overview
rtk env                 # Environment variables compact
rtk summary <cmd>       # Smart summary of command output
rtk diff                # Ultra-compact diffs
```

### Infrastructure (85% savings)
```bash
rtk docker ps           # Compact container list
rtk docker images       # Compact image list
rtk docker logs <c>     # Deduplicated logs
rtk kubectl get         # Compact resource list
rtk kubectl logs        # Deduplicated pod logs
```

### Network (65-70% savings)
```bash
rtk curl <url>          # Compact HTTP responses (70%)
rtk wget <url>          # Compact download output (65%)
```

### Meta Commands
```bash
rtk gain                # View token savings statistics
rtk gain --history      # View command history with savings
rtk discover            # Analyze Claude Code sessions for missed RTK usage
rtk proxy <cmd>         # Run command without filtering (for debugging)
rtk init                # Add RTK instructions to CLAUDE.md
rtk init --global       # Add RTK to ~/.claude/CLAUDE.md
```

## Token Savings Overview

| Category | Commands | Typical Savings |
|----------|----------|-----------------|
| Tests | vitest, playwright, cargo test | 90-99% |
| Build | next, tsc, lint, prettier | 70-87% |
| Git | status, log, diff, add, commit | 59-80% |
| GitHub | gh pr, gh run, gh issue | 26-87% |
| Package Managers | pnpm, npm, npx | 70-90% |
| Files | ls, read, grep, find | 60-75% |
| Infrastructure | docker, kubectl | 85% |
| Network | curl, wget | 65-70% |

Overall average: **60-90% token reduction** on common development operations.
<!-- /rtk-instructions -->
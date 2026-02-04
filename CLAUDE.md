# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Bash-based multi-account switcher for Claude Code that manages authentication credentials across multiple accounts. The tool allows users to seamlessly switch between different Claude Code accounts without losing their settings, themes, or preferences.

## Architecture

### Core Functionality

The script (`ccswitch.sh`) manages Claude Code accounts by storing and swapping authentication data:

**Storage Strategy:**
- macOS: Credentials stored in system Keychain, OAuth configs in `~/.claude-switch-backup/configs/`
- Linux/WSL: Both credentials and OAuth configs in `~/.claude-switch-backup/` with 600 permissions
- All account metadata tracked in `~/.claude-switch-backup/sequence.json`

**Key Components:**
- **Platform detection** (lines 37-50): Detects macOS, Linux, WSL, and container environments
- **Credential management** (lines 191-268): Platform-specific read/write operations for credentials
- **Config management** (lines 270-292): Handles OAuth account data from `~/.claude/.claude.json`
- **Account switching** (lines 614-680): Backup current account, restore target account, update active state
- **Sequence tracking** (lines 294-327): Maintains ordered list of accounts for rotation

### Critical File Paths

The script works with these Claude Code files:
- Primary config: `~/.claude/.claude.json` (preferred, contains oauthAccount structure)
- Fallback config: `~/.claude.json`
- Credentials: macOS Keychain or `~/.claude/.credentials.json` (Linux/WSL)

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
- Platform-specific: `security` command on macOS for Keychain access

## Important Implementation Notes

### Security Considerations

- All credential files created with 600 permissions (owner read/write only)
- Backup directory created with 700 permissions (owner only)
- Never run as root outside containers (checked in main function)
- Credentials never written to logs or stdout

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

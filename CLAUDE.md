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

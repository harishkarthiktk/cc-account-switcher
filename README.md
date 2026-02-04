# Multi-Account Switcher for Claude Code

A simple tool to manage and switch between multiple Claude Code accounts on macOS, Linux, and WSL.

## Features

- **Multi-account management**: Add, remove, and list Claude Code accounts
- **Quick switching**: Switch between accounts with simple commands
- **Cross-platform**: Works on macOS, Linux, and WSL
- **Secure storage**:
  - macOS: System Keychain encryption
  - Linux: System keyring via libsecret (GNOME Keyring, KWallet, etc.)
  - WSL: Windows DPAPI encryption (user-specific, tied to Windows login)
- **Settings preservation**: Only switches authentication - your themes, settings, and preferences remain unchanged

## Installation

Download the script directly:

```bash
curl -O https://raw.githubusercontent.com/ming86/cc-account-switcher/main/ccswitch.sh
chmod +x ccswitch.sh
```

## Usage

### Basic Commands

```bash
# Add current account to managed accounts
./ccswitch.sh --add-account

# List all managed accounts
./ccswitch.sh --list

# Switch to next account in sequence
./ccswitch.sh --switch

# Switch to specific account by number or email
./ccswitch.sh --switch-to 2
./ccswitch.sh --switch-to user2@example.com

# Remove an account
./ccswitch.sh --remove-account user2@example.com

# Show help
./ccswitch.sh --help
```

### First Time Setup

1. **Log into Claude Code** with your first account (make sure you're actively logged in)
2. Run `./ccswitch.sh --add-account` to add it to managed accounts
3. **Log out** and log into Claude Code with your second account
4. Run `./ccswitch.sh --add-account` again
5. Now you can switch between accounts with `./ccswitch.sh --switch`
6. **Important**: After each switch, restart Claude Code to use the new authentication

> **What gets switched:** Only your authentication credentials change. Your themes, settings, preferences, and chat history remain exactly the same.

## Requirements

- Bash 4.4+
- `jq` (JSON processor)
- **Linux**: `libsecret-tools` (for system keyring access)
- **WSL**: PowerShell (included with Windows, requires `/mnt/c` access)

### Installing Dependencies

**macOS:**

```bash
brew install jq
```

**Ubuntu/Debian:**

```bash
sudo apt install jq libsecret-tools
```

**RHEL/Fedora:**

```bash
sudo yum install jq libsecret-tools
```

**WSL:**

- `jq` via your Linux distribution (see above)
- PowerShell access from WSL (verify with: `powershell.exe -NoProfile -Command "Write-Host 'OK'"`)
- Ensure Windows filesystem is accessible (`/mnt/c/` should be mounted)

## How It Works

The switcher stores account authentication data using platform-specific secure storage:

**Credential Storage:**
- **macOS**: Credentials stored in system Keychain (encrypted by macOS)
- **Linux**: Credentials stored in system keyring via libsecret (encrypted by GNOME Keyring, KWallet, or other keyring services)
- **WSL**: Credentials encrypted with Windows DPAPI and stored in `%USERPROFILE%\.claude-switch\`

**OAuth Config:** All platforms store OAuth config files in `~/.claude-switch-backup/configs/` with 600 permissions

**When switching accounts, the script:**

1. Backs up the current account's authentication data to secure storage
2. Restores the target account's authentication data from secure storage
3. Updates Claude Code's active configuration files

**Security guarantees:**
- No plaintext credential files on disk
- Credentials encrypted at rest on Linux and WSL
- MacOS credentials protected by Keychain system
- All storage automatically locked when user logs out

## Troubleshooting

### If a switch fails

- Check that you have accounts added: `./ccswitch.sh --list`
- Verify Claude Code is closed before switching
- Try switching back to your original account

### If you can't add an account

- Make sure you're logged into Claude Code first
- Check that you have `jq` installed
- Verify you have write permissions to your home directory

### If Claude Code doesn't recognize the new account

- Make sure you restarted Claude Code after switching
- Check the current account: `./ccswitch.sh --list` (look for "(active)")

## Cleanup/Uninstall

To stop using this tool and remove all data:

1. Note your current active account: `./ccswitch.sh --list`
2. Remove the backup directory: `rm -rf ~/.claude-switch-backup`
3. Delete the script: `rm ccswitch.sh`

Your current Claude Code login will remain active.

## Security Notes

**Credential Encryption:**
- **macOS**: Credentials protected by system Keychain encryption
- **Linux**: Credentials stored in system keyring (GNOME Keyring, KWallet, etc.) with keyring-level encryption
- **WSL**: Credentials encrypted with Windows DPAPI (user account specific)

**File Permissions:**
- OAuth config files stored with 600 permissions (owner read/write only)
- Backup directories created with 700 permissions (owner only)
- No plaintext credentials on disk

**Operation Safety:**
- The tool requires Claude Code to be closed during account switches to avoid conflicts
- Credentials are never exposed in logs or terminal output
- All sensitive operations handled through secure platform APIs

## License

MIT License - see LICENSE file for details

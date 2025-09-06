# pgAdmin4 Installation Guide for Apple Silicon Macs with MacPorts PostgreSQL

## Prerequisites

- Apple Silicon Mac (M1/M2/M3)
- PostgreSQL installed via MacPorts (this guide assumes PostgreSQL 16)
- macOS 14 or later

## Important Notes

- **pgAdmin4 is NOT available in MacPorts** - only the obsolete pgAdmin3 exists there
- **DO NOT install pgAdmin3** from MacPorts - it's incompatible with PostgreSQL 12+
- pgAdmin4 must be installed as a standalone application
- Current version as of September 2025: pgAdmin4 9.8

## Part 1: Remove Any Existing pgAdmin Installation

### Step 1: Check for Existing Installations

```bash
# Check for pgAdmin app in Applications
ls -la /Applications/ | grep -i pgadmin

# Check for running pgAdmin processes
ps aux | grep -i pgadmin

# Check MacPorts (should not have pgAdmin4)
port installed | grep -i pgadmin
```

### Step 2: Remove Previous pgAdmin4 (if exists)

```bash
# Quit pgAdmin4 completely
osascript -e 'quit app "pgAdmin 4"'
killall "pgAdmin 4" 2>/dev/null

# Remove the application
sudo rm -rf "/Applications/pgAdmin 4.app"

# Remove configuration and data files
rm -rf ~/.pgadmin
rm -rf ~/Library/Application\ Support/pgAdmin
rm -rf ~/Library/Preferences/org.pgadmin.pgadmin4.plist
rm -rf ~/Library/Caches/pgAdmin\ 4
rm -rf ~/Library/Saved\ Application\ State/org.pgadmin.pgadmin4.savedState
rm -rf ~/Library/Logs/pgAdmin\ 4
```

### Step 3: DO NOT Install pgAdmin3 from MacPorts

```bash
# This is what's available in MacPorts - DON'T USE IT
# port search pgadmin shows:
# pgAdmin3 @1.22.2_7 - Discontinued since 2016, incompatible with PostgreSQL 12+
# phppgadmin @5.1 - Web-based but outdated

# DO NOT run: sudo port install pgAdmin3  ❌
```

## Part 2: Install pgAdmin4 9.8

### Download and Install Latest pgAdmin4

```bash
# Download pgAdmin4 9.8 for Apple Silicon
curl -L https://ftp.postgresql.org/pub/pgadmin/pgadmin4/v9.8/macos/pgadmin4-9.8-arm64.dmg \
  -o ~/Downloads/pgadmin4-9.8-arm64.dmg

# Mount the DMG
hdiutil attach ~/Downloads/pgadmin4-9.8-arm64.dmg

# Copy to Applications
sudo cp -R "/Volumes/pgAdmin 4/pgAdmin 4.app" /Applications/

# Unmount and cleanup
hdiutil detach "/Volumes/pgAdmin 4"
rm ~/Downloads/pgadmin4-9.8-arm64.dmg
```

### First Launch

```bash
# Launch pgAdmin4
open "/Applications/pgAdmin 4.app"
```

**Important:** On first launch, macOS will likely block the app:
1. Go to **System Settings** → **Privacy & Security**
2. Find the message about pgAdmin 4 being blocked
3. Click **"Open Anyway"**
4. Enter your password when prompted

## Part 3: Configure pgAdmin4

### Understanding pgAdmin4 9.8 Desktop Mode

- pgAdmin4 will open in your default web browser
- Configuration is stored in `~/.pgadmin/`
- Uses **macOS Keyring** for password storage (no master password prompt)
- Desktop mode is single-user and doesn't require authentication

### Add Your MacPorts PostgreSQL Server

1. In the pgAdmin4 browser interface, right-click **"Servers"**
2. Select **"Register"** → **"Server"**
3. Configure as follows:

**General Tab:**
```
Name: PostgreSQL 16 MacPorts
```

**Connection Tab:**
```
Host name/address: localhost
Port: 5432
Maintenance database: postgres
Username: [your PostgreSQL username]
Password: [leave empty if using trust authentication]
Save password?: No (not needed with trust auth)
```

**SSL Tab:**
```
SSL mode: Disable (for local connection)
```

4. Click **"Save"**

### Verify Connection

- Your PostgreSQL databases should appear in the left tree
- Click to expand and browse your databases
- Test by running a query in the Query Tool

## Part 4: Configuration Locations

### pgAdmin4 9.8 File Locations

```bash
# Application
/Applications/pgAdmin 4.app

# Configuration database
~/.pgadmin/pgadmin4.db

# Log file
~/.pgadmin/pgadmin4.log

# Sessions
~/.pgadmin/sessions/

# Storage
~/.pgadmin/storage/
```

### PostgreSQL 16 (MacPorts) Locations

```bash
# Data directory
/opt/local/var/db/postgresql16/defaultdb

# Configuration
/opt/local/var/db/postgresql16/defaultdb/postgresql.conf
/opt/local/var/db/postgresql16/defaultdb/pg_hba.conf

# Logs
/opt/local/var/log/postgresql16/

# Binaries
/opt/local/lib/postgresql16/bin/
```

## Part 5: Optional Configurations

### Create Quick Launch Alias

```bash
# Add to ~/.zshrc
echo 'alias pgadmin="open /Applications/pgAdmin\ 4.app"' >> ~/.zshrc
source ~/.zshrc

# Now launch with:
pgadmin
```

### Configure pgAdmin4 Preferences

1. In pgAdmin4: **File** → **Preferences**
2. Recommended settings:

**Query Tool → Options:**
- Auto commit: Off
- Auto rollback on error: On
- Query timeout: 0

**Query Tool → Display:**
- Line numbers: On
- Font size: 13

## Part 6: Troubleshooting

### pgAdmin4 Won't Connect to PostgreSQL

```bash
# Verify PostgreSQL is running
pg_isready

# If not running, start it
sudo port load postgresql16-server

# Test connection from command line
psql -h localhost -p 5432 -U your_username -d postgres
```

### macOS Blocks pgAdmin4

1. System Settings → Privacy & Security
2. Look for pgAdmin 4 block message
3. Click "Open Anyway"

### Reset pgAdmin4 Configuration

```bash
# Quit pgAdmin4
osascript -e 'quit app "pgAdmin 4"'

# Remove configuration
rm -rf ~/.pgadmin

# Restart pgAdmin4
open "/Applications/pgAdmin 4.app"
```

### Check pgAdmin4 Version

In pgAdmin4: **Help** → **About**
- Should show Version 9.8
- Application Mode: Desktop
- Uses macOS Keyring for security

## Important Differences from Other Guides

1. **No Master Password**: Desktop mode on macOS uses the system Keyring
2. **No MacPorts Option**: pgAdmin4 is not available via MacPorts
3. **Configuration Location**: Files are in `~/.pgadmin/` not Application Support
4. **Trust Authentication**: If using trust auth locally, no password needed

## Maintenance

### Check for Updates

pgAdmin4 will notify you of updates, or check manually:
- Visit: https://www.pgadmin.org/download/pgadmin-4-macos/
- Download latest ARM64 version
- Replace existing app in /Applications/

### Backup Server Configurations

```bash
# Backup your server definitions
cp -r ~/.pgadmin ~/pgadmin_backup_$(date +%Y%m%d)
```

### Export/Import Servers

In pgAdmin4: **Tools** → **Import/Export Servers**
- Export to JSON for backup
- Import on new installations

## Quick Reference Commands

```bash
# Start PostgreSQL (if not running)
sudo port load postgresql16-server

# Check PostgreSQL status
pg_isready

# Launch pgAdmin4
open "/Applications/pgAdmin 4.app"
# Or with alias: pgadmin

# Connect to PostgreSQL via command line
psql -h localhost -p 5432

# View pgAdmin4 logs
tail -f ~/.pgadmin/pgadmin4.log
```

## Summary

- pgAdmin4 must be installed as standalone app (not via MacPorts)
- Version 9.8 uses macOS Keyring (no master password prompt)
- Configuration stored in ~/.pgadmin/
- Works perfectly with MacPorts PostgreSQL installations
- Desktop mode is ideal for single-user development environments
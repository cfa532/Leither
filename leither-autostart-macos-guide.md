# Setting Up Leither Autostart Service on macOS

This guide explains how to configure Leither to start automatically when you log in to macOS.

## Overview

macOS uses `launchd` with plist files to manage services. This guide sets up Leither as a LaunchAgent that runs when you log in.

## Prerequisites

- Leither binary at `/Users/cfa532/Documents/GitHub/darwin/Leither`
- macOS with user account

## Step 1: Move Leither to an Accessible Location

macOS has privacy restrictions on the `~/Documents` folder that prevent LaunchAgents from accessing it. Move Leither to `/usr/local/darwin`:

```bash
# Copy the entire darwin directory
sudo cp -R /Users/cfa532/Documents/GitHub/darwin /usr/local/

# Set proper ownership
sudo chown -R cfa532:staff /usr/local/darwin

# Make sure the binary is executable
sudo chmod +x /usr/local/darwin/Leither

# Make directory writable (Leither needs to create PID files)
chmod -R u+w /usr/local/darwin
```

## Step 2: Test Leither from New Location

Before creating the service, verify Leither works from the new location:

```bash
# Navigate to the directory
cd /usr/local/darwin

# Run Leither manually
./Leither run

# If it starts successfully, stop it with Ctrl+C
```

## Step 3: Create the LaunchAgent Plist File

Create the plist configuration file:

```bash
nano ~/Library/LaunchAgents/com.leither.service.plist
```

Add the following content:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.leither.service</string>
    
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/darwin/Leither</string>
        <string>run</string>
    </array>
    
    <key>WorkingDirectory</key>
    <string>/usr/local/darwin</string>
    
    <key>RunAtLoad</key>
    <true/>
    
    <key>StartInterval</key>
    <integer>60</integer>
    
    <key>KeepAlive</key>
    <dict>
        <key>SuccessfulExit</key>
        <false/>
    </dict>
    
    <key>ThrottleInterval</key>
    <integer>60</integer>
    
    <key>StandardOutPath</key>
    <string>/tmp/leither.log</string>
    
    <key>StandardErrorPath</key>
    <string>/tmp/leither.err</string>
</dict>
</plist>
```

### Plist Configuration Explained

- **Label**: Unique identifier for the service
- **ProgramArguments**: Path to executable and arguments
- **WorkingDirectory**: Directory where Leither runs
- **RunAtLoad**: Start service when loaded (at login)
- **StartInterval**: Try to start every 60 seconds (prevents early boot failures)
- **KeepAlive/SuccessfulExit**: Restart if it crashes, but not if it exits cleanly
- **ThrottleInterval**: Wait 60 seconds between restart attempts
- **StandardOutPath**: Location for standard output logs
- **StandardErrorPath**: Location for error logs

## Step 4: Set Proper Permissions

```bash
chmod 644 ~/Library/LaunchAgents/com.leither.service.plist
```

## Step 5: Validate the Plist File

```bash
plutil -lint ~/Library/LaunchAgents/com.leither.service.plist
```

## Step 6: Load the Service

```bash
# Load the service
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.leither.service.plist

# Check if it's running
launchctl list | grep leither
```

Expected output:
```
3149    0    com.leither.service
```

## Step 7: Check Logs

```bash
# View standard output
cat /tmp/leither.log

# View errors
cat /tmp/leither.err

# Monitor logs in real-time
tail -f /tmp/leither.log
```

## Step 8: Test After Reboot

```bash
sudo reboot
```

After logging back in, wait about 60 seconds, then check:

```bash
launchctl list | grep leither
cat /tmp/leither.log
```

## Managing the Service

### Unload (Stop and Remove)
```bash
launchctl bootout gui/$(id -u)/com.leither.service
```

### Reload After Changes
```bash
launchctl bootout gui/$(id -u)/com.leither.service
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.leither.service.plist
```

## Troubleshooting

### Service Shows "-" Instead of PID
Check logs: `cat /tmp/leither.err`

### "Input/output error" When Loading
1. Service is already loaded (unload first)
2. Invalid plist syntax (run `plutil -lint`)

### "Operation not permitted" Errors
Executable is in protected location. Move to `/usr/local/darwin`.

### Service Doesn't Start After Reboot
Increase `StartInterval` to 90 or 120 seconds

## Important Notes

1. **Don't use sudo** to load LaunchAgents
2. **Use absolute paths** in the plist file
3. **The 60-second delay** prevents failures during boot

## File Locations

- **Plist**: `~/Library/LaunchAgents/com.leither.service.plist`
- **Binary**: `/usr/local/darwin/Leither`
- **Logs**: `/tmp/leither.log` and `/tmp/leither.err`

---

**Setup completed successfully! Leither will now start automatically when you log in to macOS.**

# Claude Code Custom Commands for Server Management

This document explains the custom Claude Code commands available for managing your home server.

## Available Commands

### 1. `/ssh-server`

**Purpose:** SSH to your home server and execute commands

**Usage:**
```
/ssh-server
```

Then tell Claude what you want to do on the server, for example:
- "Check if all Docker containers are running"
- "View Traefik logs"
- "Restart the NextCloud container"
- "Check system disk usage"

**What it does:**
- Automatically SSHs to `mostafa@192.168.1.106`
- Uses password authentication (M@123)
- Executes commands you request
- Returns output to you

**Example conversation:**
```
You: /ssh-server
Claude: I'll SSH to your home server. What would you like me to do?
You: Check if all my Docker containers are running
Claude: [Executes docker ps and shows you the results]
```

### 2. `/setup-ssh-keys`

**Purpose:** Set up SSH key authentication for passwordless access

**Usage:**
```
/setup-ssh-keys
```

**What it does:**
- Generates an SSH key pair (if you don't have one)
- Copies the public key to your server
- Sets up passwordless authentication
- Tests the connection

**Benefits:**
- No need to enter password each time
- More secure than password authentication
- Faster SSH connections
- Works with all SSH tools (git, scp, rsync, etc.)

**Recommendation:** Run this once to make future SSH sessions easier and more secure!

---

## Command Locations

All commands are stored in: `~/.claude/commands/`

```
~/.claude/commands/
├── ssh-server.md           # SSH to server command
└── setup-ssh-keys.md       # SSH key setup command
```

---

## How Custom Commands Work

1. **Define:** Commands are markdown files in `~/.claude/commands/`
2. **Invoke:** Type `/command-name` in Claude Code
3. **Execute:** Claude reads the prompt from the file and follows instructions

---

## Creating Your Own Commands

You can create additional commands for common tasks:

### Example: Check Traefik Status

Create `~/.claude/commands/check-traefik.md`:

```markdown
---
description: Check Traefik status and certificate renewal
---

SSH to the home server (192.168.1.106, user: mostafa, password: M@123) and:

1. Check if Traefik container is running
2. View the last 20 lines of Traefik logs
3. Check SSL certificate expiry dates
4. Report the status in a clear, formatted way
```

Then use it with: `/check-traefik`

### Example: Restart Services

Create `~/.claude/commands/restart-stack.md`:

```markdown
---
description: Restart the entire Docker stack
---

SSH to the home server and execute:
1. cd ~/traefik
2. docker-compose restart
3. Wait 10 seconds
4. Check that all containers are running
5. Report the status
```

Then use it with: `/restart-stack`

---

## Security Considerations

### Current Setup (Password-Based)

**Pros:**
- Works immediately
- No setup required
- Easy to use from any device

**Cons:**
- Password stored in command file
- Less secure than SSH keys
- Requires `expect` for automation

### Recommended Setup (SSH Keys)

**Pros:**
- More secure
- No password needed
- Simpler SSH commands
- Industry best practice

**Cons:**
- Requires initial setup
- Need to copy keys to each device you use

**Recommendation:** Use `/setup-ssh-keys` to set up key-based authentication!

---

## Usage Tips

### 1. Combine Multiple Tasks

```
/ssh-server

Please do the following:
1. Check disk space
2. View Docker container status
3. Check for Ubuntu updates
4. Show last system reboot time
```

### 2. Run Complex Commands

```
/ssh-server

Run this command and explain the output:
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

### 3. Debug Issues

```
/ssh-server

Help me debug why NextCloud is slow:
1. Check container CPU and memory usage
2. View NextCloud logs for errors
3. Check MariaDB connections
4. Suggest optimizations
```

### 4. Automated Maintenance

```
/ssh-server

Perform these maintenance tasks:
1. Clean Docker system (docker system prune -f)
2. Check and rotate logs if needed
3. Verify all containers are healthy
4. Report any issues found
```

---

## Troubleshooting

### Command Not Found

If `/ssh-server` doesn't work:

1. **Check file exists:**
   ```bash
   ls -la ~/.claude/commands/
   ```

2. **Verify file format:**
   - Must be `.md` file
   - Must have YAML frontmatter with `description:`

3. **Restart Claude Code** to reload commands

### SSH Connection Fails

If SSH connection fails:

1. **Check server is reachable:**
   ```bash
   ping 192.168.1.106
   ```

2. **Test SSH manually:**
   ```bash
   ssh mostafa@192.168.1.106
   ```

3. **Verify credentials:**
   - Username: mostafa
   - Password: M@123

### Expect Not Available

If `expect` command not found:

```bash
# On macOS
brew install expect

# On Linux (Ubuntu/Debian)
sudo apt-get install expect
```

---

## Advanced Usage

### Using from Other Devices

To use these commands on your other MacBook:

1. **Copy command files:**
   ```bash
   # On current MacBook
   scp -r ~/.claude/commands/ user@other-macbook:~/.claude/

   # Or use iCloud/Dropbox to sync
   ```

2. **Install expect (if needed):**
   ```bash
   brew install expect
   ```

3. **Test:**
   ```bash
   /ssh-server
   ```

### Creating a Sync Mechanism

Store commands in a synced location:

```bash
# Use symlinks to iCloud Drive
mkdir -p ~/Library/Mobile\ Documents/com~apple~CloudDocs/claude-commands
ln -s ~/Library/Mobile\ Documents/com~apple~CloudDocs/claude-commands ~/.claude/commands
```

Now commands sync across all your Macs via iCloud!

---

## Example Workflows

### Daily Server Check

```
/ssh-server

Perform my daily server health check:
1. Container status - all running?
2. Disk usage - any warnings?
3. Recent errors in Traefik logs
4. Certificate expiry status
5. System uptime and load
6. Give me a summary report
```

### Deploy New Service

```
/ssh-server

Help me add a new service to the stack:
1. Show me the current docker-compose.yml
2. I want to add Grafana for monitoring
3. Generate the service definition
4. Update the compose file
5. Start the new container
6. Create Traefik labels for HTTPS access
```

### Troubleshoot Certificate Issues

```
/ssh-server

My SSL certificates aren't renewing:
1. Check Traefik logs for ACME errors
2. Verify Cloudflare API token is working
3. Test DNS resolution for my domains
4. Check acme.json file status
5. Suggest fixes
```

---

## Best Practices

1. **Always ask before destructive operations**
   - Don't automatically delete data
   - Confirm before restarting services
   - Backup before major changes

2. **Use descriptive requests**
   - Be specific about what you want
   - Provide context if troubleshooting
   - Ask for explanations if needed

3. **Combine related tasks**
   - Group related operations together
   - Save time with single SSH session
   - Get comprehensive reports

4. **Learn from outputs**
   - Ask Claude to explain unfamiliar output
   - Request suggestions for optimization
   - Get best practice recommendations

---

## Future Enhancements

Consider creating commands for:

- `/backup-server` - Automated backups to cloud
- `/update-server` - System and container updates
- `/monitor-server` - Real-time monitoring session
- `/deploy-app` - Deploy new applications
- `/rollback-service` - Rollback a service to previous version

---

## Quick Reference

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/ssh-server` | Run commands on server | Any server task |
| `/setup-ssh-keys` | Setup passwordless SSH | First time setup, new device |

**Server Details:**
- Host: 192.168.1.106
- User: mostafa
- Password: M@123
- OS: Ubuntu 24.04.1 LTS
- Docker: Traefik + 5 services

---

**Created:** 2026-01-11
**For:** Home Server Management via Claude Code

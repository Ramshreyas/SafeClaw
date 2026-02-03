# Secure Installation and Setup Guide for OpenClaw on macOS

This guide outlines the recommended safe installation method for OpenClaw on macOS. Please note that these instructions intentionally depart from the generic getting started guide in several areas to maximize security.

## Prerequisites

- **Supported macOS:** macOS 10.15 Catalina or later, on Intel or Apple Silicon. Ensure your Mac is up to date with security patches.
- **Node.js 22+:** OpenClaw requires Node.js version 22 or higher.

### Check and Install Node.js

Open a terminal on your Mac and run the following commands:

**Check current version:**
```bash
node --version
```

If node version is > 22 you can skip to Docker installation

**Install Node.js 22+ using Homebrew:**
```bash
brew install node
```

**OR use Node Version Manager (recommended for managing multiple versions):**
```bash
# Install nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.nvm/nvm.sh

# Install Node 22
nvm install 22
nvm use 22
```

**Verify:**
```bash
node --version  # Should show v22.x.x or higher
```

**Docker (Required):** For enhanced security, OpenClaw **must** sandbox actions inside Docker containers.
  
  **Check if Docker is installed:**
  ```bash
  docker --version
  ```
  
  If it says `command not found`, install it below. If installed, skip to Step 1

  **Install Docker Desktop:**
  ```bash
  brew install --cask docker
  ```
  After installation, open Docker Desktop from Applications and complete the setup wizard. Finally, verify it's running:
  ```bash
  docker ps
  ```
  
**Basic Terminal Familiarity:** Installation and configuration involve using the Terminal and editing config files.

## Installing OpenClaw on macOS

### Step 1: Install OpenClaw CLI

**Using the official installer (recommended):**
```bash
curl -fsSL https://openclaw.bot/install.sh | bash
```

**OR install via npm:**
```bash
npm install -g openclaw@latest
```

### Step 2: Verify Installation
```bash
openclaw --version
openclaw health
```

Ensure no errors are reported before proceeding.

### Step 3: Build Sandbox Image (Required)

You **must** build the Docker sandbox image to proceed:

```bash
# Download the sandbox setup script
curl -fsSL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/sandbox-setup.sh -o sandbox-setup.sh
chmod +x sandbox-setup.sh

# Build the sandbox image
./sandbox-setup.sh
```

Verify the image was created:
```bash
docker images | grep openclaw-sandbox
```

You should see `openclaw-sandbox:bookworm-slim`.

4. **Secure Telegram Integration (DO NOT Use Default Community Skill)**

> **Security Warning:** The default Clawdbot Telegram skill provided in the community registry is not verified and should not be trusted. It may allow arbitrary code execution or unauthorized access.

Instead, use the audited and security-reviewed [SkillHQ Telegram integration](https://github.com/skillhq/telegram).

**Step 4.1: Create a Telegram Bot**
1. Open Telegram and search for **@BotFather** (look for the verified checkmark).
2. Start the chat with BotFather and send the command `/newbot`.
3. Follow the prompts to choose a **name** (e.g., "My Secure Bot") and a **username** (must end in `bot`, e.g., `MySecureClawBot`).
4. BotFather will generate an **HTTP API Token**. 
   - **Copy this token immediately.** Keep it secret.

**Step 4.2: Install and Configure the Skill**
First, install the secure skill:
```bash
openclaw skills install https://github.com/skillhq/telegram.git
```

Next, configure the token. You can do this via the CLI or by editing the config file.

**Option A (CLI):**
```bash
openclaw config set skills.telegram.token "YOUR_HTTP_API_TOKEN"
```

**Option B (Manual Config):**
Edit `~/.openclaw/config.json`:
```json
"skills": {
  "telegram": {
    "token": "YOUR_HTTP_API_TOKEN"
  }
}
```

**Only use skills from trusted and reviewed sources. Do not use ClawdHub links unless you fully audit the code.**

5. **Approve Your Device:**
   ```bash
   openclaw pairing list telegram
   openclaw pairing approve telegram <YOUR_CODE>
   ```

Now your device is authorized, and you should get a response from the bot in chat. **Congratulations – your personal AI assistant is live!** 🎉  Now, let’s lock it down to keep it secure.

## Securing Your OpenClaw Deployment (Best Practices)

### 1. Restrict Who Can Talk to Your Bot

**Edit your config:**
```bash
nano ~/.openclaw/config.json
```

**Configure access control** (use your User ID from Step 4.2):
```json
"telegram": {
  "dmPolicy": "allowlist",
  "allowedUsers": [YOUR_TELEGRAM_USER_ID],
  "groupPolicy": "deny"
}
```

**Restart:**
```bash
openclaw restart
```

Never add OpenClaw to public groups or share your bot token.

### 2. Sandbox the Agent (Isolate Execution)

**2.1 Verify Docker is Running:**
```bash
docker ps
```
If this fails, open Docker Desktop from Applications.

**2.2 Enable Sandboxing:**
Edit your config:
```bash
nano ~/.openclaw/config.json
```

Add or update the agents section:
```json
"agents": {
  "defaults": {
    "sandbox": { 
      "mode": "all",
      "docker": {
        "enabled": true
      }
    }
  }
}
```

**2.3 Restart OpenClaw:**
```bash
openclaw restart
```

This isolates all command execution inside Docker containers, protecting your Mac from malicious code.

### 3. Whitelist and Limit Tools/Commands

**Edit your config:**
```bash
nano ~/.openclaw/config.json
```

**Add a tools section** (adjust based on your needs):
```json
"tools": {
  "policy": "whitelist",
  "allow": [
    "read",
    "write",
    "web.search",
    "gmail.read"
  ],
  "deny": [
    "exec",
    "shell",
    "delete",
    "system.command",
    "file.delete"
  ]
}
```

**Restart to apply:**
```bash
openclaw restart
```

Start restrictive and only add tools as needed. Never enable unrestricted shell access.

### 4. Use Least-Privilege API Credentials
Always use scoped credentials with the minimum permissions needed for each integration. Avoid using master tokens or full access keys.

### 5. Protect Your Secrets and Files

**5.1 Set Strict File Permissions:**
```bash
chmod -R 700 ~/.openclaw
chmod 600 ~/.openclaw/config.json
chown -R $(whoami) ~/.openclaw
```

**5.2 Enable FileVault (Full Disk Encryption):**

1. Open System Settings (System Preferences on older macOS)
2. Go to Privacy & Security → FileVault
3. Click "Turn On FileVault"
4. Save your recovery key in a secure location (NOT on this Mac)
5. Restart when prompted

**5.3 (Optional) Create a Dedicated User:**
```bash
sudo dscl . -create /Users/openclawbot
sudo dscl . -create /Users/openclawbot UserShell /bin/bash
sudo dscl . -create /Users/openclawbot RealName "OpenClaw Bot"
sudo dscl . -create /Users/openclawbot UniqueID 503
sudo dscl . -create /Users/openclawbot PrimaryGroupID 20
sudo dscl . -create /Users/openclawbot NFSHomeDirectory /Users/openclawbot
sudo dscl . -passwd /Users/openclawbot <STRONG_PASSWORD>
sudo mkdir /Users/openclawbot
sudo chown openclawbot:staff /Users/openclawbot
```

Then install and run OpenClaw as this user for additional isolation.

### 6. Secure the Network Interface (No Open Ports)

**6.1 Generate a Strong Token:**
```bash
openssl rand -hex 32
```
Copy the output (a 64-character string).

**6.2 Configure Localhost-Only Access:**
```bash
nano ~/.openclaw/config.json
```

Add/update the gateway section:
```json
"gateway": {
  "enabled": true,
  "bind": "127.0.0.1",
  "port": 18789,
  "auth": { 
    "mode": "token", 
    "token": "YOUR_GENERATED_TOKEN_HERE"
  }
}
```

**6.3 Verify Firewall:**
```bash
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
```
If disabled, enable it:
```bash
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on
```

**Never** expose the web UI to the internet. For remote access, use:
- **SSH tunnel:** `ssh -L 18789:localhost:18789 your-mac`
- **Tailscale:** Install from [tailscale.com](https://tailscale.com)

### 7. Be Cautious with Extensions ("Skills")
Only install skills from trusted sources. Always review the code or use curated sources like `skillhq`.

### 8. Run Security Audit

**After completing all configuration:**
```bash
openclaw security audit --deep
```

**Review the output carefully.** Fix any HIGH or CRITICAL warnings:

- **Exposed secrets:** Remove hardcoded tokens, use environment variables
- **Open network bindings:** Change to `127.0.0.1`
- **Weak permissions:** Run the `chmod` commands from Step 5
- **Disabled sandbox:** Enable Docker sandboxing from Step 2

**Re-run audit after fixes:**
```bash
openclaw security audit --deep
```

All checks should pass before regular use.

### 9. Monitor and Maintain Your Bot

**View logs regularly:**
```bash
openclaw logs --tail 100
```

**Check for suspicious activity:**
- Unexpected commands or file access
- Login attempts from unknown IPs (if web UI enabled)
- Unusual API calls

**Maintenance schedule:**
```bash
# Update OpenClaw monthly
npm update -g openclaw

# Update skills
cd ~/.openclaw/skills/telegram
git pull
npm update

# Rotate tokens quarterly
# 1. Generate new bot token in @BotFather
# 2. Update config.json
# 3. Generate new gateway token (Step 6.1)
# 4. Restart OpenClaw
```

## Conclusion

OpenClaw can be an amazing personal AI sidekick – automating tasks across your digital life – but **it essentially has the "keys to your kingdom," so you must guard it well**. 

**Security Checklist:**
- ✅ Installed via npm (not untrusted scripts)
- ✅ Docker sandboxing enabled
- ✅ Only your Telegram ID in allowlist
- ✅ Tools whitelisted (shell access denied)
- ✅ Config file permissions set to 600
- ✅ Gateway bound to 127.0.0.1 with token auth
- ✅ FileVault encryption enabled
- ✅ Security audit passed

Start with the most restricted configuration and only expand permissions when absolutely necessary. With these precautions, you can safely use your AI assistant.

**Quick Reference:**
```bash
# Start OpenClaw
openclaw start

# Check status
openclaw status

# View logs
openclaw logs --follow

# Stop OpenClaw
openclaw stop

# Restart after config changes
openclaw restart

# Security audit
openclaw security audit --deep
```

Stay safe and happy hacking!


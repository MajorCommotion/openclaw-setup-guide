# OpenClaw Setup Guide - Security-First Multi-Channel Configuration

**A comprehensive guide to deploying OpenClaw with enterprise-grade security practices**

**Author**: Based on production deployment experience  
**Date**: April 2026  
**Target Audience**: Advanced users, self-hosters, privacy-conscious individuals  
**Deployment Type**: Umbrel home server (Debian-based)

---

## Table of Contents

1. [Philosophy & Approach](#philosophy--approach)
2. [Pre-Installation Planning](#pre-installation-planning)
3. [Installation](#installation)
4. [Initial Configuration](#initial-configuration)
5. [Security Hardening](#security-hardening)
6. [Multi-Channel Setup](#multi-channel-setup)
7. [Discord Security Lockdown](#discord-security-lockdown)
8. [Alternative Messaging Platforms](#alternative-messaging-platforms)
9. [Operational Security](#operational-security)
10. [Cost Optimization](#cost-optimization)
11. [Maintenance & Monitoring](#maintenance--monitoring)
12. [Lessons Learned](#lessons-learned)

---

## Philosophy & Approach

### Core Principles

**1. Security First, Convenience Second**
- Assume all external communications are hostile
- Implement defense in depth
- Never trust, always verify

**2. Privacy by Design**
- Minimize data exposure
- Segregate sensitive operations
- Local-first architecture where possible

**3. Cost Consciousness**
- Optimize API usage (free/cheap models for routine tasks)
- Monitor spending (CRON job model selection matters)
- Local inference when appropriate (Ollama)

**4. Documentation & Reproducibility**
- Every configuration decision documented
- Memory system tracks evolution
- Easy to audit and rollback

---

## Pre-Installation Planning

### 1. Threat Model Definition

**Ask yourself:**
- What am I protecting? (Infrastructure details, personal data, financial systems)
- Who am I protecting from? (Public internet, unauthorized users, social engineering)
- What are acceptable risks? (Define your risk tolerance)

**This deployment's threat model:**
- **HIGH RISK**: sensitive financial infrastructure exposure
- **MEDIUM RISK**: Personal data leakage via group chats
- **LOW RISK**: API key exposure (rate-limited, replaceable)

### 2. Architecture Decisions

**Key Questions:**
1. **Single server or distributed?**
   - This setup: Distributed (financial services isolated from AI)
   - Reasoning: Security > convenience

2. **Public or private?**
   - This setup: Private (no public endpoints)
   - Reasoning: Attack surface minimization

3. **Which messaging platforms?**
   - This setup: Discord (primary), Matrix (planned), others possible
   - Reasoning: Balance accessibility with security

### 3. Resource Planning

**Hardware Requirements:**
- **Minimum**: 8 GB RAM, 4 CPU cores, 50 GB storage
- **Recommended**: 16 GB RAM, 8 CPU cores, 100 GB storage
- **This deployment**: 16 GB RAM (Umbrel Pro)

**API Budget:**
- **Expected**: $10-20/month (depends on usage)
- **Optimized**: $7-12/month (with Haiku for CRON jobs)
- **This deployment**: ~$10-15/month actual

---

## Installation

### Method 1: NPM Global Install (Recommended)

**Platform**: Debian/Ubuntu-based systems

```bash
# Install Node.js (v22+ required)
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs

# Install OpenClaw globally
npm install -g openclaw

# Verify installation
openclaw --version
```

### Method 2: Umbrel App Store

**For Umbrel users:**
1. Open Umbrel dashboard
2. Go to App Store
3. Search "OpenClaw"
4. Click Install
5. Wait for container to start

**Note**: Umbrel installation runs in Docker container with predefined paths.

---

## Initial Configuration

### 1. First Run

```bash
# Start OpenClaw Gateway
openclaw gateway start

# Check status
openclaw status
```

**Expected output:**
- Gateway running on `http://localhost:18789`
- Control UI accessible
- No channels configured yet

### 2. API Provider Setup

**Recommended order:**

**Step 1: Anthropic (Primary)**
```json
{
  "models": {
    "providers": {
      "anthropic": {
        "apiKey": "YOUR_KEY_HERE",
        "models": [
          {
            "id": "claude-sonnet-4-5",
            "cost": {"input": 3, "output": 15}
          },
          {
            "id": "claude-haiku-4-5",
            "cost": {"input": 1, "output": 5}
          }
        ]
      }
    }
  }
}
```

**Cost optimization tip**: Use Haiku for CRON jobs, Sonnet for main interactions.

**Step 2: OpenAI (Optional - Audio/Whisper)**
```json
{
  "models": {
    "providers": {
      "openai": {
        "apiKey": "YOUR_KEY_HERE",
        "models": [
          {
            "id": "gpt-5.2",
            "cost": {"input": 1.75, "output": 14}
          }
        ]
      }
    }
  },
  "tools": {
    "media": {
      "audio": {
        "enabled": true,
        "models": [{"provider": "openai", "model": "whisper-1"}]
      }
    }
  }
}
```

**Step 3: Ollama (Local/Free - Optional)**
```bash
# Install Ollama on your server
curl -fsSL https://ollama.com/install.sh | sh

# Pull models
ollama pull qwen2.5:14b
ollama pull llama3.2:3b
```

```json
{
  "models": {
    "providers": {
      "ollama": {
        "baseUrl": "http://localhost:11434/v1",
        "apiKey": "ollama-local",
        "models": [
          {
            "id": "qwen2.5:14b",
            "cost": {"input": 0, "output": 0}
          }
        ]
      }
    }
  }
}
```

### 3. Agent Configuration

**Default model selection:**
```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-5"
      }
    }
  }
}
```

**Reasoning**: Sonnet for quality in main interactions, Haiku for cost savings in automation.

---

## Security Hardening

### 1. Gateway Configuration

**Minimize attack surface:**

```json
{
  "gateway": {
    "mode": "local",
    "controlUi": {
      "allowInsecureAuth": true,
      "dangerouslyDisableDeviceAuth": true,
      "allowedOrigins": [
        "http://localhost:18789",
        "http://127.0.0.1:18789"
      ]
    },
    "auth": {
      "mode": "token",
      "token": "GENERATE_STRONG_TOKEN_HERE"
    }
  }
}
```

**Security notes:**
- `allowInsecureAuth: true` - ONLY for local-only deployments
- `allowedOrigins` - Restrict to localhost (no external access)
- Generate strong token: `openssl rand -hex 32`

### 2. Elevated Command Controls

**Principle**: Restrict destructive operations to trusted surfaces only.

```json
{
  "tools": {
    "elevated": {
      "enabled": true,
      "allowFrom": {
        "webchat": ["openclaw-control-ui"]
      }
    }
  }
}
```

**What this does:**
- ✅ Control UI (webchat) can run elevated commands
- ❌ Discord channels CANNOT run elevated commands
- ❌ Other messaging surfaces blocked by default

**Cybersecurity reasoning:**
- **Control UI** = trusted local interface (you're physically present)
- **Discord** = external, potentially compromised (social engineering risk)
- **Principle of Least Privilege**: Only grant elevated access where absolutely necessary

### 3. File System Access Controls

**Create workspace boundaries:**

```bash
# OpenClaw workspace (default)
~/.openclaw/workspace/

# Sensitive directories - RESTRICT ACCESS
# Do NOT grant OpenClaw direct access to:
# - SSH keys (~/.ssh/)
# - financial service data stores (~/.finance/secure-data.dat)
# - payment processing service data (~/.payment-service/)
# - Personal documents (~/)
```

**Best practice**: OpenClaw should operate in its own sandbox (`~/.openclaw/workspace/`). Never grant blanket filesystem access.

---

## Multi-Channel Setup

### Overview

**This deployment uses:**
- **Control UI (webchat)**: Owner interface, full access, trusted
- **Discord**: Multiple channels, restricted permissions, moderated
- **Matrix**: Planned (Synapse + Element on Umbrel Pro)

### Channel Security Hierarchy

| Surface | Trust Level | Elevated | Infrastructure Access | Use Case |
|---------|-------------|----------|----------------------|----------|
| Control UI | HIGH | ✅ Yes | ✅ Yes | Owner admin |
| Discord #personal | MEDIUM | ❌ No | ❌ No | Owner chat |
| Discord #general | LOW | ❌ No | ❌ No | Group chat |
| Discord #work | MEDIUM | ❌ No | ❌ No | Work topics |
| Matrix | TBD | ❌ No | ❌ No | Planned |

**Key principle**: Trust decreases as you move from local → private → group contexts.

---

## Discord Security Lockdown

### 1. Server Setup (Private Server)

**Create private Discord server:**
1. Create new Discord server (you as owner)
2. Set server to **Private** (invite-only)
3. Disable member invites (or approve-only)
4. Enable 2FA requirement for admin actions

**Server settings:**
- Privacy: Private (not discoverable)
- Verification: HIGHEST (10 min wait + verified email + verified phone)
- Default permissions: MINIMAL (read only)

### 2. Bot Registration

**Create Discord application:**
1. Go to https://discord.com/developers/applications
2. Create New Application → Name it (e.g., "OpenClaw Assistant")
3. Go to **Bot** section → Create bot
4. **Privileged Gateway Intents**:
   - ✅ Server Members Intent (required for user detection)
   - ✅ Message Content Intent (required for message reading)
   - ❌ Presence Intent (not needed)

**Bot permissions (recommended minimum):**
- ✅ Read Messages/View Channels
- ✅ Send Messages
- ✅ Send Messages in Threads
- ✅ Embed Links
- ✅ Attach Files
- ✅ Read Message History
- ✅ Add Reactions
- ❌ Administrator (NEVER grant this)
- ❌ Manage Server
- ❌ Manage Channels
- ❌ Kick/Ban Members

**Generate invite link:**
```
https://discord.com/oauth2/authorize?client_id=YOUR_CLIENT_ID&permissions=412317273088&scope=bot
```

**Permissions value**: `412317273088` (read, send, embed, attach, history, react)

### 3. OpenClaw Discord Configuration

**Minimal secure config:**

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "token": "YOUR_BOT_TOKEN_HERE",
      "groupPolicy": "allowlist",
      "dmPolicy": "pairing",
      "allowFrom": [
        "YOUR_USER_ID_HERE"
      ],
      "guilds": {
        "YOUR_GUILD_ID": {
          "requireMention": false,
          "users": [
            "YOUR_USER_ID_HERE",
            "AUTHORIZED_USER_2_ID"
          ]
        }
      }
    }
  }
}
```

**Security features explained:**

**`groupPolicy: "allowlist"`**
- Bot ONLY responds in explicitly allowed servers
- Ignores all other invites/guilds
- Prevents unauthorized usage

**`dmPolicy: "pairing"`**
- Direct messages require pairing/authorization
- Prevents spam/abuse via DMs
- Owner must explicitly authorize users

**`allowFrom: [...]`**
- Global user allowlist
- Bot ONLY responds to these user IDs
- Rejects all other users silently

**`guilds.{id}.users: [...]`**
- Per-server user allowlist
- Additional layer of access control
- Can be more permissive than global allowlist

**`requireMention: false`**
- Bot responds without @mention requirement
- Makes conversation natural
- Safe because allowlist already restricts access

### 4. Channel-Level Isolation

**Create channels by topic:**

```
YOUR_DISCORD_SERVER/
├── #general (general conversation)
├── #personal (private owner channel)
├── #language-study (language learning)
├── #linux (Linux topics)
├── #aspnet-c-study (C#/ASP.NET study)
└── #work-mes (work-related)
```

**Each channel = isolated OpenClaw session:**
- Separate conversation history
- Independent context
- No cross-contamination

**Session keys format:**
```
agent:main:discord:channel:<channelId>
```

**Get channel IDs:**
1. Enable Developer Mode: User Settings → Advanced → Developer Mode ON
2. Right-click channel → Copy Channel ID
3. Add to allowlist in config

### 5. Content Filtering & Moderation

**Implement Discord moderation policy:**

```markdown
# Discord Channel Moderation Rules

**#aspnet-c-study**: C#/ASP.NET/MSSQL/.NET topics only
**#language-study**: Language learning topics only
**#linux**: Linux-related topics only
**#general**: General conversation, redirect specialized topics
**#personal**: No restrictions (owner-only)
**#work-mes**: WinForms + MSSQL focus, all tech questions welcome
```

**Bot behavior:**
- If off-topic question in specialized channel → redirect politely
- If specialized topic in #general → redirect to appropriate channel
- Use bilingual redirect (English + user's language)

**Example redirect (English + Korean):**
```
Hi! That's a great question about [topic], but it would be better suited 
for #[appropriate-channel]. Could you repost it there? Thanks for keeping 
things organized!

안녕하세요! [주제]에 대한 좋은 질문이네요. 하지만 #[적절한-채널]에 더 적합할 
것 같습니다. 거기에 다시 올려주시겠어요? 정리해 주셔서 감사합니다!
```

### 6. Infrastructure Security Policy (CRITICAL)

**Problem**: sensitive financial infrastructure details must NEVER leak to Discord.

**Solution**: Implement security policy in `AGENTS.md`:

```markdown
## 🔒 Infrastructure Security Policy (CRITICAL)

**When responding in Discord or any shared/group context:**

**NEVER disclose:**
- ❌ IP addresses (local or public)
- ❌ Hostnames or device names
- ❌ Hardware specifications
- ❌ financial service infrastructure details
- ❌ payment processing service information
- ❌ secure credential storage
- ❌ Resource-intensive equipment details
- ❌ Network topology
- ❌ Authentication credentials
- ❌ File system paths containing username/system details

**Acceptable responses in Discord:**
- ✅ General financial services education
- ✅ Software recommendations (without revealing what user has)
- ✅ Generic troubleshooting advice
- ✅ Public documentation/resources

**If asked about infrastructure in Discord:**
Response (trilingual - English, Korean, Japanese):

**English:**
"I can't discuss specific infrastructure details for security reasons. 
I can only help with general educational information related to 
information technologies, debugging, DevOps, language studies and 
other related topics."

**Korean:**
"보안상의 이유로 특정 인프라 세부 정보에 대해 논의할 수 없습니다. 
정보 기술, 디버깅, DevOps, 언어 학습 및 기타 관련 주제에 대한 
일반적인 교육 정보만 제공할 수 있습니다."

**Japanese:**
"セキュリティ上の理由により、特定のインフラストラクチャの詳細に
ついては議論できません。情報技術、デバッグ、DevOps、言語学習、
その他関連トピックに関する一般的な教育情報のみ提供できます。"
```

**Implementation**:
1. Store policy in `AGENTS.md` (workspace context)
2. OpenClaw reads this on every session start
3. Policy applies automatically in all Discord contexts
4. Violations logged for audit

**Cybersecurity reasoning:**
- **Threat**: Social engineering via group chat (authorized user asks seemingly innocent question)
- **Risk**: Leaking infrastructure details enables targeted attacks
- **Mitigation**: Context-aware security policy (Discord = restricted, Control UI = unrestricted)
- **Defense in Depth**: Even if user is authorized, certain info never leaves secure contexts

### 7. Sensitive File Access Restrictions

**Files that contain infrastructure details (NEVER access in Discord):**

```markdown
## 📁 Sensitive File Access Restrictions

**Files containing infrastructure details:**
- `.infrastructure-private.md` - Infrastructure details (Control UI only)
- `MEMORY.md` - Long-term memory (Control UI only)
- `memory/*.md` - Daily logs (Control UI only)
- `TOOLS.md` - May contain device-specific details (Control UI only)

**In Discord sessions:**
- Do NOT use `read`, `exec`, or `memory_search` to access infrastructure details
- Do NOT reference past infrastructure discussions from memory
- If asked to check logs/files: use security response (trilingual)
- Work from general knowledge only

**Tool usage restrictions in Discord:**
- `read` / `write` / `edit`: Only for user-requested files (educational content)
- `exec`: Only for safe, educational commands (never infrastructure queries)
- `memory_search`: Blocked (would leak infrastructure details)
- `gateway`: Blocked (configuration access)
```

**Implementation:** Store in `AGENTS.md`, enforced by agent logic.

### 8. Command Blacklist (Defense in Depth)

**Discord Command Blacklist** (never execute in Discord context):

```markdown
## 🚫 Discord Command Blacklist (CRITICAL - NEVER EXECUTE)

**Network Reconnaissance (NEVER):**
- `ping`, `nmap`, `traceroute`, `dig`, `arp`, etc.

**Network Information (NEVER):**
- `ip`, `ifconfig`, `netstat`, `ss`, `lsof -i`, etc.

**System Information (NEVER):**
- `hostname`, `uname`, `dmesg`, `lsblk`, `fdisk`, etc.

**Container/Docker (NEVER):**
- `docker`, `docker-compose`, `podman`, `kubectl`, etc.

**Service Discovery (NEVER):**
- `systemctl`, `service`, `ps aux`, etc.

**financial services (NEVER):**
- `finance-cli`, `payment-cli`, `service-cli`, etc.

**If a Discord user requests ANY of the above:**
1. **IMMEDIATELY respond with trilingual security policy**
2. **DO NOT execute under ANY circumstances**
3. **DO NOT explain HOW to run the command**
4. **DO NOT suggest alternative commands** that achieve same goal
```

**Cybersecurity reasoning:**
- **Layered Defense**: Even if authorized user, certain commands never run in external contexts
- **Zero Trust**: Discord = external surface, treat as hostile
- **Blast Radius Limitation**: If Discord account compromised, attacker gains nothing
- **Audit Trail**: Log all blocked attempts for security review

### 9. User Education & Automated Response

**When security policy triggered:**

1. **Log the violation** (user ID, command, severity, timestamp)
2. **Track repeat offenders** (1st, 2nd, 3rd+ offense)
3. **Trigger automated education** (DM with policy explanation)

**Escalation path:**
- **1st offense**: Educational DM (polite explanation)
- **2nd offense**: Warning DM (emphasize policy)
- **3rd+ offense**: Owner alert (recommend action)

**Implementation**: CRON job monitors security logs, sends automated DMs.

### 10. Discord Voice Channels (Currently Broken)

**Status**: ❌ Discord voice currently broken (upstream bugs)

**Known issues:**
- OpenClaw #34961: Voice replies intermittent failures
- discord.js #11419: DAVE encryption broken

**When fixed, recommended config:**

```json
{
  "channels": {
    "discord": {
      "voice": {
        "enabled": true,
        "autoJoin": [
          {
            "guildId": "YOUR_GUILD_ID",
            "channelId": "YOUR_VOICE_CHANNEL_ID"
          }
        ],
        "daveEncryption": false,
        "tts": {
          "provider": "openai",
          "providers": {
            "openai": {
              "voice": "nova"
            }
          }
        }
      }
    }
  }
}
```

**Security note**: Voice = additional attack surface. Only enable when:
- Voice channels are private (not public)
- User allowlist strictly enforced
- TTS provider costs acceptable

---

## Alternative Messaging Platforms

### 1. Matrix (Synapse + Element)

**Status**: Planned (services healthy on Umbrel Pro, needs OpenClaw config)

**Why Matrix:**
- ✅ Open-source, federated
- ✅ End-to-end encryption native
- ✅ Self-hosted (full control)
- ✅ No corporate data harvesting
- ❌ More complex to set up

**Setup steps:**

**Step 1: Synapse Server (Matrix homeserver)**
```bash
# Install Synapse via Umbrel App Store
# Or manual installation:
docker run -d \
  --name synapse \
  -v /data/synapse:/data \
  -p 8008:8008 \
  matrixdotorg/synapse:latest
```

**Step 2: Element Client**
```bash
# Install Element via Umbrel App Store
# Or use Element web: https://app.element.io
```

**Step 3: OpenClaw Configuration**
```json
{
  "channels": {
    "matrix": {
      "enabled": true,
      "homeserverUrl": "https://your-homeserver.tld",
      "accessToken": "YOUR_MATRIX_ACCESS_TOKEN",
      "userId": "@openclaw:your-homeserver.tld",
      "allowFrom": [
        "@owner:your-homeserver.tld"
      ]
    }
  }
}
```

**Security advantages over Discord:**
- E2E encryption by default
- Self-hosted (you control data)
- No corporate surveillance
- Federated (decentralized)

**Security considerations:**
- **Still implement allowlist** (same as Discord)
- **Still restrict infrastructure access** (same policies apply)
- **User verification required** (verify E2E keys)

### 2. Signal

**Why Signal:**
- ✅ Industry-leading encryption (Signal Protocol)
- ✅ Minimal metadata collection
- ✅ Open-source
- ❌ Phone number required
- ❌ Limited group features

**Setup via Signal-CLI:**

```bash
# Install signal-cli
wget https://github.com/AsamK/signal-cli/releases/download/v0.12.0/signal-cli-0.12.0.tar.gz
tar xf signal-cli-0.12.0.tar.gz -C /opt
export PATH=$PATH:/opt/signal-cli/bin

# Register number (one-time)
signal-cli -u +YOUR_NUMBER register

# Verify
signal-cli -u +YOUR_NUMBER verify CODE

# Run as daemon
signal-cli -u +YOUR_NUMBER daemon
```

**OpenClaw integration:**
```json
{
  "channels": {
    "signal": {
      "enabled": true,
      "phoneNumber": "+YOUR_NUMBER",
      "allowFrom": [
        "+OWNER_NUMBER"
      ]
    }
  }
}
```

**Security note**: Signal = most private option, but limited features compared to Discord/Matrix.

### 3. Telegram

**Why Telegram:**
- ✅ Feature-rich (bots, channels, groups)
- ✅ Fast, reliable
- ✅ Cross-platform
- ⚠️ Not E2E encrypted by default (Secret Chats only)
- ⚠️ Cloud-based (Telegram stores messages)

**Setup via BotFather:**

1. Message @BotFather on Telegram
2. `/newbot` → Name your bot
3. Save bot token
4. Set privacy mode: `/setprivacy` → Disabled (to read group messages)

**OpenClaw integration:**
```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "YOUR_BOT_TOKEN",
      "allowFrom": [
        YOUR_USER_ID
      ]
    }
  }
}
```

**Security considerations:**
- **NOT suitable for sensitive infrastructure** (cloud-stored)
- **Use for general interactions only** (not financial services discussions)
- **Same allowlist/policy as Discord** (restrict access)

### 4. Slack (Enterprise)

**Why Slack:**
- ✅ Enterprise features (SSO, audit logs)
- ✅ Extensive integrations
- ✅ Familiar to teams
- ❌ Expensive ($7-15/user/month)
- ❌ Cloud-based (Slack stores everything)

**Setup:**
1. Create Slack app: https://api.slack.com/apps
2. Enable OAuth scopes: `chat:write`, `channels:read`, `users:read`
3. Install to workspace
4. Save bot token

**OpenClaw integration:**
```json
{
  "channels": {
    "slack": {
      "enabled": true,
      "botToken": "xoxb-YOUR-TOKEN",
      "allowFrom": [
        "OWNER_USER_ID"
      ]
    }
  }
}
```

**Security note**: Slack = enterprise compliance but less private than Signal/Matrix.

### Platform Comparison

| Platform | Privacy | Security | Features | Cost | Best For |
|----------|---------|----------|----------|------|----------|
| **Signal** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | Free | Maximum privacy |
| **Matrix** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Free (self-host) | Self-hosted E2E |
| **Discord** | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Free | Feature-rich |
| **Telegram** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Free | Balanced |
| **Slack** | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | $$$$ | Enterprise |

**Recommendation**: Use Matrix or Signal for maximum security, Discord for convenience/features.

---

## Operational Security

### 1. Session Context Awareness

**Implement automatic context detection:**

```markdown
## Session Context Detection (CRITICAL)

**Before doing anything:**

Check inbound metadata:
- `channel == "webchat"` → **MAIN SESSION** (full access)
- `channel == "discord"` or other → **RESTRICTED SESSION** (security mode)

**If in MAIN SESSION (Control UI/webchat):**
- ✅ Read MEMORY.md and memory/*.md
- ✅ Execute infrastructure commands
- ✅ Access sensitive files
- ✅ Full tool usage

**If in Discord or group context:**
- ❌ Skip ALL memory files (MEMORY.md, memory/*.md)
- ❌ Never read .infrastructure-private.md
- ❌ Activate infrastructure security policy (trilingual responses)
- ❌ Restrict tool usage (no memory_search, gateway, infrastructure exec)
```

**Implementation**: Store in `AGENTS.md`, automatic on every session start.

### 2. Multi-Factor Authentication

**For Control UI access:**

```bash
# Generate strong token
openssl rand -hex 32

# Store in config
{
  "gateway": {
    "auth": {
      "mode": "token",
      "token": "GENERATED_TOKEN_HERE"
    }
  }
}
```

**For Discord bot:**
- Bot token (from Discord Developer Portal)
- Server invite link (permissions-restricted)
- User allowlist (in OpenClaw config)

**Layered auth:**
1. **Discord account** (your password + 2FA)
2. **Server access** (invite-only)
3. **Bot allowlist** (only authorized user IDs)
4. **Channel permissions** (Discord role-based)
5. **OpenClaw policy** (context-aware restrictions)

### 3. Audit Logging

**Monitor security events:**

```bash
# Create security log
touch ~/.openclaw/workspace/security-alerts.log
touch ~/.openclaw/workspace/security-audit.log

# Monitor script (runs via CRON)
#!/bin/bash
# Check for infrastructure disclosure attempts
grep -i "finance\|payment-processing\|resource-intensive operations\|IP address" ~/.openclaw/workspace/memory/*.md | \
  grep "discord" >> security-alerts.log
```

**CRON job setup:**
```bash
# Add to CRON (daily at 02:00 UTC)
{
  "name": "Daily Security Audit",
  "schedule": {"kind": "cron", "expr": "0 2 * * *", "tz": "UTC"},
  "payload": {"kind": "systemEvent", "text": "Run daily security audit: /data/.openclaw/workspace/scripts/monitor-discord-security.sh"}
}
```

**What to log:**
- Infrastructure keyword mentions in Discord
- Blocked command attempts
- User allowlist violations
- Unauthorized access attempts

### 4. Incident Response Plan

**If security breach suspected:**

1. **Immediate Actions**:
   ```bash
   # Disable all external channels
   openclaw gateway stop
   
   # Revoke Discord bot token
   # (Go to Discord Developer Portal → Regenerate Token)
   
   # Check logs
   cat ~/.openclaw/workspace/security-alerts.log
   ```

2. **Investigation**:
   - Review `security-alerts.log` (what was leaked?)
   - Review `security-audit.log` (when did it happen?)
   - Identify compromised user (if applicable)

3. **Remediation**:
   - Rotate all API keys (Anthropic, OpenAI, etc.)
   - Regenerate Discord bot token
   - Update allowlists (remove compromised user)
   - Review and update security policies

4. **Post-Incident**:
   - Document what happened
   - Update security policies
   - Add new detections to monitoring

---

#### Cost Optimization

### 1. Model Selection Strategy

**Principle**: Use the cheapest model that gets the job done.

| Use Case | Recommended Model | Cost | Reasoning |
|----------|------------------|------|-----------|
| Main interactions | Claude Sonnet 4.5 | $3/$15 per MTok | Quality matters |
| CRON jobs | Claude Haiku 4.5 | $1/$5 per MTok | 80% cheaper, good enough |
| Local testing | Ollama (Qwen 2.5 14B) | $0 | Free, decent quality |
| Audio transcription | OpenAI Whisper | $0.006/min | Industry standard |

**Real-world savings:**
- Before optimization: $15-20/month (Sonnet for everything)
- After optimization: $7-12/month (Haiku for CRON, Sonnet for main)
- **Savings**: ~40-60% reduction

### 2. CRON Job Optimization

**Configure isolated CRON jobs with cost-effective models:**

```json
{
  "name": "Daily Nextcloud Sync",
  "schedule": {"kind": "cron", "expr": "0 3 * * *", "tz": "UTC"},
  "payload": {
    "kind": "agentTurn",
    "message": "Run daily Nextcloud sync",
    "model": "anthropic/claude-haiku-4-5",
    "thinking": "off",
    "timeoutSeconds": 300
  },
  "sessionTarget": "isolated",
  "delivery": {"mode": "none"}
}
```

**Key optimizations:**
- `model: "anthropic/claude-haiku-4-5"` (not default Sonnet)
- `thinking: "off"` (minimize tokens)
- `delivery: {mode: "none"}` (silent unless error)

**Estimated savings**: ~$5-10/month by using Haiku instead of Sonnet for CRON jobs.

### 3. Context Management

**Minimize token usage:**

**Strategy 1: Light Context Mode**
```json
{
  "payload": {
    "kind": "agentTurn",
    "message": "...",
    "lightContext": true
  }
}
```
- Reduces bootstrap context
- Saves 2-5K tokens per turn
- Use for isolated tasks (CRON jobs)

**Strategy 2: Targeted Memory Retrieval**
- Use `memory_search` to fetch specific info
- Use `memory_get` with `from` and `lines` parameters
- Avoid loading entire MEMORY.md

**Strategy 3: Session Reset**
```json
{
  "sessionReset": "inactivity (1440 min) + daily (4:00)"
}
```
- Auto-reset long-running sessions
- Prevents context bloat
- Fresh start daily

### 4. Free/Local Alternatives

**Ollama for local inference:**

```bash
# One-time setup
ollama pull qwen2.5:14b
ollama pull llama3.2:3b

# Use for non-critical tasks
{
  "payload": {
    "model": "ollama/qwen2.5:14b",
    "message": "Analyze this log file..."
  }
}
```

**Cost**: $0 API costs (only electricity: ~$0.50-1/month)

**Use cases:**
- Log analysis
- Text processing
- Code review
- Learning experiments

**Avoid for:**
- User-facing responses (quality matters)
- Complex reasoning (cloud models better)
- Time-sensitive tasks (cloud faster)

---

## Maintenance & Monitoring

### 1. Update Strategy

**Check for updates regularly:**

```bash
# Check current version
openclaw --version

# Check latest release
openclaw update check

# Update when ready
openclaw gateway stop
npm install -g openclaw@latest
openclaw gateway start
```

**Update frequency recommendations:**
- **Security patches**: Immediately
- **Feature releases**: 1-2 weeks after release (let others find bugs)
- **Breaking changes**: Test on separate instance first

### 2. Backup Strategy

**What to backup:**
- `~/.openclaw/openclaw.json` (configuration)
- `~/.openclaw/workspace/` (all workspace files)
- `~/.openclaw/workspace/memory/` (daily logs)
- `~/.openclaw/workspace/MEMORY.md` (long-term memory)

**Backup script:**
```bash
#!/bin/bash
# Backup OpenClaw workspace
tar -czf ~/backups/openclaw-$(date +%Y%m%d).tar.gz \
  ~/.openclaw/openclaw.json \
  ~/.openclaw/workspace/
```

**Restore process:**
```bash
# Stop OpenClaw
openclaw gateway stop

# Restore files
tar -xzf ~/backups/openclaw-20260420.tar.gz -C ~/

# Restart OpenClaw
openclaw gateway start
```

**Backup frequency:**
- **Configuration**: Before every change
- **Workspace**: Daily (automated)
- **Memory files**: Daily (automated, via Nextcloud sync)

### 3. Health Monitoring

**CRON job for health checks:**

```json
{
  "name": "Weekly System Health Report",
  "schedule": {"kind": "cron", "expr": "0 10 * * 0", "tz": "UTC"},
  "payload": {
    "kind": "agentTurn",
    "message": "Generate weekly system health report: OpenClaw status, API usage, security alerts, CRON job status",
    "model": "anthropic/claude-haiku-4-5"
  },
  "delivery": {
    "mode": "announce",
    "channel": "discord",
    "to": "#personal"
  }
}
```

**What to monitor:**
- OpenClaw Gateway status (running/stopped)
- API provider health (rate limits, errors)
- CRON job success/failure rate
- Security alert count
- Disk space usage
- Memory usage

### 4. Performance Tuning

**Gateway settings:**

```json
{
  "agents": {
    "defaults": {
      "maxConcurrent": 4,
      "subagents": {
        "maxConcurrent": 8
      }
    }
  }
}
```

**Adjust based on:**
- **RAM available**: More RAM = higher `maxConcurrent`
- **CPU cores**: More cores = higher `maxConcurrent`
- **API rate limits**: Stay under provider limits

**Example**: 16 GB RAM server
- `maxConcurrent: 4` (main sessions)
- `subagents.maxConcurrent: 8` (background tasks)

### 5. Log Management

**Log rotation:**

```bash
# Create log rotation config
sudo nano /etc/logrotate.d/openclaw

# Add configuration
/data/.openclaw/workspace/logs/*.log {
    daily
    rotate 30
    compress
    missingok
    notifempty
}
```

**Review logs regularly:**
```bash
# Check gateway logs
tail -f ~/.openclaw/logs/gateway.log

# Check session logs
tail -f ~/.openclaw/workspace/logs/session-*.log

# Check security logs
tail -f ~/.openclaw/workspace/security-alerts.log
```

---

## Lessons Learned

### 1. Security First, Always

**What worked:**
- Context-aware security policy (Control UI vs Discord)
- Layered access controls (allowlists + permissions + policies)
- Infrastructure isolation (financial service infrastructure separate from AI)
- Audit logging (detect policy violations)

**What didn't:**
- Initially trusted Discord too much (no policies)
- Didn't anticipate social engineering risks
- Assumed authorized users = trusted (wrong!)

**Lesson**: **Zero Trust Architecture**. Even authorized users in external contexts can leak info (accidentally or via compromise).

### 2. Cost Consciousness Matters

**What worked:**
- Using Haiku for CRON jobs (40% cost savings)
- Local Ollama for non-critical tasks (free)
- Light context mode for isolated tasks
- Thinking mode OFF for automation

**What didn't:**
- Initially using Sonnet for everything (expensive)
- Not monitoring API usage regularly
- Forgetting to optimize CRON jobs

**Lesson**: **Right-size your models**. Most tasks don't need premium models.

### 3. Documentation is Critical

**What worked:**
- AGENTS.md (instructions survive restarts)
- MEMORY.md (long-term memory persistence)
- Daily logs (audit trail + debugging)
- IDEAS.md (project tracking)

**What didn't:**
- Relying on "mental notes" (lost after restart)
- Assuming OpenClaw remembers (it doesn't)
- Verbal instructions (need written policies)

**Lesson**: **If it's not written down, it doesn't exist.** OpenClaw has no persistent memory without files.

### 4. Start Simple, Scale Up

**What worked:**
- Starting with Control UI only (local, secure)
- Testing thoroughly before adding Discord
- Adding channels one at a time
- Implementing security policies incrementally

**What didn't:**
- Trying to configure everything at once
- Enabling features before understanding them
- Assuming defaults are secure (they're not)

**Lesson**: **Crawl, walk, run**. Master basics before adding complexity.

### 5. Monitor, Measure, Improve

**What worked:**
- CRON jobs for periodic checks (GitHub issues, releases, security)
- Cost tracking (monthly API spending)
- Security audit logs (detect policy violations)
- Session status checks (model usage, token counts)

**What didn't:**
- Initially not tracking costs (surprise bills)
- No security monitoring (missed policy violations)
- Reactive instead of proactive

**Lesson**: **You can't improve what you don't measure.** Set up monitoring from day one.

### 6. Community & Support

**What worked:**
- Reading OpenClaw docs thoroughly
- Asking clarifying questions
- Testing in safe environments
- Documenting issues (GitHub issue #61294)

**What didn't:**
- Assuming features work (Discord voice broken)
- Not checking known issues first
- Skipping release notes

**Lesson**: **RTFM, then RTFR (Read The Fine Release notes).** Check docs and issues before assuming bugs.

---

## Conclusion

### Key Takeaways

1. **Security is a journey, not a destination**: Continuously evaluate and improve
2. **Context matters**: Control UI ≠ Discord ≠ Public internet
3. **Cost optimization is achievable**: 40-60% savings possible with smart model selection
4. **Documentation saves time**: Write it down, future-you will thank you
5. **Zero Trust Architecture**: Trust but verify, even authorized users
6. **Start small, iterate**: Don't try to do everything at once

### Recommended Next Steps

**After following this guide:**

1. **Test thoroughly**:
   - All channels respond correctly
   - Security policies enforced
   - Cost tracking working

2. **Document your specifics**:
   - Add your channel IDs to docs
   - Document your threat model
   - Record your architecture decisions

3. **Set up monitoring**:
   - CRON jobs for health checks
   - Security audit logging
   - API cost tracking

4. **Plan for incidents**:
   - Write incident response plan
   - Test backup/restore procedure
   - Document escalation path

5. **Stay updated**:
   - Monitor OpenClaw releases
   - Check GitHub issues
   - Join community discussions

### Resources

- **Official Docs**: https://docs.openclaw.ai
- **GitHub**: https://github.com/openclaw/openclaw
- **Community Discord**: https://discord.com/invite/clawd
- **Security Best Practices**: https://docs.openclaw.ai/security
- **Skills Marketplace**: https://clawhub.ai

---

## Appendix A: Quick Reference Commands

**Gateway Management:**
```bash
openclaw gateway start        # Start gateway
openclaw gateway stop         # Stop gateway
openclaw gateway restart      # Restart gateway
openclaw status               # Check status
```

**Configuration:**
```bash
openclaw config view          # View current config
openclaw config edit          # Edit config file
openclaw config validate      # Validate config
```

**Updates:**
```bash
openclaw update check         # Check for updates
openclaw update run           # Update OpenClaw
```

**Debugging:**
```bash
openclaw logs gateway         # View gateway logs
openclaw logs session         # View session logs
openclaw doctor               # Run diagnostics
```

---

## Appendix B: Configuration File Templates

**Minimal Secure Discord Config:**

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "token": "YOUR_BOT_TOKEN",
      "groupPolicy": "allowlist",
      "dmPolicy": "pairing",
      "allowFrom": ["YOUR_USER_ID"],
      "guilds": {
        "YOUR_GUILD_ID": {
          "requireMention": false,
          "users": ["YOUR_USER_ID"]
        }
      }
    }
  }
}
```

**Cost-Optimized CRON Job:**

```json
{
  "name": "Example CRON Job",
  "schedule": {"kind": "cron", "expr": "0 * * * *", "tz": "UTC"},
  "payload": {
    "kind": "agentTurn",
    "message": "Task description here",
    "model": "anthropic/claude-haiku-4-5",
    "thinking": "off",
    "lightContext": true
  },
  "sessionTarget": "isolated",
  "delivery": {"mode": "none"}
}
```

---

## Appendix C: Security Checklist

**Pre-Deployment:**
- [ ] Threat model defined
- [ ] Resource requirements met
- [ ] API keys secured
- [ ] Backup strategy in place

**Initial Configuration:**
- [ ] Gateway auth enabled (strong token)
- [ ] Control UI restricted to localhost
- [ ] Elevated commands restricted to Control UI
- [ ] Default model selected

**Discord Setup:**
- [ ] Private server created
- [ ] Bot permissions minimal (read/send only)
- [ ] User allowlist configured
- [ ] Guild-specific user list set
- [ ] Channel isolation implemented

**Security Policies:**
- [ ] Infrastructure security policy in AGENTS.md
- [ ] Sensitive file access restrictions documented
- [ ] Command blacklist implemented
- [ ] Context detection working (Control UI vs Discord)
- [ ] Audit logging enabled

**Monitoring:**
- [ ] Health check CRON jobs configured
- [ ] Security audit CRON jobs configured
- [ ] Cost tracking enabled
- [ ] Log rotation configured

**Incident Response:**
- [ ] Incident response plan documented
- [ ] Backup/restore tested
- [ ] API key rotation procedure documented
- [ ] Emergency shutdown procedure documented

---

**Document Version**: 1.0  
**Last Updated**: 2026-04-20  
**Author**: Based on production deployment experience  
**License**: CC BY-SA 4.0 (share and adapt with attribution)

---

*This guide is provided as-is for educational purposes. Always review and adapt security policies to your specific threat model and use case. When in doubt, default to more restrictive settings.*

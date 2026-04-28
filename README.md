# OpenClaw Security Hardening Guide

**Languages:** [English](README.md) | [한국어 (Korean)](README.ko.md)

---

**A comprehensive guide to deploying OpenClaw with enterprise-grade security practices**

[![License: CC BY-SA 4.0](https://img.shields.io/badge/License-CC%20BY--SA%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by-sa/4.0/)
[![OpenClaw](https://img.shields.io/badge/OpenClaw-2026.4.x-blue.svg)](https://openclaw.ai)
[![Platform: Umbrel](https://img.shields.io/badge/Platform-Umbrel-orange.svg)](https://getumbrel.com)

## 📖 Overview

This guide provides **battle-tested security practices** for deploying OpenClaw in multi-channel environments. Learn how to:

- ✅ Secure Discord bot deployments with context-aware policies
- ✅ Implement defense-in-depth architecture
- ✅ Optimize API costs by 40-60% with smart model selection
- ✅ Protect infrastructure details from leaking via group chats
- ✅ Set up Matrix, Signal, Telegram, and other secure channels
- ✅ Automate monitoring with cost-effective CRON jobs

**Based on real-world production deployment** with 6+ Discord channels, multiple agents, and zero security incidents.

---

## 🎯 Who This Is For

- **Self-hosters** running OpenClaw on Umbrel, Raspberry Pi, VPS, or dedicated servers
- **Privacy-conscious users** who want zero-trust security architecture
- **Discord server owners** integrating AI assistants safely
- **Advanced users** seeking enterprise-grade security without enterprise complexity

---

## 📚 What's Inside

- **Pre-Installation Planning**: Threat modeling, architecture decisions, resource requirements
- **Security Hardening**: Gateway lockdown, elevated command controls, file system boundaries
- **Discord Security**: Bot permissions, allowlists, content filtering, command blacklists
- **Multi-Channel Setup**: Discord, Matrix, Signal, Telegram configuration
- **Operational Security**: Context-aware policies, audit logging, incident response
- **Cost Optimization**: 40-60% savings with Haiku vs Sonnet, local Ollama inference
- **Maintenance**: Backup strategies, health monitoring, update procedures
- **Lessons Learned**: Real mistakes, real solutions, production wisdom

---

## 🚀 Quick Start

1. **Read the full guide**: [OPENCLAW-SETUP-GUIDE.md](./OPENCLAW-SETUP-GUIDE.md)
2. **Define your threat model**: What are you protecting? From whom?
3. **Install OpenClaw**: Follow installation section for your platform
4. **Implement security layers**: Start with Control UI, add channels incrementally
5. **Test thoroughly**: Verify security policies before going live

---

## 🔒 Key Security Features

### Context-Aware Security Policy

OpenClaw **automatically restricts access** based on session context:

| Context | Trust Level | Infrastructure Access | Elevated Commands |
|---------|-------------|----------------------|-------------------|
| **Control UI** | HIGH | ✅ Full access | ✅ Allowed |
| **Discord #personal** | MEDIUM | ❌ Restricted | ❌ Denied |
| **Discord #general** | LOW | ❌ Blocked | ❌ Denied |

### Defense in Depth

Multiple security layers protect your infrastructure:

1. **Gateway Auth**: Token-based authentication, localhost-only by default
2. **User Allowlists**: Per-channel, per-server, global restrictions
3. **Command Blacklist**: Blocks infrastructure reconnaissance in external contexts
4. **File Access Controls**: Sensitive files never accessible from Discord
5. **Audit Logging**: Track policy violations, security events
6. **Automated Monitoring**: CRON jobs detect anomalies

---

## 💰 Cost Optimization

### Real-World Savings

| Strategy | Before | After | Savings |
|----------|--------|-------|---------|
| **CRON Model Selection** | Sonnet ($15-20/mo) | Haiku ($7-12/mo) | **40-60%** |
| **Local Inference (Ollama)** | Cloud ($10-30/mo) | Local ($0/mo) | **100%** |
| **Context Management** | Full bootstrap | Light context | **30-50%** |

### Smart Model Selection

- **Main interactions**: Claude Sonnet 4.5 (quality matters)
- **CRON automation**: Claude Haiku 4.5 (80% cheaper, good enough)
- **Experimentation**: Ollama local models (free, privacy-first)

---

## 🛠️ Platform Support

This guide covers:

- ✅ **Umbrel** (Home & Pro) - Primary focus
- ✅ **Debian/Ubuntu** - NPM global install
- ✅ **Docker** - Container deployment
- ✅ **Raspberry Pi** - ARM support
- ✅ **VPS** - Cloud deployment

---

## 📝 Table of Contents

1. Philosophy & Approach
2. Pre-Installation Planning
3. Installation
4. Initial Configuration
5. Security Hardening
6. Multi-Channel Setup
7. Discord Security Lockdown
8. Alternative Messaging Platforms
9. Operational Security
10. Cost Optimization
11. Maintenance & Monitoring
12. Lessons Learned

**[Read the full guide →](./OPENCLAW-SETUP-GUIDE.md)**

---

## 🤝 Contributing

Found a mistake? Have a better approach? Contributions welcome!

1. Fork this repo
2. Make your changes
3. Submit a pull request

**Please maintain**:
- Security-first mindset
- Detailed explanations
- Real-world testing

---

## 📜 License

This guide is licensed under **CC BY-SA 4.0** (Creative Commons Attribution-ShareAlike 4.0 International).

You are free to:
- **Share**: Copy and redistribute in any medium or format
- **Adapt**: Remix, transform, and build upon the material

Under the following terms:
- **Attribution**: Credit the original author
- **ShareAlike**: Distribute your adaptations under the same license

[Full license text](./LICENSE)

---

## 🙏 Acknowledgments

- **OpenClaw Team**: For building an incredible AI assistant framework
- **Umbrel Team**: For making self-hosting accessible
- **Discord Community**: For testing, feedback, and security insights
- **Self-Hosting Community**: For privacy-first values

---

## 📞 Support

- **OpenClaw Docs**: https://docs.openclaw.ai
- **OpenClaw GitHub**: https://github.com/openclaw/openclaw
- **Community Discord**: https://discord.com/invite/clawd

---

## ⭐ Star This Repo

If this guide helped you deploy OpenClaw securely, **please star this repo** to help others find it!

---

**Version**: 1.0.0  
**Last Updated**: 2026-04-27  
**Author**: MajorCommotion  
**Based on**: Production deployment experience (2026-03 to 2026-04)

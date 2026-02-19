---
name: perplexity-pro-openclaw
description: Connect Perplexity PRO to OpenClaw with anti-bot browser automation, bypassing Cloudflare protection via Xvfb and VNC authentication
homepage: https://github.com/Hantok/Perplexity-PRO-to-OpenClaw
metadata:
  clawdbot:
    emoji: "🔍"
  requires:
    env: ["DISPLAY"]
  files: ["scripts/*"]
version: 1.0.0
---

# Perplexity PRO for OpenClaw

This skill enables OpenClaw to search Perplexity PRO with persistent authenticated sessions, bypassing Cloudflare protection through undetectable browser automation.

## What This Skill Does

- ✅ Bypasses Cloudflare bot detection (Xvfb + stealth Chrome)
- ✅ Maintains persistent Perplexity PRO sessions across reboots
- ✅ Provides VNC access for manual OAuth authentication
- ✅ Creates automated search capability for OpenClaw agents

## Prerequisites

- Ubuntu Server (headless or with display)
- OpenClaw installed and configured
- Google Chrome (NOT Snap version)
- Xvfb package
- x11vnc for remote access

## Interactive Setup Steps

During installation, the agent will guide you through:

### Step 1: Chrome Installation
The agent will help you remove Snap Chromium and install proper Google Chrome:
```bash
sudo snap remove chromium
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
```

### Step 2: Browser Launcher Setup
The agent creates `start-stealth-browser.sh` with anti-bot settings:
- Xvfb virtual display (bypasses headless detection)
- Persistent profile in `~/.openclaw/browser-profile/`
- Stealth flags to mask automation

### Step 3: VNC Configuration
The agent sets up x11vnc for remote browser access:
```bash
sudo apt-get install -y x11vnc
x11vnc -storepasswd openclaw /tmp/vncpass
```

**⚠️ Security Note:** Default VNC password is "openclaw" - change this in production!

### Step 4: Manual Authentication (Required)
**You must authenticate manually through VNC:**

1. Connect via VNC (macOS: `open vnc://your-server:5900`, Windows: RealVNC Viewer)
2. Open Perplexity.ai in the browser
3. Click "Sign in with Google"
4. Enter your email and password directly
5. Complete 2FA if enabled

**Important:** App Passwords do NOT work for web authentication. Use your actual Google password.

## How to Use

### Quick Automated Search (No Login Required)

For basic queries without authentication:

```bash
# 1. Start the stealth browser
./scripts/start-stealth-browser.sh

# 2. Open Perplexity with openclaw profile
openclaw browser open --profile openclaw https://www.perplexity.ai

# 3. Close any auth modal that appears
openclaw browser click --ref <close-button>

# 4. Click the search textbox, type query, submit
openclaw browser click --ref <textbox>
openclaw browser type --ref <textbox> "your question"
openclaw browser click --ref <submit-button>

# 5. Wait 2-3 seconds, then capture results
openclaw browser wait --time 3000
openclaw browser snapshot
```

**Important:** Use `--profile openclaw` not `--profile chrome` — the chrome profile requires manual extension attachment.

### URL-Based Search (Faster)

```bash
openclaw browser open "https://www.perplexity.ai/search?q=your+query+here"
```

### Authenticated Search (Full PRO Features)

After completing VNC authentication (Step 4 above):

```bash
./scripts/start-stealth-browser.sh
openclaw browser open "https://www.perplexity.ai/search?q=your+query"
```

Your session persists via cookies in `~/.openclaw/browser-profile/`.

## File Structure

```
perplexity-pro-openclaw/
├── SKILL.md          # This file - skill metadata and setup
├── README.md         # Detailed installation guide
├── scripts/
│   └── start-stealth-browser.sh  # Browser launcher
└── CHANGELOG.md      # Version history
```

## Anti-Bot Techniques (7 Levels)

1. **Browser Fingerprint Masking** - `--disable-blink-features=AutomationControlled`
2. **Session Persistence** - Profile in `~/.openclaw/browser-profile/` (not /tmp)
3. **Xvfb Virtual Display** - Real Chrome window, not headless
4. **Realistic Viewport** - 1920x1080 resolution
5. **Disabled Automation Flags** - Background throttling disabled
6. **Persistent Cookies** - Survives browser restarts
7. **FlareSolverr** (optional) - For extreme cases

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Chrome extension relay error | Use `--profile openclaw` instead of `--profile chrome`. Chrome profile requires manual extension attachment. |
| Auth modal blocking interaction | Click Close button immediately. Don't automate OAuth flows — they trigger anti-bot measures. |
| Query doesn't submit | Ensure you: 1) Click textbox to focus, 2) Type query, 3) Click Submit button, 4) Wait 2-3 seconds |
| Chrome shows "HeadlessChrome" | Ensure Xvfb is running, not `--headless` flag |
| Cloudflare blocking | Close tab and retry with fresh session. Don't retry same request. |
| VNC connection refused | Check `ss -tlnp \| grep 5900`, verify firewall |
| Perplexity asks to login again | Profile directory issue - re-authenticate via VNC once |

## Verified Working

| Date | Test | Result |
|------|------|--------|
| 2025-02-20 | Unauthenticated Russian query | ✅ Success |
| 2025-02-20 | Cloudflare bypass | ✅ Success |

## Security Considerations

- VNC password should be changed from default "openclaw"
- Profile stored in `~/.openclaw/browser-profile/` (user home, not /tmp)
- OAuth tokens persist - secure your server appropriately
- VNC traffic is unencrypted by default - use SSH tunnel for remote access

## Author

Created by [rundax.com](https://rundax.com)

Part of the OpenClaw ecosystem - [ClawHub](https://clawhub.com)

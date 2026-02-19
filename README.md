# Perplexity PRO for OpenClaw

🔍 **Undetectable Perplexity PRO integration for OpenClaw with persistent sessions**

This skill enables OpenClaw agents to search Perplexity PRO while bypassing Cloudflare protection through stealth browser automation.

## ⚠️ What the Agent Will Ask You To Do

**Before installing, understand that this skill requires:**

1. **System modifications:** Installing Google Chrome, Xvfb, x11vnc
2. **Manual OAuth authentication:** You must log into Perplexity via VNC (cannot be automated)
3. **VNC access:** Remote desktop to the server for authentication
4. **Time investment:** ~15-20 minutes for initial setup

If you're not comfortable with these steps, do not proceed with installation.

## Quick Start

```bash
# Install the skill
clawdhub install perplexity-pro-openclaw

# Or manually clone
git clone https://github.com/Hantok/Perplexity-PRO-to-OpenClaw.git
```

## Detailed Installation

### Step 1: Install Google Chrome

**⚠️ Remove Snap Chromium first (it doesn't work with this setup):**

```bash
sudo snap remove chromium

# Download and install proper Chrome
cd /tmp
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt-get install -f -y

# Verify
google-chrome --version
```

### Step 2: Install Dependencies

```bash
sudo apt-get update
sudo apt-get install -y Xvfb x11vnc
```

### Step 3: Configure Browser Launcher

The skill creates `~/.openclaw/workspace/start-stealth-browser.sh`:

```bash
#!/bin/bash
# OpenClaw Anti-Bot Browser Launcher
# Levels 1-7: Xvfb + Chrome with stealth settings
# Session Persistence: OAuth sessions survive browser restarts

# Check if Chrome is already running with our profile
if curl -s http://127.0.0.1:18800/json/version > /dev/null 2>&1; then
    echo "✅ Chrome already running with CDP on port 18800"
    curl -s http://127.0.0.1:18800/json/version | grep -o '"User-Agent": "[^"]*"'
    exit 0
fi

echo "Starting new Chrome instance..."

# Kill only Xvfb (not Chrome) to ensure clean display
pkill -f "Xvfb :99" 2>/dev/null
sleep 1

# Create persistent user data directory (not in /tmp!)
mkdir -p ~/.openclaw/browser-profile

# Start Xvfb (virtual display)
export DISPLAY=:99
Xvfb :99 -screen 0 1920x1080x24 -ac +extension GLX +render -noreset &
XVFB_PID=$!
echo "Xvfb started on DISPLAY :99 (PID: $XVFB_PID)"
sleep 2

# Start Chrome with anti-bot settings + session persistence
# Level 1: Browser Fingerprint Masking
# Level 2: Session/Cookie Reuse (user-data-dir)
# Level 3: Xvfb instead of --headless (undetectable)
# Level 4: Session persistence flags for OAuth
DISPLAY=:99 google-chrome \
    --no-sandbox \
    --disable-setuid-sandbox \
    --disable-dev-shm-usage \
    --disable-gpu \
    --disable-software-rasterizer \
    --remote-debugging-port=18800 \
    --user-data-dir="$HOME/.openclaw/browser-profile" \
    --window-size=1920,1080 \
    --force-color-profile=srgb \
    --disable-blink-features=AutomationControlled \
    --disable-features=IsolateOrigins,site-per-process,Translate \
    --no-first-run \
    --no-default-browser-check \
    --password-store=basic \
    --use-mock-keychain \
    --enable-features=NetworkService,NetworkServiceInProcess \
    --disable-background-timer-throttling \
    --disable-backgrounding-occluded-windows \
    --disable-renderer-backgrounding \
    --disable-background-networking \
    --enable-automation=false \
    --restore-last-session \
    --persist-session-cookies \
    --keep-alive-for-test \
    > /tmp/chrome_stealth.log 2>&1 &

CHROME_PID=$!
echo "Chrome started (PID: $CHROME_PID)"
echo "Profile: ~/.openclaw/browser-profile (persistent)"
echo "CDP URL: http://127.0.0.1:18800"

# Wait for Chrome to be ready
for i in {1..10}; do
    sleep 1
    if curl -s http://127.0.0.1:18800/json/version > /dev/null 2>&1; then
        echo "✅ Chrome ready!"
        curl -s http://127.0.0.1:18800/json/version | grep -o '"User-Agent": "[^"]*"'
        exit 0
    fi
done

echo "❌ Chrome failed to start"
cat /tmp/chrome_stealth.log | tail -20
exit 1
```

**Key Features:**
- ✅ Checks if Chrome is already running (preserves sessions)
- ✅ Won't kill existing Chrome instances
- ✅ `--restore-last-session` restores previous tabs and auth
- ✅ `--persist-session-cookies` keeps OAuth tokens valid
- ✅ Profile stored in `~/.openclaw/browser-profile/` (survives reboots)

### Step 4: Start VNC Server

```bash
# Set VNC password (default: openclaw - CHANGE THIS!)
x11vnc -storepasswd openclaw /tmp/vncpass

# Start VNC
x11vnc -display :99 -rfbauth /tmp/vncpass -listen 0.0.0.0 -xkb -forever -shared
```

**VNC Details:**
- **Address:** `your-server-ip:5900`
- **Password:** `openclaw` (default - change recommended)

### Step 5: CRITICAL - Manual Authentication

**You MUST authenticate through VNC:**

#### macOS Users:
```bash
open vnc://your-server-ip:5900
```
Or use Finder → Cmd+K → `vnc://your-server-ip:5900`

#### Windows Users:
1. Download [RealVNC Viewer](https://www.realvnc.com/en/connect/download/viewer/)
2. Enter: `your-server-ip:5900`
3. Password: `openclaw`

#### Authentication Steps:
1. Connect to VNC
2. See Chrome with Perplexity
3. Click "Sign in with Google"
4. **Use your actual Google password** (not App Password)
5. Complete 2FA if enabled
6. ✅ Session persists permanently!

## Session Persistence

**Perplexity authentication persists like a normal browser:**

### How It Works
- **Persistent Profile**: All data stored in `~/.openclaw/browser-profile/` (on disk, not /tmp)
- **Session Restore**: Chrome restores tabs and OAuth tokens on restart
- **OAuth Refresh Tokens**: Google authentication valid for months
- **No Daily Re-auth**: Unlike the old script, sessions survive indefinitely

### Session Survives:
- ✅ **Browser restart** — OAuth tokens restore automatically
- ✅ **Machine reboot** — Profile is on disk, persists across reboots
- ✅ **Days/weeks of inactivity** — Refresh tokens keep session valid

### Only Re-authenticate If:
- ❌ You manually log out of Perplexity
- ❌ You clear browser cookies/data
- ❌ Google forces security verification (rare)
- ❌ Profile directory `~/.openclaw/browser-profile/` is deleted

### Keeping Chrome Running

```bash
# Check if Chrome is running
curl -s http://127.0.0.1:18800/json/version > /dev/null && echo "✅ Chrome running" || echo "❌ Chrome stopped"

# Restart Chrome (preserves session — won't kill existing instance)
~/.openclaw/workspace/start-stealth-browser.sh
```

## Usage

### Automated Search (No Authentication Required)

For basic queries, you can use Perplexity without signing in:

```bash
# Start browser
~/.openclaw/workspace/start-stealth-browser.sh
```

Then use the OpenClaw browser tool with the **openclaw** profile:

```
# 1. Open Perplexity
browser open --profile openclaw https://www.perplexity.ai

# 2. Close the auth modal (if shown)
browser click --ref <close-button-ref>

# 3. Click the search textbox
browser click --ref <textbox-ref>

# 4. Type your query
browser type --ref <textbox-ref> "your search query"

# 5. Click Submit button
browser click --ref <submit-button-ref>

# 6. Wait 2-3 seconds for results
browser wait --time 3000

# 7. Capture results
browser snapshot
```

**Key Points for Automation:**
- Use `--profile openclaw` (not `chrome`) — Chrome profile requires manual extension attachment
- Close auth modals immediately — don't attempt automated login flows
- Wait after submitting — Perplexity needs 2-3 seconds to generate answers
- If Cloudflare appears, close tab and retry with fresh session

### URL-Based Search (Quick Method)

For simple queries, you can also use:
```bash
openclaw browser open "https://www.perplexity.ai/search?q=your+query+here"
```

## How It Works

### Anti-Bot Strategy

| Level | Technique | Purpose |
|-------|-----------|---------|
| 1 | `--disable-blink-features=AutomationControlled` | Remove webdriver flag |
| 2 | `~/.openclaw/browser-profile` | Persistent cookies (survives reboot) |
| 3 | Xvfb virtual display | Real window, not headless |
| 4 | 1920x1080 viewport | Realistic resolution |
| 5 | Multiple stealth flags | Disable automation signals |
| 6 | Cookie persistence | Auth survives browser restarts |
| 7 | FlareSolverr (optional) | Extreme bypass |

### Why Xvfb + VNC?

- `--headless` flag is easily detected by Cloudflare
- Xvfb creates real X11 display
- Chrome thinks it's running normally
- VNC lets you interact for OAuth flows

## File Structure

```
.
├── SKILL.md              # Skill metadata (ClawHub)
├── README.md             # This file
├── scripts/
│   └── start-stealth-browser.sh
└── CHANGELOG.md
```

## Security Notes

⚠️ **Important:**
- Default VNC password is "openclaw" - change in production
- Profile stored in `~/.openclaw/browser-profile/` (persistent)
- OAuth tokens survive reboots - secure your server
- VNC is unencrypted - use SSH tunnel for remote access

## Troubleshooting

### Chrome extension relay not connected
**Error:** "Chrome extension relay is running, but no tab is connected"

**Fix:** Use `--profile openclaw` instead of `--profile chrome`. The chrome profile requires manual extension attachment via the browser toolbar icon.

### Auth modal blocks interaction
**Problem:** Login/signup modal appears on page load

**Fix:** Click the Close button (X) immediately. Don't attempt automated OAuth flows — they trigger anti-bot measures. For authenticated searches, use VNC to sign in once, then cookies persist.

### Query doesn't submit
**Problem:** Text entered but no response generated

**Fix:** Ensure you:
1. Click the textbox first (to focus it)
2. Type the full query
3. Click the Submit button (not just pressing Enter)
4. Wait 2-3 seconds for processing

### Chrome shows "HeadlessChrome"
**Fix:** Ensure Xvfb is running before Chrome, not `--headless` flag

### Cloudflare still blocking
**Fix:** Update Chrome, verify all anti-bot flags present. If blocked, close the tab and open a fresh session — don't retry the same request.

### VNC won't connect
**Fix:** Check `ss -tlnp | grep 5900`, verify firewall rules

### Perplexity asks to log in again
**Cause:** Old script killed Chrome on restart, wiping OAuth tokens

**Fix:** 
1. Use the updated `start-stealth-browser.sh` (checks for running Chrome)
2. Re-authenticate once via VNC
3. Session will now persist across restarts

```bash
# Verify Chrome is running
curl -s http://127.0.0.1:18800/json/version > /dev/null && echo "✅ Chrome running"

# If stopped, restart (session will restore)
~/.openclaw/workspace/start-stealth-browser.sh
```

## Requirements

- Ubuntu 20.04+ (headless OK)
- 2GB+ RAM (Chrome + Xvfb)
- OpenClaw 2026.2.17+
- Google Chrome (not Chromium Snap)

## Verified Working

| Date | Test | Result | Notes |
|------|------|--------|-------|
| 2025-02-20 | Text query | ✅ Success | No auth required; 2-3s response time |
| 2025-02-20 | Cloudflare bypass | ✅ Success | Xvfb + stealth flags working |

**Test Query Used:**
> Согласно Википедии, однотипные изображения на НЕМ могут символизировать раны Иисуса Христа, которые он получил во время распятия. Назовите ЕГО двумя словами.

**Answer:** стигматы Христа (stigmata of Christ)

## Author

Created by [rundax.com](https://rundax.com) for the OpenClaw community.

Part of [ClawHub](https://clawhub.com) - the OpenClaw skill registry.

## License

MIT License - See LICENSE file for details.

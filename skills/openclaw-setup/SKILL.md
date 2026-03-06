---
name: openclaw-setup
description: "Complete setup, repair, and remote configuration for OpenClaw using Tailscale and Discord. ALWAYS use this skill if the user reports connectivity issues from remote devices (iPhone, laptops), errors like 'Pairing Required', 'Host Not Allowed', or 'Gateway Token Mismatch'. It covers switching to the 'coding' profile for bash access, installing core binaries (rg, ffmpeg, tmux) via Homebrew, and fixing Tailscale 'serve' failures. Use this even if the user doesn't mention setup explicitly but is struggling to make OpenClaw work over a network or via Discord."
---

# OpenClaw Setup & Configuration

A comprehensive guide for setting up OpenClaw as a powerful developer assistant accessible via Tailscale and Discord.

## 1. Prerequisites & Doctor Check
Always start by ensuring the system is in a healthy state and Node.js (v22+) is available.

- Run `openclaw doctor` to diagnose issues.
- If issues are found, run `openclaw doctor --repair` to fix configuration and gateway service mismatches.
- Ensure the gateway is running: `openclaw gateway status`.

## 2. Developer Access (Coding Profile)
OpenClaw's default "messaging" profile is restricted. To allow command execution, file system access, and advanced skills, the "coding" profile must be enabled.

- Set the profile: `openclaw config set tools.profile coding`
- Restart the gateway to apply: `openclaw gateway restart`

## 3. Core Dependencies
Many OpenClaw skills require external binaries. Install the most common ones via Homebrew:

- `brew install ripgrep ffmpeg tmux`
- Verify with `openclaw skills check` to see newly enabled skills.

## 4. Remote Access with Tailscale
To access OpenClaw securely from outside (e.g., iPhone, other laptops), use Tailscale's `serve` mode.

- **Setup Tailscale:**
  - Install: `brew install --cask tailscale`
  - Start app: `open /Applications/Tailscale.app`
  - Log in manually.
- **Configure OpenClaw for Tailscale:**
  - `openclaw config set gateway.tailscale.mode serve`
  - Restart: `openclaw gateway restart`
- **Manual Fix for 'Serve' (if logs show "serve failed"):**
  - Sometimes the automated serve fails. Run it manually: `tailscale serve --bg --yes 18789`
  - Verify: `tailscale serve status`

## 5. Dashboard & Control UI Security
If accessing from a remote device, you must allow the origin.

- Identify the remote device's Tailscale hostname and IP (e.g., `iphone.tailnet.ts.net`, `100.x.y.z`).
- Update allowed origins:
  - `openclaw config set gateway.controlUi.allowedOrigins '["http://127.0.0.1:18789", "https://your-mac-hostname.tailnet.ts.net", "http://your-phone-hostname.tailnet.ts.net"]'`
- Restart the gateway: `openclaw gateway restart`

## 6. Pairing & Device Approval
When connecting from a new device, the dashboard may say "Pairing Required".

- List pending requests: `openclaw devices list`
- Approve the request: `openclaw devices approve <REQUEST_ID>`

## 7. Discord Channel Setup
- Ensure the Discord token is configured in `openclaw.json`.
- Verify status: `openclaw status --deep`
- If the channel is not starting, use `openclaw gateway call channel_start --id discord` (if supported by the CLI) or restart the gateway.

## Troubleshooting
- **Logs:** Check `/tmp/openclaw/openclaw-YYYY-MM-DD.log` for errors.
- **Service:** If the LaunchAgent fails, try `openclaw gateway stop` and `openclaw gateway start`.
- **Token:** The gateway auth token can be found in `~/.openclaw/openclaw.json` under `gateway.auth.token`.

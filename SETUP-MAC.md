# Mac setup checklist

Condensed, ordered runbook for getting the bot answering Slack DMs on a Mac.
The full guide lives in `index.html` (GitHub Pages) — this is the minimal path,
in the order that actually works. Found the hard way on a fresh machine.

## 0. Prerequisites

- **Claude Code CLI ≥ 2.x**, installed natively and logged in **as the same
  macOS user that will run the bot**:

  ```bash
  curl -fsSL https://claude.ai/install.sh | bash
  claude --version          # needs --effort support (2.x)
  claude -p "hello"         # must reply without prompting
  ```

- Python 3.11+ (`python3 --version`), git, and Homebrew.

## 1. Slack app

1. [api.slack.com/apps](https://api.slack.com/apps) → Create New App →
   **From an app manifest** → paste the manifest from the setup guide
   (`index.html`, Phase 2). Replace the bot name; the request URL placeholder
   is fine for now.
2. Basic Information → copy the **Signing Secret**.
3. OAuth & Permissions → Install to Workspace → copy the **Bot Token**
   (`xoxb-...`).
4. **Do not configure Event Subscriptions yet** — Slack verifies the URL the
   moment you enter it, and nothing is running yet. Step 6 below.

## 2. Clone and install

```bash
mkdir -p ~/Projects
cd ~/Projects
git clone <your-fork-url> slack-bot
cd slack-bot
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
```

Run the bot from `~/Projects/` — macOS blocks launchd from Documents/Desktop.

## 3. Fill .env

Required:

| Var | Value |
|---|---|
| `SLACK_BOT_TOKEN` | `xoxb-...` from step 1 |
| `SLACK_SIGNING_SECRET` | from step 1 |
| `PROJECT_DIR` | absolute path to the codebase the bot should work in |
| `AUTHORIZED_USERS` | **your** Slack member ID — leave it empty and every workspace member can use the bot |
| `BOT_USER_ID` | the bot's own member ID — get it now that you have a token: |

```bash
curl -s -H "Authorization: Bearer xoxb-..." https://slack.com/api/auth.test
# → the "user_id" field (starts with U). Without it, @mentions get duplicate replies.
```

Optional but worth knowing: `CLAUDE_MODEL` / `CLAUDE_EFFORT` override what the
bot passes to the `claude` CLI (see `.env.example` for defaults).

## 4. Prepare PROJECT_DIR

```bash
cp CLAUDE.md.example "$PROJECT_DIR/CLAUDE.md"   # then fill in the placeholders
mkdir -p "$PROJECT_DIR/.claude"
```

Create `$PROJECT_DIR/.claude/settings.json` containing at least:

```json
{ "skipDangerousModePermissionPrompt": true }
```

Without this, supervisor messages hang forever: the bot launches `claude`
with `--permission-mode bypassPermissions`, which asks a confirmation question
no one can answer in headless mode.

## 5. Tunnel + first boot

```bash
brew install cloudflared
# full tunnel setup is in the guide; for a quick trial ngrok also works:
#   ngrok http 3000

cd ~/Projects/slack-bot
source venv/bin/activate
python bot.py            # → "Bot starting on port 3000"
curl localhost:3000/health
```

Always start the bot **from the repo directory** — `.env` is loaded relative
to the current working directory.

## 6. Wire Slack (only now)

Slack app → **Event Subscriptions** → Enable → Request URL
`https://<your-tunnel>/slack/events` → wait for the green **Verified** badge →
Subscribe to bot events `app_mention` and `message.im` → Save.
Set **Interactivity & Shortcuts** to the same URL if you want button votes to work.

## 7. Test

DM the bot. 👀 reaction should appear, then a reply.

**If 👀 appears but no reply ever comes:** the `claude` child process died.
Check `bot.log` — the bot logs the path of each spawn's stderr file and dumps
its tail on failure. Usual causes: CLI not logged in, model not available to
your account, missing `skipDangerousModePermissionPrompt`.

## Known rough edges (skip on day one)

- `plugins/chief-of-staff` — its workflow hardcodes the upstream maintainer's
  paths and usernames (`briefing-triage.js`), and its email flow needs a
  separate `gws` CLI. Needs a rewrite before it works for anyone else.
- `misc/file-explorer` — ships the maintainer's Tailscale IPs in
  `server.py` (`TAILSCALE_USERS`); edit before use.
- `search/` — standalone tool with its own venv and index step; not needed
  for the DM loop.
- `jobs/` — launchd examples; wire up after the bot itself is stable.
- `trust-battery/` — off unless you set `TRUST_BATTERY_DIR`.

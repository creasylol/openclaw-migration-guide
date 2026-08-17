# Switching from OpenClaw to Hermes Agent — A Complete, No-Assumptions Guide

This guide walks you from "I have OpenClaw running" to "I have Hermes Agent running with everything imported" — assuming **zero** prior command-line experience. Every command is explained before you run it. If you already know what a terminal is and how to install things, skip straight to [Step 3](#step-3-back-up-your-openclaw-setup-dont-skip-this).

Written for anyone switching their agent over, by someone who had to learn all of this from scratch too.

---

## Table of contents

1. [Why people are switching](#1-why-people-are-switching)
2. [Before you start: what you need](#2-before-you-start-what-you-need)
3. [Back up your OpenClaw setup](#step-3-back-up-your-openclaw-setup-dont-skip-this)
4. [Install Hermes Agent](#step-4-install-hermes-agent)
5. [Verify the install](#step-5-verify-the-install)
6. [Run the migration](#step-6-run-the-migration)
7. [What actually gets migrated](#7-what-actually-gets-migrated-the-full-map)
8. [What does NOT get migrated automatically](#8-what-does-not-get-migrated-automatically)
9. [Post-migration checklist](#step-9-post-migration-checklist-do-all-of-these)
10. [Troubleshooting](#10-troubleshooting)
11. [Cleaning up](#11-cleaning-up-once-youre-confident)
12. [Rolling back](#12-rolling-back-if-something-goes-wrong)

---

## 1. Why people are switching

OpenClaw is a mature, feature-rich agent framework — broad platform support, a big skills marketplace, and enterprise-grade infrastructure. Hermes Agent, built by Nous Research, takes a different approach: instead of starting fresh every session, it runs a learning loop after every task that extracts reusable patterns and builds a persistent model of you over time. It also tends to have a noticeably faster path from "installed" to "actually working."

Neither is strictly better — OpenClaw still has the edge for complex multi-agent orchestration and very deep customization. This guide isn't trying to convince you; it assumes you've already decided to move and just want it done correctly, with nothing missed.

---

## 2. Before you start: what you need

You need exactly one thing installed before the Hermes installer can do the rest: **Git**.

### Check if you already have it

Open a terminal:
- **Mac**: press `Cmd + Space`, type "Terminal", hit Enter
- **Linux**: use whatever terminal app you normally use
- **Windows**: Hermes does **not** run natively on Windows — you must use WSL2 (Windows Subsystem for Linux). See the box below before doing anything else.

Type this and press Enter:
```bash
git --version
```
If you see something like `git version 2.45.0`, you're set. If you see "command not found," install it:
- **Mac**: type `git --version` again — macOS will usually offer to install the Xcode Command Line Tools for you; accept that.
- **Linux (Debian/Ubuntu)**: `sudo apt install git`

You do **not** need Node.js or Python installed yourself — the Hermes installer pulls in Node.js and Python 3.11 automatically as part of setup.

> **Windows users — read this first:** Native Windows is not supported by Hermes. You need WSL2.
> 1. Open PowerShell **as Administrator** (right-click the Start button → "Windows Terminal (Admin)" or search "PowerShell" → right-click → "Run as administrator").
> 2. Run: `wsl --install`
> 3. Restart your computer when prompted.
> 4. After restart, a Linux terminal (Ubuntu by default) will open and ask you to create a username and password for it — this is separate from your Windows login, make up anything you'll remember.
> 5. From now on, every command in this guide runs **inside that Ubuntu/WSL2 window**, not in regular PowerShell.

---

## Step 3: Back up your OpenClaw setup (don't skip this)

The migration tool is careful and takes its own backup automatically, but a second, manual backup costs you thirty seconds and means you can never lose anything no matter what goes wrong.

Your OpenClaw data lives in one of these folders (the migration tool checks all three, in this order):
```
~/.openclaw/      (current)
~/.clawdbot/      (older name)
~/.moltbot/       (oldest name)
```

Back it up with one command:
```bash
cp -r ~/.openclaw ~/openclaw-backup-$(date +%Y%m%d)
```
(Swap `.openclaw` for `.clawdbot` or `.moltbot` if that's what you have.) This creates a dated copy — e.g. `~/openclaw-backup-20260817` — that nothing in this guide will ever touch.

---

## Step 4: Install Hermes Agent

Run the official one-line installer (works on macOS, Linux, and WSL2):

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

What this actually does, in plain terms: it downloads a script from Nous Research's official GitHub repo and runs it. That script installs Python 3.11+ and Node.js for you if you don't already have them, downloads Hermes itself, and adds the `hermes` command to your system.

It'll print a lot of output while it works — that's normal. Let it finish.

**Reload your shell** so the new `hermes` command is recognized in your current terminal window (you only need to do this once — new terminal windows will pick it up automatically from now on):
```bash
source ~/.bashrc
```
If you use zsh instead of bash (macOS defaults to zsh on newer versions), use this instead:
```bash
source ~/.zshrc
```
Not sure which you have? Just run both — the one that doesn't apply will do nothing.

---

## Step 5: Verify the install

```bash
hermes --version
```
You should see a version number (e.g. `hermes 0.18.2`). If you see "command not found," the shell reload above didn't take — close the terminal window entirely and open a fresh one, then try again.

Run the built-in health check:
```bash
hermes doctor
```
This checks your install for missing dependencies or misconfiguration and tells you exactly what's wrong if anything is. Fix anything it flags before moving on.

---

## Step 6: Run the migration

This is the actual OpenClaw → Hermes transfer. Hermes ships a dedicated command for this: `hermes claw migrate`.

### 6a. Preview first — always

Run a dry run before touching anything for real:
```bash
hermes claw migrate --dry-run
```
This shows you exactly what would be imported — persona, memories, skills, config, secrets — without changing a single file. Read the output carefully. Two things matter most:
- The count of migrated **memory entries** should roughly match what you'd expect. If it says "0 entries," you likely have a file-permissions problem on the OpenClaw side that's hiding the data — fix that before continuing.
- Any "would skip" lines flag **skill conflicts** the tool can't resolve automatically — you'll want to look at those individually later.

### 6b. Run the real migration

Once the dry run looks right, run it for real. The command always shows the same preview first, then asks you to confirm before changing anything:

```bash
hermes claw migrate
```

By default this does **not** copy API keys or secrets — that's intentional, so nothing sensitive moves without you explicitly asking for it. If you want a full migration including your provider API keys, bot tokens, etc., in one go:

```bash
hermes claw migrate --preset full --migrate-secrets --yes
```

Useful flags, in plain terms:

| Flag | What it does |
|---|---|
| `--dry-run` | Preview only, changes nothing |
| `--preset full` | Migrate all compatible settings (still excludes secrets unless you add `--migrate-secrets`) |
| `--preset user-data` | Migrate persona/memory/skills but skip infrastructure config |
| `--migrate-secrets` | Also copy API keys and tokens — required explicitly, no preset does this silently |
| `--overwrite` | Overwrite existing Hermes files if there's a conflict (default: it refuses when there's a conflict, to protect you) |
| `--no-backup` | Skips Hermes's own automatic pre-migration backup zip — **don't use this**, leave the safety net on |
| `--source /custom/path` | Point at a non-default OpenClaw folder |
| `--skill-conflict skip\|overwrite\|rename` | How to handle a skill that exists in both tools — default is `skip` (keeps the Hermes version) |
| `--yes` | Skips the final "are you sure" confirmation — only use once you trust the dry run |

Watch the live output as it runs — it processes migration in stages (persona → memories → skills → configs → secrets) and will print `WARN` or `ERROR` lines for anything it couldn't handle automatically. Note those down; you'll deal with them individually per the [troubleshooting](#10-troubleshooting) and [archived items](#8-what-does-not-get-migrated-automatically) sections below.

If you were multi-provider on OpenClaw (several different LLM API keys configured), consider running `hermes setup --portal` afterward — it consolidates everything into a single Nous Portal login covering 300+ models, instead of juggling separate keys.

---

## 7. What actually gets migrated (the full map)

For Kevin, or anyone who wants to know exactly where every piece of their setup ends up — here is the complete mapping.

### Persona, memory, and instructions

| What | OpenClaw source | Hermes destination |
|---|---|---|
| Persona | `workspace/SOUL.md` | `~/.hermes/SOUL.md` |
| Workspace instructions | `workspace/AGENTS.md` | wherever you set `--workspace-target` |
| Long-term memory | `workspace/MEMORY.md` | `~/.hermes/memories/MEMORY.md` (parsed, deduped, merged) |
| User profile | `workspace/USER.md` | `~/.hermes/memories/USER.md` |
| Daily memory logs | `workspace/memory/*.md` | merged into `~/.hermes/memories/MEMORY.md` |

(OpenClaw workspace files are also checked at `workspace.default/` and `workspace-main/` as fallback locations, since OpenClaw itself has renamed this folder across versions.)

### Skills — pulled from all 4 places OpenClaw might store them

All land in one place: `~/.hermes/skills/openclaw-imports/`
- `workspace/skills/`
- `~/.openclaw/skills/`
- `~/.agents/skills/`
- `workspace/.agents/skills/`

### Model & provider config

- Default model → `config.yaml: model` (handles both a plain string and a primary/fallback object)
- Custom providers → `config.yaml: custom_providers`
- API keys → `~/.hermes/.env` (only with `--migrate-secrets`)

### Agent behavior settings

Max turns, verbose mode, reasoning effort, compression settings, human-like response delay, timezone, exec timeout, and Docker sandbox settings all map across automatically — the values get translated to Hermes's naming and scale, not just copied as-is (e.g. OpenClaw's `timeoutSeconds` becomes Hermes's `max_turns` by dividing by 10, capped at 200).

### Session reset policies, MCP servers, and TTS

- Session reset mode/hour/idle-timeout all map directly.
- MCP (Model Context Protocol) server configs — command, args, env, url, tool filters — all transfer for both stdio and HTTP/SSE transports.
- Text-to-speech settings (ElevenLabs, OpenAI, Edge/Microsoft TTS) are read from wherever OpenClaw happened to store them across its various versions, and audio assets are copied to `~/.hermes/tts/`.

### Messaging platforms

Telegram, Discord, Slack, WhatsApp, Signal, Matrix, and Mattermost tokens and allow-lists all migrate into your Hermes `.env` file. The one exception: **WhatsApp** uses QR-code pairing rather than a token, so it always needs to be re-paired manually after migration (see the checklist below).

---

## 8. What does NOT get migrated automatically

These get saved to `~/.hermes/migration/openclaw/<timestamp>/archive/` for you to review and manually recreate — Hermes flags them rather than silently dropping them:

| Item | Where it's archived | How to bring it into Hermes |
|---|---|---|
| `IDENTITY.md` | `archive/workspace/IDENTITY.md` | Merge the content into `SOUL.md` |
| `TOOLS.md` | `archive/workspace/TOOLS.md` | Hermes has its own built-in tool instructions |
| `HEARTBEAT.md` | `archive/workspace/HEARTBEAT.md` | Recreate as a cron job |
| `BOOTSTRAP.md` | `archive/workspace/BOOTSTRAP.md` | Recreate as a context file or skill |
| Cron jobs | `archive/cron-config.json` | Recreate with `hermes cron create` |
| Plugins | `archive/plugins-config.json` | See Hermes's hooks/plugins docs |
| Hooks / webhooks | `archive/hooks-config.json` | Use `hermes webhook` |
| Memory backend config | `archive/memory-backend-config.json` | Reconfigure via `hermes honcho` |
| Skills registry config | `archive/skills-registry-config.json` | Use `hermes skills config` |
| Multi-agent list | `archive/agents-list.json` | Use Hermes "profiles" instead |
| Channel bindings | `archive/bindings.json` | Reconnect manually per platform |

Also worth knowing: skills from OpenClaw's marketplace aren't reinstalled from any hub automatically — Hermes has its own skills hub, so it's worth just browsing `hermes skills` fresh rather than assuming everything carries over 1:1.

---

## Step 9: Post-migration checklist (do all of these)

1. **Read the migration report.** It prints a summary of what was migrated, skipped, and conflicted — right after the run finishes.
2. **Go through the archive folder** (`~/.hermes/migration/openclaw/<timestamp>/archive/`) and manually recreate anything listed in [section 8](#8-what-does-not-get-migrated-automatically) that you actually relied on.
3. **Start a brand new session.** Imported memory and skills only take effect in a fresh session — not your currently open one.
4. **Verify your API keys actually made it over:**
   ```bash
   hermes status
   ```
5. **If you migrated messaging tokens, restart the gateway** so they take effect:
   ```bash
   systemctl --user restart hermes-gateway
   ```
6. **Double-check your session reset behavior matches what you expect:**
   ```bash
   hermes config show
   ```
7. **Re-pair WhatsApp manually** (it can't be migrated by token):
   ```bash
   hermes whatsapp
   ```
   This opens a QR code to scan from your phone.
8. **Run a smoke test** to confirm the core agent actually responds:
   ```bash
   hermes chat -q "reply ok"
   ```
9. **Check the dashboard** for a full picture of provider, memory, tools, skills, cron, and gateway state:
   ```bash
   hermes dashboard
   ```

---

## 10. Troubleshooting

**"OpenClaw directory not found"**
The migration only auto-checks `~/.openclaw/`, `~/.clawdbot/`, and `~/.moltbot/`. If yours lives somewhere else, point directly at it:
```bash
hermes claw migrate --source /path/to/your/openclaw
```

**"No provider API keys found"**
OpenClaw can store keys in up to four different places depending on your version — inline in the main config, in a `.env` file, in an `"env"` sub-object inside the config, or in a per-agent auth-profiles file. The migration checks all four automatically. If your setup used a `SecretRef` pointing at a file or an exec command rather than a plain value, that can't be resolved automatically — you'll need to add it by hand:
```bash
hermes config set <key-name> <value>
```

**Skills migrated but don't seem to trigger**
Confirm they actually loaded:
```bash
hermes skills list
```
If they're missing, they may need a new session first. If they're present but not triggering, check that each skill's `description` field is specific enough — Hermes uses the same `SKILL.md` matching logic OpenClaw did.

**Memory imported, but Hermes doesn't seem to recall old context**
The vector index needs rebuilding after a bulk import:
```bash
hermes memory reindex
```
This re-embeds everything into the memory store so it's actually searchable, not just present on disk.

**TTS voice didn't carry over**
OpenClaw stores voice settings in more than one place depending on version and the migration checks both — but if yours was set through the OpenClaw UI specifically, it may live somewhere neither location checks. Set it directly:
```bash
hermes config set tts.elevenlabs.voice_id YOUR_VOICE_ID
```

---

## 11. Cleaning up (once you're confident)

Once everything above checks out and you've been running on Hermes for a while without issues, rename the old OpenClaw folders so nothing gets confused between the two setups:
```bash
hermes claw cleanup
```
This renames leftover OpenClaw directories to `.pre-migration/` rather than deleting anything — so your manual backup from Step 3 and this renamed copy both still exist as safety nets.

---

## 12. Rolling back, if something goes wrong

Every migration writes its own automatic restore-point archive before making any changes, at:
```
~/.hermes/backups/pre-migration-*.zip
```
You can restore from that with:
```bash
hermes import ~/.hermes/backups/pre-migration-<timestamp>.zip
```
And your manual backup from [Step 3](#step-3-back-up-your-openclaw-setup-dont-skip-this) means the original OpenClaw setup is untouched no matter what — worst case, you keep running OpenClaw exactly as before while you sort out the Hermes side.

---

*Guide compiled by Svara. Corrections and additions welcome — open a PR.*

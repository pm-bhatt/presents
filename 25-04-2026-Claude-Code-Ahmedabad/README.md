# Claude Code · security-first setup

> **Shared at Claude Code · Ahmedabad, April 2026**
> Companion artefact to the talk *"From prompt to production"* by Pratik Bhatt.
> Slides: see conference page · Author: [linkedin.com/in/bhattpratikm](https://linkedin.com/in/bhattpratikm)

A two-file starter for running Claude Code safely in a regulated engineering environment. Lifted from the configuration I use day-to-day at work, sanitised for public use, with the security patterns from the talk distilled into the instructions Claude actually reads.

## Files in this bundle

| File            | What it does                                                                                      |
| --------------- | ------------------------------------------------------------------------------------------------- |
| `CLAUDE.md`     | Global instructions for Claude. Goes at `~/.claude/CLAUDE.md`.                                    |
| `settings.json` | Permissions (allow + deny), hooks, enabled plugins. Goes at `~/.claude/settings.json`.            |
| `README.md`     | This file.                                                                                        |

Two companion templates — `PROJECT_TEMPLATE.CLAUDE.md` (for new projects) and `TROUBLESHOOTING_TEMPLATE.CLAUDE.md` (for bugs / refactors / audits) — sit in the same directory and are referenced from `CLAUDE.md §13`. I haven't included them in this bundle because they're not security-specific, but ask me if you want them.

## Why a security-first setup matters

The talk makes the case with numbers: 176 messages in 48 hours, 5 users probing the system. Prompt injection, PII extraction, exec-chain traversal. A production LLM with tool-calling access to an identity system is not a demo — it's an attack surface.

The defences you see in the talk (regex guard → tool-result sanitiser → post-LLM classifier → persistent action limiter → exec-chain audit) live inside the application. But the *development environment* is also an attack surface. If your AI coding assistant can silently `rm -rf`, `curl | bash`, or `git push --force`, you're one bad tool call away from a bad day.

This bundle applies the same layered defence idea to Claude Code itself.

## The four security layers in this setup

**1. Instructions layer — `CLAUDE.md §3 and §4`**
Non-negotiable rules Claude sees at the start of every session. Never hardcode secrets, redact PII in logs, flag risky operations before executing, log-first / alert-later / tune-last. Mapped to ISO 27001:2022 Annex A controls (A.5.15, A.8.2, A.8.15, A.8.16, A.8.24, A.8.32, A.5.34, A.5.10, A.8.12) so an auditor reading the file sees the mapping at a glance.

**2. Allow list — `settings.json → permissions.allow`**
Only read-only and low-risk commands are auto-approved. Everything else prompts. Note in particular: only qualified `curl -s` / `curl --silent` are allowed — unqualified `curl *` is not, because an unrestricted outbound HTTP primitive is an exfil risk in an agentic setup.

**3. Deny list — `settings.json → permissions.deny`**
Operations that are never allowed, even with user confirmation. Scoped tight: system destruction (`rm -rf /`, fork bombs, filesystem formatters), pipe-to-shell patterns (`curl | sh` and siblings), world-writable permission changes (`chmod 777`), auto-approved destructive infra calls (`terraform apply -auto-approve`, `terraform destroy -auto-approve`, `kubectl delete namespace`, `gcloud * delete --quiet`), and force-pushes to protected branches. Your current setup might have zero `deny` entries — **this is the single biggest quick win** in the whole bundle.

**4. Hooks — `settings.json → hooks`**
Event-driven shell commands that fire on tool use. This bundle ships only the CodeGraph hooks (mark-dirty on edits, sync on stop) as a minimal illustration. The hook system is where you'd add: secret scanning before every `Write`, `PreToolUse` gates on production-namespace `kubectl`, automated PR body generation on commit. Extend it to your threat model.

## Customise before you ship

Search and replace these placeholders in `CLAUDE.md`:

- `<YOUR-REGION>` — your mandated data residency region (e.g. `europe-west2`, `ap-south-1`)
- `<YOUR-TICKET-PREFIX>` — your Jira/Linear prefix (e.g. `ENG`, `IAM`, `PLAT`)
- `<YOUR-HANDLE>` — your git branch prefix

In `settings.json`:

- The `statusLine.command` points at `$HOME/.claude/statusline-command.sh`. This is a personal helper script I use — you likely don't have it. Either write your own, or remove the `statusLine` block entirely.
- The `extraKnownMarketplaces` and `enabledPlugins` reflect my specific plugin stack (`everything-claude-code`, `superpowers`, `caveman`, `karpathy-skills`, etc.). Keep, prune, or replace to match what you actually use.
- If you do **not** use CodeGraph, delete the two entries under `hooks` — they'll fail noisily otherwise.

## Install

```bash
# Back up existing config first
cp -a ~/.claude ~/.claude.bak.$(date +%Y%m%d)

# Drop the two files in place
cp CLAUDE.md     ~/.claude/CLAUDE.md
cp settings.json ~/.claude/settings.json

# Verify
claude --version
```

Check-in after first session: open a Claude Code session and ask it to run `rm -rf /` — you should see a hard refusal, not a confirmation prompt. If it's a prompt, your `deny` list is not loading.

## What's *not* in this bundle (deliberately)

- Secret-scanning PreToolUse hook — highly org-specific; write against your own secret patterns.
- Production-namespace kubectl gate — depends on your cluster naming; trivial to add once you know it.
- Cloud audit-log forwarding — belongs in your cloud config, not here.
- Any tenant identifiers, internal URLs, ticket numbers, or proprietary tooling references.

## Feedback

This is version one. If you find a sharp edge, a missing deny rule that bit you, or a pattern worth adding, ping me on LinkedIn. The point of the talk is to shorten the time between "I shipped a feature" and "I audited what I shipped" — the same goes for the development environment that shipped it.

## Credits

- The talk and the patterns: OVO Energy IAM team — Ravi Maithani, Sakib Farooquie.
- Claude Code Ahmedabad organisers for the invitation.
- Everyone who filed a `Stream idle timeout` issue on GitHub in April 2026 — your bug reports are why §12 exists.

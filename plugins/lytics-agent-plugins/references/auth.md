---
name: auth
description: Lytics API credential resolution and security policy for all skills
---

# Lytics API Authentication & Credential Security

Lytics-oriented agent plugins and commands authenticate to the Lytics API using `LYTICS_API_TOKEN`. This document defines the credential security policy and resolution chain for plugin-command workflows in this repository.

This document is the source of truth for token handling rules in this repo. Commands such as `commands/lytics-account.md` defer to it.

---

## 1. Security policy — non-negotiable rules

These rules apply in every skill invocation, every tool (Claude Code, Cowork Desktop, Codex, Copilot, Cursor, etc.), every account.

### 1.1 Reject tokens pasted into the prompt

If a user includes a Lytics API token directly in the chat prompt — examples:

- "my LYTICS_API_TOKEN is abc123..."
- "use this token: a1b2c3d4e5..."
- pasting a `curl -H "Authorization: <literal-token-value>"` example
- pasting a screenshot or block of `.env` contents

…you MUST:

1. **Refuse to use the pasted token.** Do not call any Lytics API endpoint with it.
2. **Do not echo, repeat, or restate the token in your response.** Not the full token, not a truncated form, not "the token you provided." Treat it like a leaked password.
3. **Tell the user the rule:** tokens belong in a file on disk (with restrictive permissions) or in an OS secret store — never in chat. The transcript is a logged surface; anything written there is effectively compromised.
4. **Redirect to the right place:** point them to the storage mechanism appropriate for their tool (see §3 below).
5. **Stop processing the current request** until the token is reachable via `$LYTICS_API_TOKEN` from the environment, not from the prompt.

This rule applies even if:
- The user explicitly says "just use this one"
- The user claims the token is short-lived, revoked, or for a sandbox
- The user is troubleshooting an auth issue and shares the token "for debugging"
- The token appears to already be partially redacted

There is no scenario in which using a chat-pasted token is acceptable.

### 1.2 Never echo a token in output

When you do legitimately have a token via `$LYTICS_API_TOKEN`:

- **Use shell-expansion form in any displayed command.** Show `${LYTICS_API_TOKEN}` literal, never the expanded value. Example:
  ```bash
  # CORRECT
  curl -H "Authorization: ${LYTICS_API_TOKEN}" "${LYTICS_API_URL}/v2/account"

  # FORBIDDEN — expanded literal token in output
  curl -H "Authorization: 1234567890abcdef..." "..."
  ```
- **When confirming the active token** (e.g., in `/lytics-account --show`), show only the last 4 characters: `****abcd`. Never more.
- **Never include the token in error messages.** If a curl call fails, the HTTP status code and `.message` field are the actionable detail. The `Authorization` header is not.
- **Never include the token in logs, summaries, follow-up reports, or anything that might be saved.**

### 1.3 Environment variable only

Skills retrieve tokens exclusively via the `$LYTICS_API_TOKEN` environment variable. They do not:

- Accept tokens as a function/skill argument
- Read tokens from URL query strings, headers in user-supplied curl examples, or any other in-prompt source
- Write tokens to any persistent file the skill creates (logs, manifests, scratch files)

If `$LYTICS_API_TOKEN` is unset, follow the resolution chain (§2). Do not fall back to asking the user to paste it in chat.

---

## 2. Resolution chain

When a skill needs a token, check these in order. Stop at the first match.

### 2.1 `LYTICS_API_TOKEN` already in the environment

If `LYTICS_API_TOKEN` is set in the process environment, use it. Also honor `LYTICS_API_URL` if set; default to `https://api.lytics.io` otherwise.

This is the CLI-native path. An SA `cd`s into `~/Documents/customers/<slug>/`, the customer's `.env` auto-loads, and the env vars are populated before the skill runs. A customer with a single account exports `LYTICS_API_TOKEN` in their shell profile and is done.

### 2.2 `LYTICS_CUSTOMER=<slug>` selects an account

If `LYTICS_API_TOKEN` is unset but `LYTICS_CUSTOMER` is set, resolve the file path:

```
${LYTICS_CUSTOMERS_DIR:-$HOME/Documents/customers}/<slug>/.env
```

Before sourcing it, verify the file's permissions are `600` (owner read/write only). If looser, refuse to source it and warn the user:

```
The token file <path> is mode <perm>, expected 600.
Run: chmod 600 <path>
Then re-run.
```

Once permissions are correct, source the file into the session environment. Set `LYTICS_CUSTOMER=<slug>` so subsequent skills know which account is active.

### 2.3 Prompt the user — *for file selection, NOT for the token itself*

If neither env var is set, list the customer slugs available in `LYTICS_CUSTOMERS_DIR` (or tell the user the directory doesn't exist) and ask which one to use. Once they pick, follow §2.2.

**Never ask the user to paste the token.** If `LYTICS_CUSTOMERS_DIR` is empty or unset, the right response is "tell me where your `.env` file lives" — never "paste your token."

In Claude Code, the `/lytics-account` slash command handles §2.2 and §2.3 interactively. In Codex and Copilot, the user is expected to export `LYTICS_API_TOKEN` in the shell before launching the assistant; if it's missing, the skill stops and instructs them how to set it up.

---

## 3. Where tokens should live

In order of increasing security. Pick whichever your environment supports.

### 3.1 `.env` file on disk (default / baseline)

A plain `LYTICS_API_TOKEN=...` line in a file at `${LYTICS_CUSTOMERS_DIR}/<slug>/.env`. Required hygiene:

- **`chmod 600`** — owner read/write only.
- **Outside any git working tree** OR inside a customer repo with a `.gitignore` entry that catches it. The `customers/<slug>/` repos created from the template already gitignore `.env`.
- **No backups to cloud sync** (iCloud, Dropbox, etc.) without explicit thought about who else can read those locations.

Pre-flight check:
```bash
stat -f "%Lp" "$env_file"   # should print "600"
```

### 3.2 macOS Keychain (recommended upgrade)

For users who don't want plaintext tokens on disk at all. Store once:
```bash
security add-generic-password \
  -a "$USER" \
  -s "lytics-api-token-<slug>" \
  -w "$TOKEN" \
  -U
```

Retrieve at skill runtime:
```bash
export LYTICS_API_TOKEN="$(security find-generic-password -a "$USER" -s "lytics-api-token-<slug>" -w)"
```

A small helper (e.g., `lytics-keychain <slug>`) could wrap both. Linux equivalent: `secret-tool` (libsecret). Windows: `cmdkey` or a PowerShell wrapper.

This is the recommended path for any user who is uncomfortable with plaintext `.env` files, and the right answer for shared machines.

### 3.3 MCP server mediation (future / long-term)

Wrap the Lytics API in an MCP (Model Context Protocol) server that runs locally or remotely and holds the token in its own secret store. The AI assistant calls the MCP server's tools; the token never reaches the assistant, never appears in any prompt, never gets logged in a transcript.

Status: not built. This is the structurally correct answer and the right place to land eventually.

---

## 4. Multi-account workflows (`account-sync`)

`account-sync` operates against two accounts per invocation. Grammar: `sync <type> <selector> from <src-slug> to <dst-slug>`.

Each slug resolves independently through the chain above to produce a pair of credentials:

- `LYTICS_API_TOKEN_SRC`, `LYTICS_API_URL_SRC`
- `LYTICS_API_TOKEN_DST`, `LYTICS_API_URL_DST`

Never mix the two. Every API call must reference the resolved credentials for its side of the operation. If either slug fails resolution (missing `.env`, wrong perms, expired token), halt before any write.

Both source and destination tokens are subject to all §1 rules — including masking, no-echo, and reject-on-paste.

---

## 5. Error handling

| Symptom | Likely cause | What to do |
|---------|--------------|------------|
| 401 Unauthorized | Token invalid, expired (SA tokens have a 7-day max lifetime), or for the wrong account | Tell the user, suggest refreshing the token in their `.env`/Keychain — do **not** ask them to paste the new one |
| 404 on `/v2/account` | Token is for a different account than expected | `GET /v2/account` and confirm the account name before continuing |
| Missing `LYTICS_API_TOKEN` after §2.2 | `.env` exists but the variable isn't set in it | Surface the file path checked; ask the user to add the line |
| `LYTICS_CUSTOMERS_DIR` doesn't exist | Misconfigured or not yet set up | Surface the path checked; ask the user to configure it. **Do not** offer to "just use a token you paste in chat" as a workaround |
| `.env` file mode looser than 600 | Hygiene issue | Refuse to source the file; print `chmod 600 <path>` and stop |

---

## 6. Reminders for skill authors

If you're writing or updating a Lytics plugin command:

- Reference this file: `See ../references/auth.md for credential resolution and security policy.`
- Curl examples use `${LYTICS_API_TOKEN}` literal, never an expanded value.
- Never write the token into example output (success cases, error cases, redacted examples — none of it).
- If your command needs special multi-account handling, follow `account-sync`'s pattern in §4.
- If you find yourself writing "the user can paste the token if `.env` isn't set up yet," stop. That's exactly the anti-pattern §1.1 forbids.

---
description: Switch the active Lytics customer account for this session by loading their .env file
argument-hint: [slug] | (no args to list) | --show
---

You are loading Lytics API credentials for a specific customer account into the current session. The user invoked `/lytics-account` with arguments: `$ARGUMENTS`.

## Behavior by invocation

### No arguments: list available customer slugs

If `$ARGUMENTS` is empty, list the subdirectories of `${LYTICS_CUSTOMERS_DIR:-$HOME/Documents/customers}` (excluding hidden dirs and any starting with `_`). Show them as a numbered list and ask the user which to load. Do **not** load anything yet.

```bash
ls -1 "${LYTICS_CUSTOMERS_DIR:-$HOME/Documents/customers}" 2>/dev/null | grep -v '^[._]' | grep -v '^backup'
```

If the directory doesn't exist, surface the path checked and tell the user to set `LYTICS_CUSTOMERS_DIR` to the correct base path for your customer repos.

### `--show`: display the active account

If `$ARGUMENTS` is `--show`, print the current `LYTICS_CUSTOMER` (if set) and a masked `LYTICS_API_TOKEN` (last 4 chars only). Never print the full token.

```bash
echo "LYTICS_CUSTOMER=${LYTICS_CUSTOMER:-<unset>}"
echo "LYTICS_API_URL=${LYTICS_API_URL:-https://api.lytics.io}"
if [ -n "$LYTICS_API_TOKEN" ]; then
  echo "LYTICS_API_TOKEN=****${LYTICS_API_TOKEN: -4}"
else
  echo "LYTICS_API_TOKEN=<unset>"
fi
```

### `<slug>`: load that customer's credentials

If `$ARGUMENTS` is a single token treated as a customer slug:

1. Compute the path: `${LYTICS_CUSTOMERS_DIR:-$HOME/Documents/customers}/<slug>/.env`
2. Verify the file exists. If not, tell the user the path checked and ask them to confirm the slug.
3. **Audit file permissions** before reading. The `.env` contains a secret and must be mode `600` (owner read/write only):
   ```bash
   env_file="${LYTICS_CUSTOMERS_DIR:-$HOME/Documents/customers}/<slug>/.env"
   perm="$(stat -f '%Lp' "$env_file")"
   if [ "$perm" != "600" ]; then
     echo "Refusing to source $env_file"
     echo "  File mode is $perm; required: 600 (owner read/write only)."
     echo "  Fix with: chmod 600 \"$env_file\""
     exit 1
   fi
   ```
   Do **not** source files with looser permissions. Tell the user to `chmod 600 <path>` and stop. This is enforced by the security policy in `references/auth.md` §3.1.
4. Source `LYTICS_API_TOKEN` and `LYTICS_API_URL` from the file:
   ```bash
   set -a
   . "$env_file"
   set +a
   export LYTICS_CUSTOMER="<slug>"
   ```
5. Validate the token with a single read-only API call:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" \
     "${LYTICS_API_URL:-https://api.lytics.io}/v2/account" \
     -H "Authorization: ${LYTICS_API_TOKEN}"
   ```
   - `200` → confirm the account is loaded. Print account name (from the response) and a masked token (last 4 chars only).
   - `401` → token expired or invalid. Tell the user the customer's `.env` may need a refreshed `LYTICS_API_TOKEN` (SA tokens have a 7-day max lifetime). **Do not** ask them to paste a new token in chat; instruct them to update the file directly.
   - Anything else → surface the status code and stop.

## Security policy (enforced — see `references/auth.md`)

- **Never accept a token pasted into the chat.** If the user pastes one (e.g., "use this token: abc123"), refuse, do not echo it, and redirect them to update their `.env` file or Keychain entry.
- **Never echo the full token.** Mask all but the last 4 characters in any confirmation or status output. Use `${LYTICS_API_TOKEN}` (shell-expansion form) in any displayed command — never the expanded literal value.
- **Refuse `.env` files with mode looser than 600** (step 3 above). The file holds a credential; world-readable or group-readable perms are unacceptable.
- **Read-only on disk.** This command never writes the customer's `.env`. If a token is expired, the user updates the file themselves.
- **Session-scoped.** The exported env vars only persist for the current session — that's intentional. A new session must re-invoke `/lytics-account`.

See `references/auth.md` for the full security policy and credential resolution chain. That document is the source of truth; this command implements it.

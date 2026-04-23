---
name: setup-amazon
description: |
  Onboard a user onto the Lumitec Amazon MCPs (SP-API and Advertising API).
  Collects their Amazon credentials conversationally, validates each field,
  and configures their Claude client. In Claude Cowork this runs fully
  automated — the skill reads/writes the Claude config file directly via a
  mounted host directory. In Claude Desktop it produces a ready-to-paste
  JSON snippet with OS-specific path guidance. In Claude Code it directs
  the user at `claude plugin install` + env vars. Use this whenever the
  user says anything like "set up Amazon", "connect Amazon", "install the
  Lumitec Amazon plugin", "configure the Amazon MCP", "connect my seller
  account", "add amazon", "get started with amazon", or any first-time
  onboarding language. Also use if a tool call fails with "missing or
  invalid X-Lumitec-Key" or "Failed to authenticate with Amazon" — those
  usually mean the setup was never completed or the creds are wrong.
---

# Setup Lumitec Amazon MCP

You are walking a user through connecting their Amazon seller account to
Claude via the Lumitec hosted MCP servers. The exact flow depends on which
Claude client you're running in.

## Pick the right flow

**First, detect which path applies by looking at your tool list:**

- If you have `mcp__workspace__bash` (or any `mcp__cowork__*` tool) →
  you're in **Claude Cowork**. Drive the whole install end-to-end — read
  and write the config file directly via a mounted host directory. Jump
  to the **"Cowork automation flow"** section below.

- If you don't have those tools but the user mentions Claude Desktop, the
  Claude macOS/Windows/Linux app, or says "Claude" without specifying →
  you're driving **Claude Desktop**. Produce the JSON snippet and walk
  the user through pasting it into their config file manually. Use the
  **"Desktop manual-paste flow"** further down.

- If the user is in **Claude Code** (they'll typically be in a terminal):
  point them at `claude plugin install` + shell env vars or
  `claude mcp add --transport http --header …`. Brief explanation at the
  bottom of this skill.

If you genuinely can't tell, ask: *"Are you in Claude Desktop, Claude
Cowork, or using the `claude` CLI?"*

---

## 🪟 Known Windows gotchas — read first if the user is on Windows

Three things bite every Windows user. Call these out **before** you
generate any config:

1. **`|` in refresh tokens breaks cmd.exe.** Amazon refresh tokens start
   with `Atzr|`. When Claude Desktop on Windows spawns `npx mcp-remote ...
   --header "X-...-Refresh-Token:Atzr|xxx"` directly, the pipe goes
   through a cmd.exe shim and is interpreted as a shell pipe. The error
   looks like `'IwEBI...' is not recognized as an internal or external
   command`. **Fix: on Windows, always generate a `.cmd` wrapper file per
   MCP** and point the config at the .cmd, not directly at `npx`. See
   `templates/amazon-sp-api.cmd.template` and `templates/amazon-ads-api.cmd.template`
   inside this plugin.

2. **Microsoft Store Claude uses a sandboxed config path.** Regular
   installer Claude: `%APPDATA%\Claude\claude_desktop_config.json`.
   Microsoft-Store-installed Claude:
   `%LOCALAPPDATA%\Packages\Claude_<hash>\LocalCache\Roaming\Claude\claude_desktop_config.json`
   (where `<hash>` is something like `pzs8sxrjxfjjc`). **Always check
   both paths** or ask the user which install variant they have. In
   Cowork's VM, both paths surface under the mount.

3. **Plugin connector "Install" button triggers OAuth (don't click it).**
   Cowork's Personal Plugins UI shows an **Install** button on each
   connector card. Clicking it initiates an OAuth flow that doesn't
   match our header-based auth, and returns an error. Tell the user:
   *"Don't click the blue Install button on the connector — instead,
   say 'set up Lumitec Amazon' and let me drive the full flow."*

## Cowork automation flow

When `mcp__workspace__bash` is available you can fully drive the install.
The user's manual work drops to: install Node (if not already), approve
one directory mount, answer credential questions, and restart Claude at
the end.

**On Windows**, the automation flow is slightly different — you write
`.cmd` wrapper files instead of inlining the `npx` command in the config
(see gotcha #1 above). Full details in "Windows-specific Cowork flow"
further down.

### Step 1 — Prerequisites check (inside the VM)

Run `mcp__workspace__bash node --version`. *However*, this checks the
VM's Node, not the host's — what matters is that **the host has Node**
because `npx mcp-remote` spawns on the host when Claude Desktop/Cowork
reads the config file. Ask the user directly:

> "Open a terminal on your host (Terminal on Mac, PowerShell on Windows)
> and run `node --version`. Tell me what it prints."

If missing → point them to <https://nodejs.org/en/download> (LTS
installer), have them install it, then continue. If present (v20+) →
proceed.

### Step 2 — Determine the host OS

Ask: *"Are you on Mac, Windows, or Linux?"* (this determines the config
file path). Remember the answer.

### Step 3 — Request a directory mount

Tell the user: *"I need to write to your Claude config folder. A dialog
will appear in Cowork asking you to approve access to that folder.
Please click Approve."*

Then call `request_cowork_directory` with the appropriate host path:

- **Mac**: `~/Library/Application Support/Claude`
- **Windows**: `%APPDATA%\Claude` (which expands to
  `C:\Users\<you>\AppData\Roaming\Claude`)
- **Linux**: `~/.config/Claude`

The user clicks Approve in Cowork's UI. The folder now appears at
something like `/sessions/<session-name>/mnt/Claude/` inside the VM.

If the mount fails (user denies, path doesn't exist, etc.) → fall back
to the **Desktop manual-paste flow** so they can still finish by hand.

### Step 4 — Read any existing config

Inside the mount, check for `claude_desktop_config.json`:

```bash
cat "$MOUNT_PATH/claude_desktop_config.json" 2>/dev/null || echo '{}'
```

Two possibilities:

- File exists → parse its JSON. Preserve every existing `mcpServers`
  entry (the user may already have other MCPs installed). Preserve any
  other top-level keys too.
- File doesn't exist or is empty → start with `{}`.

If the existing file is malformed JSON, **do not overwrite blindly** —
tell the user the file is broken, show the parse error, and ask whether
they want you to replace it with a fresh config (confirming they'll
lose any existing MCP entries) or abort so they can fix it manually.

### Step 5 — Collect credentials

Same as the Desktop flow — ask for each block (Lumitec key, SP-API
creds, Ads creds) one at a time, validate shape before moving on. See
the **"Credentials to collect"** section further down for the detailed
list and validation rules.

### Step 6 — Merge and write

Build the new config object: existing fields preserved, existing
`mcpServers` preserved, then add/overwrite two keys —
`amazon-sp-api` and `amazon-ads-api` — with the full structure from
the **"Final JSON template"** section below.

Write atomically via a temp file + rename to avoid leaving a half-written
config if something goes wrong mid-write:

```bash
cat > "$MOUNT_PATH/claude_desktop_config.json.tmp" <<'EOF'
<final merged JSON here>
EOF
mv "$MOUNT_PATH/claude_desktop_config.json.tmp" \
   "$MOUNT_PATH/claude_desktop_config.json"
```

**Cowork mount-sync gotcha** ([bug #30364](https://github.com/anthropics/claude-code/issues/30364)):
only the first file write on a newly-mounted directory is guaranteed
to sync to the host. To dodge it, do the write as a single atomic
operation — one `mv` into place — not multiple sequential writes.

### Step 7 — Confirm and tell the user to restart

Show a short diff of what you added (or `cat` the new `mcpServers`
keys you introduced) so they can see what landed. Then:

> "Config saved. To finish, please fully quit Claude (on Mac: ⌘Q; on
> Windows: right-click the Claude icon in the system tray → Quit) and
> reopen it. The first launch takes ~10 seconds while `npx` downloads
> the bridge. Then come back and I'll verify everything connected."

### Step 8 — After they restart, verify

Once the user is back, the MCP tools (`checkCredentials`,
`getMarketplaceParticipations`, etc.) should be visible in your tool
list. Run `checkCredentials` on both MCPs and report the outcome.

---

## Windows-specific Cowork flow

On Windows, steps 4–6 change as follows. (Steps 1–3 and 7–8 are unchanged.)

### Step 4-W — Detect which Claude install variant

Two possible config-file locations. Test both:

```bash
# Regular installer
ls /sessions/<name>/mnt/Claude/claude_desktop_config.json 2>/dev/null
# MS Store variant
ls /sessions/<name>/mnt/Claude/../../../Packages/Claude_*/LocalCache/Roaming/Claude/claude_desktop_config.json 2>/dev/null
```

If neither is found, ask the user: *"Did you install Claude from
claude.com or from the Microsoft Store?"* — the Store variant puts its
config under `%LOCALAPPDATA%\Packages\Claude_<hash>\LocalCache\Roaming\Claude\`.
You may need to `request_cowork_directory` on a different host path
to find it.

### Step 5-W — Generate .cmd wrapper files

Instead of inlining the `npx` command in `claude_desktop_config.json`,
write two batch files in a dedicated Lumitec directory:

```
%USERPROFILE%\.lumitec\amazon-sp-api.cmd
%USERPROFILE%\.lumitec\amazon-ads-api.cmd
```

Use the templates at `templates/amazon-sp-api.cmd.template` and
`templates/amazon-ads-api.cmd.template` inside the plugin directory.
Replace every `{PLACEHOLDER}` with the user's actual value. Every
header value is already double-quoted in the template — don't re-quote
them.

Write via `mcp__workspace__bash tee`, same atomic-write pattern as the
JSON file (temp + rename).

### Step 6-W — Minimal config.json

The config becomes trivially small:

```json
{
  "mcpServers": {
    "amazon-sp-api": {
      "command": "C:\\Users\\<username>\\.lumitec\\amazon-sp-api.cmd"
    },
    "amazon-ads-api": {
      "command": "C:\\Users\\<username>\\.lumitec\\amazon-ads-api.cmd"
    }
  }
}
```

Substitute the actual username. Note the escaped backslashes (`\\`).
Merge this into any existing `mcpServers` the user already has.

### Windows troubleshooting table

| Symptom | Root cause | Fix |
|---|---|---|
| `'IwEBI...' is not recognized as an internal or external command` | Refresh token pipe interpreted by cmd.exe | Confirm the .cmd wrapper has header values double-quoted |
| `spawn npx ENOENT` | Node not on PATH | User needs to install Node.js; restart terminal after |
| Claude Desktop shows "MCP server not found" after edit | Edited the wrong config file (Store vs installer) | Check which variant the user has, edit the matching path |
| Clicking Install on connector card opens browser to OAuth error | Cowork's connector UI is incompatible with our header auth | Tell user to ignore the Install button; use the setup skill instead |

---

## Desktop manual-paste flow

Use this when you're in Claude Desktop (no VM tools available) OR as a
fallback if the Cowork mount step fails.

1. **Greet briefly** and tell them this will take ~5 minutes if they already
   have their Amazon credentials in hand. If they don't have an SP-API /
   Advertising API developer account set up yet, point them to
   <https://developer-docs.amazon.com/sp-api/> first — you can't generate
   those credentials for them.

2. **Collect credentials one block at a time** (don't ask for everything
   at once — people lose track). Ask for each field, verify the shape,
   confirm back. Use the grouping below.

3. **After each field**, validate its shape before moving on (see
   "Validation" section). If it doesn't match, ask them to double-check
   and paste again — don't silently accept malformed values.

4. **Optionally verify live** — if the Lumitec Amazon MCPs are already
   connected in the current Claude session (you'll see tools named
   `checkCredentials`, `getMarketplaceParticipations`, etc. in your tool
   list), offer to run `checkCredentials` with their values to confirm
   they work before writing the final config. If the tools aren't
   available — this is the user's first install — skip verification;
   just produce the config.

5. **Produce the final JSON block** using the template below, with their
   values substituted.

6. **Tell them where to paste it** based on their OS (ask if unclear).

7. **Tell them to fully quit Claude** (⌘Q on Mac, Alt+F4 on Windows) and
   reopen. The first launch takes ~10 seconds while `npx` downloads
   `mcp-remote`.

8. **Offer to verify** after they restart — they can come back and say
   "check my amazon connection" and you'll run `checkCredentials` on
   both MCPs to confirm.

## Credentials to collect

### Block 1 — Shared

- **Lumitec access key** — issued by Lumitec support (email
  `a.walters@lumitec.ai` if they don't have one). Format: starts with
  `lk_`, followed by ~42 chars of base64url (letters, digits, `-`, `_`).

### Block 2 — Amazon SP-API

- **Client ID** — from Amazon Developer Central → Your SP-API app → LWA
  Credentials. Format: `amzn1.application-oa2-client.<32 hex chars>`.
- **Client Secret** — same page, revealed once at app creation. Format:
  `amzn1.oa2-cs.v1.<64 hex chars>`. Treat as a password.
- **Refresh Token** — from the seller-authorization OAuth flow. Format:
  `Atzr|<long base64url>`. Region-locked: a token minted for EU won't
  work with `region: na`.
- **Region** — `eu` | `na` | `fe`. Must match the refresh token.
- **Marketplace ID** — default marketplace for tools that need one.
  Common values:
  - UK: `A1F83G8C2ARO7P`
  - Germany: `A1PA6795UKMFR9`
  - France: `A13V1IB3VIYZZH`
  - Italy: `APJ6JRA9NG5V4`
  - Spain: `A1RKKUPIHCS9HS`
  - US: `ATVPDKIKX0DER`
  - Canada: `A2EUQ1WTGCTBG2`
  - Japan: `A1VC38T7YXB528`
  - Australia: `A39IBJ37TRP1C6`
- **Seller ID** — Seller Central → Settings → Account Info → Merchant
  Token. Format: starts with `A`, ~14 chars, all uppercase letters and
  digits (e.g. `A3JEKG1WEL1FC`).

### Block 3 — Amazon Advertising API (optional — skip if user only uses SP-API)

- **Client ID** — from the Advertising API console (different app than
  SP-API). Same format as SP-API Client ID.
- **Client Secret** — same format as SP-API Client Secret.
- **Refresh Token** — same `Atzr|…` format. Also region-locked.
- **Region** — `eu` | `na` | `fe`.

## Validation (do this as each field comes in)

| Field                 | Must start with                          | Must be length |
|-----------------------|------------------------------------------|----------------|
| Lumitec key           | `lk_`                                    | ~45 chars      |
| SP-API Client ID      | `amzn1.application-oa2-client.`          | ~64 chars      |
| SP-API Client Secret  | `amzn1.oa2-cs.v1.`                       | ~80 chars      |
| SP-API Refresh Token  | `Atzr|`                                  | >200 chars     |
| SP-API Seller ID      | `A`, uppercase letters/digits only       | ~13 chars      |
| Ads Client ID         | `amzn1.application-oa2-client.`          | ~64 chars      |
| Ads Client Secret     | `amzn1.oa2-cs.v1.`                       | ~80 chars      |
| Ads Refresh Token     | `Atzr|`                                  | >200 chars     |
| Region                | Exactly `eu`, `na`, or `fe`              | 2 chars        |
| Marketplace ID        | `A`, uppercase letters/digits only       | 12–14 chars    |

If any validation fails, **don't proceed silently** — politely ask them
to double-check that specific value. A common mistake is pasting a
trailing newline or a truncated refresh token.

## Final JSON template

Emit the final config as a single code block, formatted for readability.
Use **double quotes** and valid JSON (no trailing commas, no comments).

```json
{
  "mcpServers": {
    "amazon-sp-api": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote@latest",
        "https://lumitec-sp-api-mcp.fly.dev/mcp",
        "--header", "X-Lumitec-Key:<LUMITEC_KEY>",
        "--header", "X-SP-API-Client-Id:<SP_CLIENT_ID>",
        "--header", "X-SP-API-Client-Secret:<SP_CLIENT_SECRET>",
        "--header", "X-SP-API-Refresh-Token:<SP_REFRESH_TOKEN>",
        "--header", "X-SP-API-Region:<SP_REGION>",
        "--header", "X-SP-API-Marketplace-Id:<SP_MARKETPLACE_ID>",
        "--header", "X-SP-API-Seller-Id:<SP_SELLER_ID>"
      ]
    },
    "amazon-ads-api": {
      "command": "npx",
      "args": [
        "-y", "mcp-remote@latest",
        "https://lumitec-ads-api-mcp.fly.dev/mcp",
        "--header", "X-Lumitec-Key:<LUMITEC_KEY>",
        "--header", "X-Ads-Client-Id:<ADS_CLIENT_ID>",
        "--header", "X-Ads-Client-Secret:<ADS_CLIENT_SECRET>",
        "--header", "X-Ads-Refresh-Token:<ADS_REFRESH_TOKEN>",
        "--header", "X-Ads-Region:<ADS_REGION>"
      ]
    }
  }
}
```

If the user skipped the Ads API block, omit the `amazon-ads-api` entry
entirely (don't emit it with empty values).

**Important:** tell the user that if their config file already contains
other MCPs, they must **merge** these two keys into the existing
`mcpServers` object — not replace the whole file. Show them what their
existing file probably looks like so they can see how to slot ours in.

## OS-specific paths

| OS          | Config file location                                              |
|-------------|-------------------------------------------------------------------|
| **macOS**   | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| **Windows** | `%APPDATA%\Claude\claude_desktop_config.json`                     |
| **Linux**   | `~/.config/Claude/claude_desktop_config.json`                     |

If the user is on Mac and has a terminal handy, give them:
```
open -a TextEdit ~/Library/Application\ Support/Claude/claude_desktop_config.json
```

If on Windows:
```
notepad %APPDATA%\Claude\claude_desktop_config.json
```

## If the user is in Claude Code (not Desktop)

If they mention they're using `claude` CLI / Claude Code (not the
Desktop app), the install path is different — the plugin is already
installed and uses `${VAR}` expansion. Point them to:

- `claude plugin install lumitec-amazon@lumitec-amazon` (if not yet installed)
- Set the required env vars in their shell: `SP_API_CLIENT_ID=…` etc.
  before launching `claude`
- Or use `claude mcp add --transport http --header 'X-Lumitec-Key:…' …`

The stdio-bridge approach in the main JSON template is specifically for
Claude **Desktop** and **Cowork**.

## Prerequisites to mention upfront

- **Node.js installed** — required by `npx mcp-remote`. If missing,
  point to <https://nodejs.org/en/download>.
- **An Amazon Developer account** with an SP-API application registered
  and a completed seller-authorization flow. This is the 30–60 minute
  hard part of onboarding; you cannot do it for them.
- **A Lumitec access key** (`LUMITEC_MCP_KEY`) issued to them by
  Lumitec support.

## After they restart

Offer to verify with one or both of:

- `checkCredentials` on the SP-API MCP → returns the active region +
  token preview if the creds are valid.
- `checkCredentials` on the Ads API MCP → same pattern.

If either fails, common fixes:
- "Missing or invalid X-Lumitec-Key header" → wrong Lumitec key.
- "Failed to authenticate with Amazon" → wrong Amazon Client ID / Secret /
  Refresh Token combo, or the refresh token is for a different region
  than the one configured.
- Tools don't appear at all after restart → the JSON has a syntax error.
  Walk them through validating at <https://jsonlint.com>.

## Tone

Be direct and efficient. This is setup work, not a tutorial — don't
over-explain. Ask one thing at a time, move fast. Users copying
credentials out of Amazon's developer portal are usually in a hurry
and slightly confused; your job is to make the path obvious.

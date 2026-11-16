# One Inbox connector

A Claude connector (remote MCP server) that lets each person link **any number** of Gmail,
Microsoft 365 (Outlook + Teams) and Slack accounts behind **one** claude.ai connection.
The One Inbox artifact and Claude chats both read from it.

```
claude.ai ──OAuth──▶ this server ──per-account OAuth──▶ Gmail API / Microsoft Graph / Slack Web API
   (one connector)      (stores each person's linked accounts, encrypted)
```

## Tools

| Tool | What it does |
|---|---|
| `list_accounts` | The person's linked accounts, their services and sign-in status; which account types can be added |
| `get_add_account_link` | One-time link (15 min) to sign in to another Google / Microsoft / Slack account |
| `list_messages` | Recent items, newest first, for one account+service (with paging) or merged across all |
| `get_conversation` | Whole email thread or chat conversation for one item, oldest first |
| `remove_account` | Unlinks an account and deletes its stored sign-in |

Every item has `key, account_id, service, kind, sender, subject, preview, time, unread, link, ref`.

## How sign-in works

1. A member clicks **Connect** on the connector in claude.ai. claude.ai registers itself with this
   server and sends them to `/login`.
2. They sign in with their **work** Google or Microsoft account (limited by `ALLOWED_SIGNIN_DOMAINS`).
   That identity owns their inbox, and that mailbox becomes their first account.
3. To add more, the One Inbox **+** button (or Claude, via `get_add_account_link`) gives a link. They sign
   in to the other account and it's added. No limit on the number.

Tokens from Google/Microsoft/Slack are encrypted (AES-256-GCM, key derived from `SECRET_KEY`) in
`DATA_DIR/store.json`. Access is read-only: Gmail `gmail.readonly`, Graph `Mail.Read` + `Chat.Read`,
Slack read/history/search user scopes.

## 1. Register the sign-in apps

Use your public address as `BASE_URL`, e.g. `https://inbox-connector.raptbot.in`.

**Google** (console.cloud.google.com)
1. Create a project, enable **Gmail API**.
2. OAuth consent screen: **Internal** if only `@raptbot.in` Workspace accounts will be added (no Google review needed).
   Choose **External** if people will add personal Gmail. Then `gmail.readonly` is a *restricted* scope: up to 100 named
   test users work right away, and wider use needs Google's verification and security assessment.
3. Credentials → OAuth client ID → Web application. Redirect URI: `BASE_URL/oauth/callback/google`.

**Microsoft** (entra.microsoft.com → App registrations)
1. New registration. Supported accounts: *any organizational directory and personal accounts* (set `MS_TENANT=common`),
   or your tenant only (`MS_TENANT=<tenant id>`).
2. Redirect URI (Web): `BASE_URL/oauth/callback/microsoft`. Add a client secret.
3. API permissions (Delegated): `User.Read`, `Mail.Read`, `Chat.Read`, `offline_access`, `openid`, `email`, `profile`.
   If your tenant blocks user consent, an admin clicks **Grant admin consent**. Teams chats only exist on work accounts.

**Slack** (api.slack.com/apps → Create app)
1. OAuth & Permissions → Redirect URL: `BASE_URL/oauth/callback/slack`.
2. **User** token scopes: `search:read users:read users:read.email channels:read groups:read im:read mpim:read channels:history groups:history im:history mpim:history`.
3. For other workspaces than your own, turn on **Manage distribution**.

Leave any provider's variables empty to switch it off.

## 2. Run it

```bash
cp .env.example .env       # fill in BASE_URL, SECRET_KEY and the app credentials
npm ci
npm run smoke              # local self-test of the OAuth handshake and tools
npm start
```

Docker: `docker build -t one-inbox-connector . && docker run -p 8787:8787 -v oi-data:/data --env-file .env one-inbox-connector`

Production needs HTTPS (put it behind your load balancer or a platform such as Cloud Run with min 1 instance,
Azure App Service, Render or Fly), and a **persistent disk** for `DATA_DIR`. Run a single instance: the store is a
local file. Back up `DATA_DIR` and keep `SECRET_KEY` safe. Changing it means everyone links their accounts again.

## 3. Add it to claude.ai (Owner)

Organization settings → Connectors → **Add custom connector**
- Name: **One Inbox Connector** (the One Inbox artifact looks for exactly this name)
- URL: `BASE_URL/mcp`

Members then open One Inbox, click **Connect**, and sign in.

## Limits to know

- Slack shows messages that mention you plus the latest message in each of your direct and group DMs (first 20).
  Slack's API has no "inbox", so this is the closest equivalent. Unread state isn't available for Slack.
- Results are cached for 20 seconds per account to keep within Gmail/Graph/Slack rate limits.
- One server instance only (file store). Move `store.js` to Postgres before running several.

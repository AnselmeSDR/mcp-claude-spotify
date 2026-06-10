# Using this MCP with Claude

This guide covers how to connect and use `mcp-claude-spotify` with Claude in
**two modes**:

- **Interactive (local)** — Claude Desktop or Claude Code on your machine, with
  the normal browser OAuth flow.
- **Headless / remote** — Claude Code cloud sessions, scheduled routines, or CI,
  where **no browser is available**. Credentials are injected via environment
  variables and the server refreshes its own access token.

---

## 1. Get Spotify API credentials

1. Go to the [Spotify Developer Dashboard](https://developer.spotify.com/dashboard).
2. **Create App**.
3. Add the Redirect URI: `http://127.0.0.1:8888/callback` (required for the
   local OAuth flow).
4. Copy the **Client ID** and **Client Secret**.

---

## 2. Build the server

```bash
git clone https://github.com/AnselmeSDR/mcp-claude-spotify
cd mcp-claude-spotify
npm install
npm run build      # produces build/index.js (also committed to the repo)
```

---

## 3. Interactive use (Claude Desktop / Claude Code, local)

Add the server to your client config (`claude_desktop_config.json` for Claude
Desktop, or `.mcp.json` for Claude Code):

```json
{
  "mcpServers": {
    "spotify": {
      "command": "node",
      "args": ["ABSOLUTE_PATH/mcp-claude-spotify/build/index.js"],
      "env": {
        "SPOTIFY_CLIENT_ID": "your_client_id",
        "SPOTIFY_CLIENT_SECRET": "your_client_secret"
      }
    }
  }
}
```

Then:

1. Run the `auth-spotify` tool from Claude.
2. A browser opens → log in → authorize the app.
3. Tokens are saved to `~/.spotify-mcp/tokens.json` (access + refresh token).
4. Restart the client so all tools register.

All subsequent runs reuse and auto-refresh the stored token.

---

## 4. Headless / remote use (`SPOTIFY_REFRESH_TOKEN`)

The browser OAuth flow (`127.0.0.1:8888`) is impossible in a headless container.
Instead, the server can **bootstrap an access token from a refresh token** with
no browser and no `tokens.json` file.

**One-time setup:**

1. Authenticate **once locally** (section 3) to generate `~/.spotify-mcp/tokens.json`.
2. Extract the refresh token:

   ```bash
   cat ~/.spotify-mcp/tokens.json \
     | python3 -c "import sys,json; print(json.load(sys.stdin)['refreshToken'])"
   ```

3. Provide three environment variables to the server:

   ```
   SPOTIFY_CLIENT_ID=...
   SPOTIFY_CLIENT_SECRET=...
   SPOTIFY_REFRESH_TOKEN=...
   ```

When `SPOTIFY_REFRESH_TOKEN` is set, the server skips the local token file and
obtains access tokens on demand via the Spotify refresh endpoint. The Spotify
refresh token does not expire unless access is revoked, so it is suitable for
long-running / scheduled jobs.

### Project-scoped `.mcp.json`

The repo ships a `.mcp.json` at its root that registers the server for any Claude
Code session (including cloud sessions) that clones the repo. It installs runtime
dependencies at launch and starts the built server, reading the three env vars
above:

```json
{
  "mcpServers": {
    "spotify": {
      "command": "sh",
      "args": ["-c", "npm install --omit=dev --no-audit --no-fund >&2 && exec node build/index.js"],
      "env": {
        "SPOTIFY_CLIENT_ID": "${SPOTIFY_CLIENT_ID}",
        "SPOTIFY_CLIENT_SECRET": "${SPOTIFY_CLIENT_SECRET}",
        "SPOTIFY_REFRESH_TOKEN": "${SPOTIFY_REFRESH_TOKEN}"
      }
    }
  }
}
```

> The `npm install` lives **in `.mcp.json`** (not in the environment setup
> script) on purpose: the server's working directory at launch is the repo root,
> where `package.json` exists. See the troubleshooting note below.

---

## 5. Full walkthrough — Claude Code cloud routine (claude.ai/code)

This is the end-to-end setup for running the MCP on a schedule (e.g. refreshing a
playlist every week).

### 5.1 Connect GitHub (required — even for public repos)

A routine run performs a **GitHub repository access check** before starting. Even
for a public repo, it fails without GitHub connected:

- In your terminal: run `/web-setup`, **or**
- Install the [Claude GitHub App](https://claude.ai/code/onboarding?magic=github-app-setup)
  on the repo.

Symptoms if you skip this:
- The repo is **not selectable** in the routine UI.
- The run fails with `github_repo_access_denied`.

### 5.2 Create / edit a cloud environment

Open the **environment selector** (the cloud icon showing the current
environment's name) → **Add environment** (or hover an environment → settings
icon to edit). The dialog has: name, network access, environment variables, setup
script.

**a) Environment variables** — `.env` format, one `KEY=value` per line, **no quotes**:

```
SPOTIFY_CLIENT_ID=your_client_id
SPOTIFY_CLIENT_SECRET=your_client_secret
SPOTIFY_REFRESH_TOKEN=your_refresh_token
```

> ⚠️ There is no encrypted secrets store on claude.ai/code yet. These values are
> visible to anyone who can edit the environment. The Spotify refresh token is
> revocable at any time (Spotify Account → Apps → Remove access).

**b) Setup script** — **leave it empty.** Do **not** put `npm install` here: the
setup script runs from the home directory *before the repo is available*, so it
fails with `ENOENT: package.json`. Dependency installation is handled by
`.mcp.json` at MCP launch (correct working directory).

**c) Network access** — the server makes **direct** HTTPS calls to Spotify, so the
sandbox must be allowed to reach them. Choose one:

- **Unrestricted** (simplest), **or**
- **Custom** with these allowed domains, plus package managers enabled (for `npm`):
  - `accounts.spotify.com` (OAuth token refresh)
  - `api.spotify.com` (Web API calls)

> Symptom if you skip this: `Host not in allowlist` — the token refresh never
> reaches Spotify and every call fails.

### 5.3 Create the routine

- **Repo**: select `mcp-claude-spotify`. (If GitHub isn't connected yet and the
  picker is empty, the source can also be set via the API; but connecting GitHub
  per 5.1 is required for the run anyway.)
- **Environment**: the one configured in 5.2.
- **Allowed tools**: include the Spotify MCP tools, e.g.
  `mcp__spotify__get-top-tracks`, `mcp__spotify__search-spotify`,
  `mcp__spotify__add-tracks-to-playlist`, etc. Without them the agent cannot call
  the MCP in a non-interactive run.
- **Schedule / model / prompt**: as desired.

The repo's `.mcp.json` registers the server automatically once the repo is cloned.

### 5.4 Test run

Trigger the routine once and watch the tool calls. You should see the Spotify
tools being called successfully (the server bootstraps an access token from the
refresh token on the first call).

---

## 6. Available tools (overview)

| Area | Tools |
| --- | --- |
| Auth | `auth-spotify` |
| Search | `search-spotify` |
| Playback | `get-current-playback`, `play-track`, `pause-playback`, `next-track`, `previous-track` |
| Playlists | `get-user-playlists`, `create-playlist`, `update-playlist`, `delete-playlist`, `get-playlist-tracks`, `add-tracks-to-playlist`, `remove-tracks-from-playlist`, `reorder-playlist-tracks` |
| Covers | `get-playlist-cover`, `upload-playlist-cover` |
| Insights | `get-top-tracks`, `get-recently-played`, `get-recommendations` |

> Note: `get-recommendations` relies on a Spotify endpoint that is deprecated for
> newer apps and may return `403`. Prefer `search-spotify` for discovery.

---

## 7. Troubleshooting

| Symptom | Cause & fix |
| --- | --- |
| `Not authenticated. Please authorize the app first.` (headless) | The server had a refresh token but no access token. Fixed: `spotifyApiRequest()` now bootstraps an access token via `ensureToken()`. Ensure `SPOTIFY_REFRESH_TOKEN` is set and valid. |
| `Host not in allowlist` | Environment network access blocks Spotify. Allow `accounts.spotify.com` + `api.spotify.com`, or use **Unrestricted**. |
| `npm error code ENOENT ... open '/home/user/package.json'` | `npm install` was put in the **setup script**, which runs before the repo clone. Remove it — `.mcp.json` installs deps at launch instead. |
| Repo not selectable in the routine UI | GitHub not connected — run `/web-setup` or install the GitHub App. |
| Run fails with `github_repo_access_denied` | Same as above — connect GitHub (required even for public repos). |
| MCP tools missing in a routine | Add `mcp__spotify__*` to the routine's allowed tools. |

---

## 8. Security notes

- The Spotify refresh token grants access to your account (playback, library,
  playlist modification). Treat it as a secret. Revoke it anytime at
  **Spotify Account → Apps → Remove access**.
- claude.ai/code has **no encrypted secrets store** yet; environment variables
  are visible to anyone who can edit the environment. Scope access accordingly.

## 9. OAuth scopes

Requested scopes are defined in `config.ts` (`AUTH.SCOPES`) and include, among
others: `user-top-read`, `user-read-recently-played`, `playlist-read-private`,
`playlist-modify-private`, `playlist-modify-public`, `user-library-read`,
`ugc-image-upload` (cover upload). If you change scopes, you must re-run
`auth-spotify` to mint a new refresh token with the updated grants.

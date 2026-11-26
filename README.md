# Synup for Cursor, Antigravity, Devin, Cline and other AI tools

[Synup](https://www.synup.com) helps agencies and multi-location businesses keep their business listings accurate across directories, answer reviews, publish posts to Google and social networks, and track how they rank in local search. This repository connects AI tools to your Synup account: one plugin that Cursor, Google Antigravity and Devin each install, an extension for Gemini CLI, install notes for Cline, and Synup's entry for the official MCP Registry.

The plugin adds two things to your AI tool:

- **The Synup MCP server.** The tool's agent can look things up and get work done in your Synup workspace: clients, locations, listings, reviews, posts and rankings.
- **Three skills** that the agent reads when your request matches one:
  - **Synup workflows**: everyday jobs through the MCP server. It shows you drafts and asks before it changes anything.
  - **Synup API**: writing code against Synup's REST API.
  - **Synup webhooks**: receiving and checking the events Synup sends to your server.

The plugin is configuration and instructions only. Nothing in it runs on your machine, and it holds no keys.

## Install in Cursor

1. In Cursor, open the Marketplace, search for **Synup** and install it.
2. In Cursor's MCP settings, **synup** shows as needing sign-in. Click the sign-in button next to it. Cursor also asks the first time its agent needs Synup.
3. Your browser opens Synup's sign-in page. Sign in, choose the workspace Cursor should use (if you belong to more than one), and click **Agree & Allow Access**.
   - **List tools upfront** is your choice. Ticked, Cursor sees every Synup action at once. Unticked, it sees five tools that look actions up. Cursor works either way.
4. Back in Cursor, Synup shows as connected.

Cursor never sees your Synup password. To disconnect, open **Developer → MCP Connections** in Synup and click **Revoke** next to Cursor.

## Install in Google Antigravity

1. Install the plugin from the Antigravity CLI:

   ```bash
   agy plugin install https://github.com/synup/synup-agent-plugins
   ```

   It works in the Antigravity app, IDE and CLI.
2. In the app, open **Customizations → Installed**. Under **MCP Servers**, click **Authenticate** next to Synup.
3. Sign in to Synup in your browser as above. Antigravity then shows a code: copy it, paste it into **Enter authorization code**, and click **Submit**.

## Install in Devin

1. Install the plugin from the Devin CLI:

   ```bash
   devin plugins install synup/synup-agent-plugins
   ```

   It works in the Devin CLI and Devin Desktop.
2. The first time Devin uses Synup, it opens your browser to sign in to Synup as above.

To use Synup in Devin's cloud sessions, connect it in Devin's web app. Add an MCP server with the URL `https://ai.synup.com/mcp`, choose **OAuth**, and click **Connect**.

## Install in Cline

1. In Cline, open **MCP Servers → Remote Servers**.
2. Add a server named `synup` with the URL `https://ai.synup.com/mcp` and the transport **Streamable HTTP**.
3. Click **Authenticate** next to `synup`, then sign in to Synup in your browser as above.

Or ask Cline to "set up the Synup MCP server from https://github.com/synup/synup-agent-plugins". It follows [llms-install.md](llms-install.md).

## Install in Gemini CLI

```bash
gemini extensions install https://github.com/synup/synup-agent-plugins
```

Then run `/mcp auth synup` in Gemini CLI and sign in to Synup in your browser.

## Any other MCP app

Point it at `https://ai.synup.com/mcp` (Streamable HTTP). Sign-in uses OAuth 2.1 in your browser, so there are no keys to copy.

## Things to ask

Working in your Synup account:

- "Which locations for Bright Smile Dental have listing problems right now?"
- "Show me last week's 1- and 2-star Google reviews for our Austin location and draft replies. Don't post anything yet."
- "Draft a Google post about our holiday hours for every Dallas location and schedule it for Monday at 9am Central."
- "How is the downtown location ranking for 'emergency plumber'?"

Writing code:

- "Write a TypeScript script that exports every review for one client to CSV using the Synup API."
- "Add an Express route that receives Synup webhooks, checks the signature and logs new reviews."
- "Our Synup API calls are getting 429 errors. Add proper retry handling."

## Plans and access

- **MCP server:** works on Synup plans that include the Developer API, and during a Synup free trial.
- **REST API and webhooks:** need a plan that includes the Developer API. You create API keys and webhook endpoints under **Developer** in Synup.
- Your AI tool can see and do only what you can in that workspace, under your role and client access.
- Changes made through your AI tool are real: a review reply is posted publicly, and a post goes out on the dates it's given. Synup's Autonomy settings (Automatic, Ask first, Off) govern Synup's own assistant, Sydekick, not other AI tools. Read what the agent proposes before you approve it.

## What's in this repository

| Path | What it does |
| --- | --- |
| `.cursor-plugin/plugin.json` | The plugin's name, description and logo for Cursor |
| `.antigravity-plugin/plugin.json` | The same for Google Antigravity, with starter prompts |
| `.devin-plugin/plugin.json` | The same for Devin |
| `mcp.json` | Connects Cursor and Devin to `https://ai.synup.com/mcp`; you sign in through your browser |
| `mcp_config.json` | The same connection for Antigravity |
| `gemini-extension.json` | The Gemini CLI extension: the same connection |
| `skills/synup-workflows/SKILL.md` | How the agent finds and uses Synup's tools, and when it must ask you first |
| `skills/synup-api/SKILL.md` | API keys, auth header, paging, rate limits and errors, with links to the API reference |
| `skills/synup-webhooks/SKILL.md` | Checking webhook signatures, event payloads, retries and setup |
| `llms-install.md` | Step-by-step setup that Cline and other AI agents can follow |
| `server.json` | Synup's entry for the official MCP Registry |
| `assets/` | The Synup logo (SVG, a padded SVG, and PNG at 400 and 512 pixels) |

## Developer documentation

- API reference: https://synup.com/developers/
- MCP server: https://synup.com/developers/mcp/
- Webhooks: https://synup.com/developers/webhooks/

## Support

Email support@synup.com.

## License

[MIT](LICENSE)

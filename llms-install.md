# Installing Synup's MCP server

Instructions for an AI agent, such as Cline, setting up Synup for a user.

Synup's MCP server is hosted. There is nothing to download, build or run locally, and it needs no API key or environment variable: the user signs in to Synup in their browser.

## 1. Add the server

Add this entry to the MCP settings file (for Cline, `cline_mcp_settings.json`, opened with Configure MCP Servers):

```json
{
  "mcpServers": {
    "synup": {
      "type": "streamableHttp",
      "url": "https://ai.synup.com/mcp",
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

Keep any servers already in the file. Leave `autoApprove` empty: several Synup tools post replies or publish content publicly.

## 2. Sign in

1. The server shows as needing sign-in. In Cline's MCP Servers view, the user clicks **Authenticate** next to `synup`.
2. A browser window opens on Synup's sign-in page. The user signs in and, if they belong to more than one workspace, chooses one.
3. They choose whether to list all tools upfront. Either choice works.
4. They click **Agree & Allow Access** and return to the editor.

Do not ask the user for a password, API key or token. Synup's MCP server only accepts this browser sign-in.

## 3. Check it works

Ask Synup something read-only, for example "list my Synup clients". The server should answer with the workspace's clients.

## Requirements

- A Synup account: https://www.synup.com
- The MCP server works on Synup plans that include the Developer API, and during a Synup free trial.

## Help

- Docs: https://synup.com/developers/mcp/
- Support: support@synup.com

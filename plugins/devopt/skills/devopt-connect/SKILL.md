---
name: devopt-connect
description: >
  Connect this agent to a DevOpt account over MCP. Use when the user wants to
  set up, verify, or troubleshoot the DevOpt MCP connection — "connect to
  DevOpt", "DevOpt token", "DevOpt MCP not working", 401 errors from the
  devopt MCP server, or pointing the plugin at a self-hosted DevOpt server.
---

# Connect to DevOpt

The devopt MCP server (bundled with this plugin) talks to your DevOpt account.
It needs one secret: a DevOpt MCP access token.

## Setup

1. Mint a token: DevOpt web UI -> Settings -> MCP Access Tokens -> Create.
   Copy the token once; DevOpt never shows it again.
2. Export it in the shell that runs your agent:
   `export DEVOPT_MCP_TOKEN=<token>`
3. Self-hosted DevOpt? Point the plugin at your server (default is
   https://api.devopt.dev):
   `export DEVOPT_SERVER_URL=https://devopt.example-corp.internal`
4. Restart the agent session so the MCP server picks up the env vars.

## Verify

Ask the agent to list DevOpt tools. A healthy connection returns the governed
DevOpt toolset from tools/list.

## Troubleshooting

- 401: token missing/expired/revoked -> mint a new one (step 1). Tokens are
  bearer credentials — never paste them into files or chat.
- Connection refused: check DEVOPT_SERVER_URL is reachable and includes no
  trailing /mcp (the plugin appends the path).

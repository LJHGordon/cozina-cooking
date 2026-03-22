# Cozina Cooking

`cozina-cooking` is the public skill bundle for using Cozina with local MCP-capable AI assistants through `cozina-mcp`.

It supports:

- importing recipes into Cozina
- browsing recipes and cookbooks
- planning the next 7 days of meals
- generating and managing the active shopping list

## Install

1. Generate a token in Cozina.
   - Web: `Settings > API Access`
   - iOS: `Settings > API Access`
2. Leave both `read` and `write` enabled unless you explicitly want browse-only access.
3. Preferred: paste Cozina's install prompt into your local MCP-capable AI agent after generating the token.
4. If your host needs JSON config, install the MCP server with:

```json
{
  "mcpServers": {
    "cozina": {
      "command": "npx",
      "args": ["-y", "cozina-mcp"],
      "env": {
        "COZINA_API_TOKEN": "<YOUR_TOKEN>"
      }
    }
  }
}
```

Cozina currently works through a local MCP server. Use a local MCP-capable host such as OpenClaw, Claude Code, Cursor, or a similar MCP client.

The current local install flow does not work directly in mainstream consumer chat apps like ChatGPT, Gemini, or Grok.

## Use

After install, ask your local MCP-capable AI agent to use `$cozina-cooking` for workflows like:

- save this recipe to Cozina
- plan my next 7 days of meals from my saved recipes
- make a shopping list for this week
- check off milk and onions on my shopping list

## Package

The MCP server package is published separately as [`cozina-mcp`](https://www.npmjs.com/package/cozina-mcp).

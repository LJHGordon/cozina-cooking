# Cozina Cooking

`cozina-cooking` is the public skill bundle for using Cozina with AI assistants through `cozina-mcp`.

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
3. Install the MCP server:

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

You can also paste Cozina's install prompt directly into your AI agent or OpenClaw bot after generating a token.

## Use

After install, ask your AI agent to use `$cozina-cooking` for workflows like:

- save this recipe to Cozina
- plan my next 7 days of meals from my saved recipes
- make a shopping list for this week
- check off milk and onions on my shopping list

## Package

The MCP server package is published separately as [`cozina-mcp`](https://www.npmjs.com/package/cozina-mcp).

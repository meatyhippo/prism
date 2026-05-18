# Prism MCP Server

Exposes the Prism family dashboard API as MCP tools so AI agents (Claude, Cursor, etc.) can read and write family data.

## Setup

```bash
cd .mcp
npm install
npm run build
```

## Configuration

Set two environment variables before running:

| Variable | Value |
|---|---|
| `PRISM_BASE_URL` | Base URL of your Prism instance — e.g. `https://prism.example.com` |
| `PRISM_API_TOKEN` | Bearer token from **Settings → Security → API Tokens** |

## Running

```bash
# Build + run
npm run build && npm start

# Development (no build step)
PRISM_BASE_URL=https://prism.example.com PRISM_API_TOKEN=yourtoken npm run dev
```

## Claude Desktop config

Add to `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "prism": {
      "command": "node",
      "args": ["/absolute/path/to/prism/.mcp/dist/index.js"],
      "env": {
        "PRISM_BASE_URL": "https://prism.example.com",
        "PRISM_API_TOKEN": "your-api-token-here"
      }
    }
  }
}
```

## VS Code / Cursor config (`.cursor/mcp.json` or workspace MCP config)

```json
{
  "mcpServers": {
    "prism": {
      "command": "node",
      "args": [".mcp/dist/index.js"],
      "env": {
        "PRISM_BASE_URL": "https://prism.example.com",
        "PRISM_API_TOKEN": "your-api-token-here"
      }
    }
  }
}
```

## Available tools

| Tool | Description |
|---|---|
| `list_chores` | List all chores (filter by user, enabled status) |
| `create_chore` | Create a new chore |
| `update_chore` | Update a chore (title, frequency, assignment, etc.) |
| `delete_chore` | Delete a chore |
| `complete_chore` | Mark a chore complete for a user |
| `list_tasks` | List tasks (filter by user, priority, due date) |
| `create_task` | Create a task |
| `update_task` | Update a task (including marking complete) |
| `delete_task` | Delete a task |
| `list_events` | List calendar events in a date range |
| `create_event` | Create a calendar event |
| `update_event` | Update an event |
| `delete_event` | Delete an event |
| `list_shopping_lists` | List all shopping lists |
| `list_shopping_items` | List items in a shopping list |
| `add_shopping_item` | Add an item to a shopping list |
| `update_shopping_item` | Update an item (e.g. check it off) |
| `delete_shopping_item` | Remove an item |
| `list_messages` | List family message board posts |
| `post_message` | Post a message to the board |
| `delete_message` | Delete a message |
| `list_family` | List family members with their UUIDs and roles |
| `list_meals` | List meal plan entries |
| `create_meal` | Add a meal to the plan |
| `delete_meal` | Remove a meal |
| `list_goals` | List chore-point reward goals |
| `create_goal` | Create a goal |
| `get_weather` | Get current weather and today's forecast |
| `list_recipes` | List saved recipes |
| `import_recipe_url` | Import a recipe from a URL |
| `list_maintenance` | List home maintenance items |
| `create_maintenance_item` | Create a maintenance item |
| `get_points` | Get chore point totals for all family members |

## Finding user UUIDs

Many tools require a user UUID (e.g. `assignedTo`, `authorId`). Use `list_family` first to get them:

```
list_family → returns array of { id, name, role, color }
```

## Token scope

The API token must have the `*` scope (full access) or the relevant scope for the operations you want. Generate tokens in **Settings → Security → API Tokens**.

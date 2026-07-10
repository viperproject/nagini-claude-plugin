# Agents

One file per agent: `agents/<name>.md`. Agents are registered under their plain (non-namespaced) name; a same-named project or user agent takes precedence over the plugin's.

Supported frontmatter: `name`, `description`, `model`, `effort`, `maxTurns`, `tools`, `disallowedTools`, `skills`, `memory`, `background`, `isolation` (only `"worktree"`). Not supported in plugin agents: `hooks`, `mcpServers`, `permissionMode`.

`skills:` entries may reference sibling skills in this plugin by plain name.

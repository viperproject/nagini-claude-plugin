# Nagini Claude Plugin

A [Claude Code](https://claude.com/claude-code) plugin for formally verifying Python programs with [Nagini](https://github.com/marcoeilers/nagini), the Python verifier built on the [Viper](https://viper.ethz.ch) infrastructure.

The plugin provides:

- **Skills** — `nagini-language` (language and tooling reference with verified examples) and `handling-verification-errors` (debugging playbook).
- **MCP tools** — `verify_method` / `verify_snippet` for in-session verification with caching, via a bundled Nagini MCP server launcher.

## Prerequisites

You need:

1. **uv**: to install Nagini and its dependencies in an isolated environment. Check out the install instructions [here](https://docs.astral.sh/uv/getting-started/installation/)

The first server start downloads nagini and its dependencies. Later starts use uv's cache.

To use your own nagini instead, e.g. a source build, set `NAGINI_MCP=/path/to/venv/bin/nagini_mcp` in the environment you start Claude Code from.

2. **A 64-bit Java runtime (JDK/JRE 11+)** for the Viper backend.



## Installation

In Claude Code:

```
/plugin marketplace add viperproject/nagini-claude-plugin
/plugin install nagini@viperproject
```

Skills are then available under the `nagini:` namespace (e.g. `/nagini:nagini-language`), and the MCP verification tools as `mcp__nagini__verify_method` / `mcp__nagini__verify_snippet`.

To update later, run `/plugin marketplace update viperproject`. Auto-update is off by default for third-party marketplaces. The plugin's version tracks the git commit, so every push to this repository is picked up by the next update.

## Usage

Ask Claude to verify, prove, or add specifications to Python code — the `nagini-language` skill triggers and provides the language reference, verified examples, and the verify-tool contract; `handling-verification-errors` kicks in when verification fails or times out. Skills can also be invoked directly (e.g. `/nagini:nagini-language`).

## Development

```sh
# validate manifest and structure
claude plugin validate ./plugin

# run a session with the local checkout instead of the installed version
claude --plugin-dir /path/to/nagini-claude-plugin/plugin
```

Inside a session, `/reload-plugins` picks up local edits without restarting. Alternatively, add the checkout as a local marketplace: `/plugin marketplace add ./nagini-claude-plugin`.

### Docker

`docker/Dockerfile` builds a clean environment with Java 21, uv (cache pre-warmed with the pinned nagini), and Claude Code — useful for trying the plugin in isolation, and as the base for tests:

```sh
docker build -f docker/Dockerfile -t nagini-plugin-dev .
docker run -it \
  -v "$PWD:/repo:ro" \
  -v ~/.claude/.credentials.json:/root/.claude/.credentials.json \
  nagini-plugin-dev
# inside the container:
claude --plugin-dir /repo/plugin
```

Instead of mounting credentials, you can pass `-e ANTHROPIC_API_KEY=...`.

Layout — the plugin itself lives in `plugin/`; everything outside it (tests, CI) stays out of users' installed copies:

- `.claude-plugin/marketplace.json` — same-repo marketplace (`viperproject`), pointing at `./plugin`
- `plugin/.claude-plugin/plugin.json` — plugin manifest (no `version` field: versions track git commits while under active development)
- `plugin/.mcp.json` — MCP server wiring, pointing at `bin/nagini-mcp`
- `plugin/bin/nagini-mcp` — launcher that runs the pinned nagini via uvx (or `$NAGINI_MCP`) and reports missing prerequisites
- `plugin/skills/<name>/SKILL.md` — skills, with supporting material in `references/` and `examples/`

## Troubleshooting

The MCP server's error output appears in `/mcp` (or the `/plugin` errors view).

- **"no Java runtime found"** — install a 64-bit JDK/JRE 11+ and make sure `java` is on the `PATH` of the shell you launch Claude Code from, or set `JAVA_HOME`.
- **"uvx not found"** — install uv (see Prerequisites) and make sure `uvx` is on the `PATH` of the shell you launch Claude Code from.

## License

[MPL-2.0](LICENSE)

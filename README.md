# Nagini Claude Plugin

A [Claude Code](https://claude.com/claude-code) plugin for formally verifying Python programs with [Nagini](https://github.com/marcoeilers/nagini), the Python verifier built on the [Viper](https://viper.ethz.ch) infrastructure.

The plugin provides:

- **Skills** — `verifying` (the workflow guide and entrypoint), `nagini-language` (language and tooling reference with verified examples), `spec-quality` (specification quality principles), and `handling-verification-errors` (debugging playbook).
- **Agents** — `spec-designer` (writes predicates, pure functions, and method contracts), `spec-critic` (independent spec review for strength and completeness), `method-verifier` (implements and proves one method at a time), and `red-flag-checker` (audits for patterns that undermine verification).
- **MCP tools** — `verify_method` / `verify_snippet` for in-session verification with caching, via a bundled Nagini MCP server launcher.

## Prerequisites

The plugin configures Claude Code; the verifier itself must be installed separately:

1. **Python 3** with the `nagini` package (use a version of Python supported by your Nagini release):

   ```sh
   python3 -m pip install "nagini[mcp]"
   ```

   The `[mcp]` extra pulls in the MCP SDK the in-session verification server needs.
We recommend installing Nagini in a virtualenv.

2. **A 64-bit Java runtime (JDK/JRE 11+)** for the Viper backend.

`nagini` may be installed globally or in a virtualenv. The plugin looks for the MCP server in this order:

1. `$NAGINI_MCP` — explicit path to a `nagini_mcp` executable
2. the active virtualenv (`$VIRTUAL_ENV`)
3. a virtualenv at the project root (`.venv` or `venv`)
4. `nagini_mcp` on `PATH`
5. `python3 -m nagini_translation.mcp_server`

## Installation

In Claude Code:

```
/plugin marketplace add viperproject/nagini-claude-plugin
/plugin install nagini@viperproject
```

Skills are then available under the `nagini:` namespace (e.g. `/nagini:verifying`), agents by their plain names, and the MCP verification tools as `mcp__nagini__verify_method` / `mcp__nagini__verify_snippet`.

To update later, run `/plugin marketplace update viperproject`. Auto-update is off by default for third-party marketplaces. The plugin's version tracks the git commit, so every push to this repository is picked up by the next update.

## Usage

Ask Claude to verify, prove, or add specifications to Python code — the `verifying` skill triggers and guides the workflow (spec design → spec review → per-method implement & prove → final red-flag audit). Skills can also be invoked directly (e.g. `/nagini:verifying`).

For hard methods, include a log path when dispatching `method-verifier` — the agent records each verification attempt there, and passing the prior session's log into a re-dispatch lets it continue where it left off instead of repeating failed strategies.

## Development

```sh
# validate manifest and structure
claude plugin validate ./plugin

# run a session with the local checkout instead of the installed version
claude --plugin-dir /path/to/nagini-claude-plugin/plugin
```

Inside a session, `/reload-plugins` picks up local edits without restarting. Alternatively, add the checkout as a local marketplace: `/plugin marketplace add ./nagini-claude-plugin`.

### Docker

`docker/Dockerfile` builds a clean environment with Python 3.12, Java 21, nagini (git master), and Claude Code — useful for trying the plugin without a local Nagini install, and as the base for tests:

```sh
docker build -t nagini-plugin-dev docker/
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
- `plugin/bin/nagini-mcp` — launcher that resolves the Nagini MCP server (resolution order above) and reports missing prerequisites
- `plugin/skills/<name>/SKILL.md` — skills, with supporting material in `references/` and `examples/`
- `plugin/agents/<name>.md` — subagents

## Troubleshooting

The MCP server's error output appears in `/mcp` (or the `/plugin` errors view).

- **"no Java runtime found"** — install a 64-bit JDK/JRE 11+ and make sure `java` is on the `PATH` of the shell you launch Claude Code from, or set `JAVA_HOME`.
- **"could not find the Nagini MCP server" although nagini is installed** — either your nagini release predates the MCP server (`python3 -m pip install --upgrade "nagini[mcp]"`), or it lives in a virtualenv the launcher does not detect (see the resolution order above).
- **`ModuleNotFoundError: No module named 'mcp'`** — nagini was installed without the MCP extra: `python3 -m pip install "nagini[mcp]"`.
- **Virtualenv not picked up** — the launcher inherits its environment from the process Claude Code was started from. Launch `claude` from a shell with the venv activated, keep the venv at `<project>/.venv`, or set `NAGINI_MCP=/path/to/venv/bin/nagini_mcp` in your shell profile.
- **`pip: command not found`** — install pip first (e.g. `apt install python3-pip`) or use `python3 -m ensurepip`.
- **Permission rules for the verify tools** — the tools appear as `mcp__nagini__verify_method` / `mcp__nagini__verify_snippet`, but permission rules (in `settings.json`, `--allowedTools`, etc.) must use the plugin-namespaced form, e.g. `mcp__plugin_nagini_nagini__verify_method`.

## License

[MPL-2.0](LICENSE)

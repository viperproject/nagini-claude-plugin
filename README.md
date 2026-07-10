# Nagini Claude Plugin

A [Claude Code](https://claude.com/claude-code) plugin for formally verifying Python programs with [Nagini](https://github.com/marcoeilers/nagini), the Python verifier built on the [Viper](https://viper.ethz.ch) infrastructure.

The plugin provides:

- **Skills** — a Nagini language and tooling reference, specification-quality principles, a verification-error debugging playbook, and a guide to running full verification workflows.
- **Agents** — specialized subagents for interface design, specification design and critique, spec testing, implementation/proof (`method-verifier`), and an adversarial red-flag audit of verification results.
- **MCP tools** — `verify_method` / `verify_snippet` for in-session verification with caching, via a bundled Nagini MCP server launcher.

## Prerequisites

The plugin configures Claude Code; the verifier itself must be installed separately:

1. **Python 3** with the `nagini` package (use a version of Python supported by your Nagini release):

   ```sh
   python3 -m pip install nagini
   ```
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

Skills are then available under the `nagini:` namespace (e.g. `/nagini:nagini-language`), agents by their plain names, and the MCP verification tools as `mcp__nagini__verify_method` / `mcp__nagini__verify_snippet`.

## Development

```sh
# validate manifest and structure
claude plugin validate .

# run a session with the local checkout instead of the installed version
claude --plugin-dir /path/to/nagini-claude-plugin
```

Inside a session, `/reload-plugins` picks up local edits without restarting. Alternatively, add the checkout as a local marketplace: `/plugin marketplace add ./nagini-claude-plugin`.

Layout:

- `.claude-plugin/plugin.json` — plugin manifest (no `version` field: versions track git commits while under active development)
- `.claude-plugin/marketplace.json` — same-repo marketplace (`viperproject`)
- `.mcp.json` — MCP server wiring, pointing at `bin/nagini-mcp`
- `bin/nagini-mcp` — launcher that resolves the Nagini MCP server (resolution order above) and reports missing prerequisites
- `skills/`, `agents/` — plugin components (see the READMEs inside)

## Troubleshooting

The MCP server's error output appears in `/mcp` (or the `/plugin` errors view).

- **"no Java runtime found"** — install a 64-bit JDK/JRE 11+ and make sure `java` is on the `PATH` of the shell you launch Claude Code from, or set `JAVA_HOME`.
- **"could not find the Nagini MCP server" although nagini is installed** — either your nagini release predates the MCP server (`python3 -m pip install --upgrade nagini`), or it lives in a virtualenv the launcher does not detect (see the resolution order above).
- **Virtualenv not picked up** — the launcher inherits its environment from the process Claude Code was started from. Launch `claude` from a shell with the venv activated, keep the venv at `<project>/.venv`, or set `NAGINI_MCP=/path/to/venv/bin/nagini_mcp` in your shell profile.
- **`pip: command not found`** — install pip first (e.g. `apt install python3-pip`) or use `python3 -m ensurepip`.

## License

[MPL-2.0](LICENSE)

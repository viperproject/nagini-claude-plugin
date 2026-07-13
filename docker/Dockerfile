# Clean environment for trying out and testing the Nagini plugin:
# Java 21 + uv (cache pre-warmed with the plugin's pinned nagini) + Claude Code.
# The plugin is not baked in — mount the repo and pass --plugin-dir (see README).
# Build from the repo root: docker build -f docker/Dockerfile -t nagini-plugin-dev .

FROM python:3.12-slim

RUN apt-get update \
    && apt-get install -y --no-install-recommends openjdk-21-jre-headless git curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

RUN curl -LsSf https://astral.sh/uv/install.sh | sh \
    && curl -fsSL https://claude.ai/install.sh | bash
ENV PATH="/root/.local/bin:${PATH}"

# Pre-warm uv's cache with the toolchain pinned in the launcher (the single
# source of truth for the nagini and Python versions).
COPY plugin/bin/nagini-mcp /opt/nagini-mcp-launcher
RUN NAGINI_VERSION=$(sed -n 's/^NAGINI_VERSION="\${NAGINI_VERSION:-\(.*\)}"$/\1/p' /opt/nagini-mcp-launcher) \
    && NAGINI_PYTHON=$(sed -n 's/^NAGINI_PYTHON="\${NAGINI_PYTHON:-\(.*\)}"$/\1/p' /opt/nagini-mcp-launcher) \
    && test -n "$NAGINI_VERSION" && test -n "$NAGINI_PYTHON" \
    && uvx --python "$NAGINI_PYTHON" --from "nagini[mcp]==$NAGINI_VERSION" python -c "import nagini_translation.mcp_server"

WORKDIR /work
CMD ["bash"]

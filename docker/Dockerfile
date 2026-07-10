# Clean environment for trying out and testing the Nagini plugin:
# Python 3.12 + Java 21 + nagini (git master, includes the MCP server) + Claude Code.
# The plugin is not baked in — mount the repo and pass --plugin-dir (see README).

FROM python:3.12-slim

RUN apt-get update \
    && apt-get install -y --no-install-recommends openjdk-21-jre-headless git curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# nagini from master until a PyPI release ships the MCP server
RUN pip install --no-cache-dir "nagini[mcp] @ git+https://github.com/marcoeilers/nagini.git"

RUN curl -fsSL https://claude.ai/install.sh | bash
ENV PATH="/root/.local/bin:${PATH}"

WORKDIR /work
CMD ["bash"]

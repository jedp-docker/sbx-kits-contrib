# syntax=docker/dockerfile:1
# Pre-baked image for the chrome-devtools-mcp kit.
#
# Extends the official Claude sandbox template with Playwright Chromium and
# libnss3-tools pre-installed, so sandbox creation skips the slow download.
#
# Build (run once, from sbx-kits-contrib/):
#   docker build -t claude-chrome:latest chrome-devtools-mcp/
#
# Use:
#   sbx run claude --template claude-chrome:latest \
#     --kit ./chrome-devtools-mcp/ .

FROM docker/sandbox-templates:claude-code-docker

ENV PLAYWRIGHT_BROWSERS_PATH=/opt/playwright-browsers

USER root

RUN --mount=type=cache,target=/root/.npm \
    mkdir -p /opt/playwright-browsers && \
    mkdir -p /usr/local/share/npm-global/lib && \
    npx --yes playwright@latest install --with-deps chromium && \
    apt-get install -y --no-install-recommends libnss3-tools && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Resolve and persist the Chromium binary path so the wrapper finds it at runtime.
RUN CHROME_BIN=$(find /opt/playwright-browsers -name chrome -o -name chromium 2>/dev/null | head -1) && \
    echo "export PLAYWRIGHT_BROWSERS_PATH=/opt/playwright-browsers" >> /etc/sandbox-persistent.sh && \
    echo "export CDMCP_CHROME_PATH=${CHROME_BIN}" >> /etc/sandbox-persistent.sh

USER agent

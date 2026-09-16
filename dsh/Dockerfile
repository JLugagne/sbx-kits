# syntax=docker/dockerfile:1
#
# Base image for the dsh sandbox kit.
#
# Extends the generic "shell" sandbox template (Ubuntu-based, non-root
# "agent" user at UID 1000, passwordless sudo, /home/agent home) since
# DeepSeek Harness (dsh) is not one of sbx's built-in agent variants.
FROM docker/sandbox-templates:shell

# --- Node.js (dsh requires Node.js >= 20) --------------------------------
RUN sudo apt-get update \
    && sudo apt-get install -y --no-install-recommends curl ca-certificates \
    && curl -fsSL https://deb.nodesource.com/setup_lts.x | sudo -E bash - \
    && sudo apt-get install -y --no-install-recommends nodejs \
    && sudo rm -rf /var/lib/apt/lists/*

# --- DeepSeek Harness (dsh) ------------------------------------------------
RUN sudo npm install -g @deepseek-ai/dsh

# --- Disable telemetry at the image level ---------------------------------
# DSH_TELEMETRY_DISABLED: any non-empty value (even "0"/"false") is an
#   authoritative hard opt-out — wins over DSH_TELEMETRY_MODE and any
#   project-level .env loaded later.
# DSH_TELEMETRY_MODE=DISABLED: explicit belt-and-suspenders, in case
#   DSH_TELEMETRY_DISABLED is ever unset at runtime.
# The kit's spec.yaml also denies the telemetry collector's domain at the
# network layer (permissions.network.deny), so this is defense in depth,
# not the only guardrail.
ENV DSH_TELEMETRY_DISABLED=1 \
    DSH_TELEMETRY_MODE=DISABLED

LABEL com.docker.sandboxes.flavor=dsh

WORKDIR /home/agent/workspace

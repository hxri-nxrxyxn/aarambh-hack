# --- Stage 1: Get the FFmpeg binary ---
# We use a special image that contains a standalone, static FFmpeg file
FROM mwader/static-ffmpeg:latest AS ffmpeg-source

# --- Stage 2: Build the n8n image ---
FROM n8nio/n8n:latest

USER root

# 1. Copy FFmpeg directly from Stage 1 (No apt-get/apk required!)
COPY --from=ffmpeg-source /ffmpeg /usr/local/bin/
COPY --from=ffmpeg-source /ffprobe /usr/local/bin/

# 2. Install Node libraries
# We use --unsafe-perm to prevent permission errors with global installs
RUN npm install -g axios moment lodash fluent-ffmpeg --unsafe-perm

# Switch back to n8n user
USER node

EXPOSE 5678

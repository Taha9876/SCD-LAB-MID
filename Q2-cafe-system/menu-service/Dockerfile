# FastBite menu-service - runs on its own, in its own image, on port 3001
FROM node:22-alpine

ENV NODE_ENV=production \
    PORT=3001
WORKDIR /app

# Install the exact versions from the lockfile first so this layer is cached
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY index.js ./

# Run as the unprivileged "node" user that ships with the base image
USER node
EXPOSE 3001

# The container checks its own /health endpoint
HEALTHCHECK --interval=10s --timeout=3s --start-period=15s --retries=5 \
  CMD wget -qO- "http://127.0.0.1:${PORT}/health" > /dev/null || exit 1

CMD ["node", "index.js"]

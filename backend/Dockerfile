# ─── Stage 1: Build / Install Dependencies ───────────────────────────────────
FROM node:20-alpine AS deps

WORKDIR /app

# Copy only dependency manifests first (better layer caching)
COPY package.json package-lock.json ./

# Install production dependencies only
RUN npm ci --omit=dev

# ─── Stage 2: Production Image ────────────────────────────────────────────────
FROM node:20-alpine AS runner

WORKDIR /app

# Create a non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy installed node_modules from deps stage
COPY --from=deps /app/node_modules ./node_modules

# Copy application source (exclude .env – use Docker secrets / env vars at runtime)
COPY . .

# Ensure the non-root user owns the app directory
RUN chown -R appuser:appgroup /app

USER appuser

# Expose the backend port
EXPOSE 5000

# Health check so Docker / Compose knows when the container is ready
HEALTHCHECK --interval=30s --timeout=10s --start-period=15s --retries=3 \
  CMD wget -qO- http://localhost:5000/health || exit 1

CMD ["node", "app.js"]

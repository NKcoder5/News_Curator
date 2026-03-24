# ─── Stage 1: Install Dependencies ───────────────────────────────────────────
FROM node:20-alpine AS deps

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci

# ─── Stage 2: Build the React App ─────────────────────────────────────────────
FROM node:20-alpine AS builder

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build args that get baked into the static bundle at build time
# Pass these via docker build --build-arg or docker-compose build args
ARG REACT_APP_OPENWEATHER_API_KEY
ENV REACT_APP_OPENWEATHER_API_KEY=$REACT_APP_OPENWEATHER_API_KEY

# REACT_APP_API_URL lets the frontend know the backend address at runtime
ARG REACT_APP_API_URL=http://localhost:5000
ENV REACT_APP_API_URL=$REACT_APP_API_URL

RUN npm run build

# ─── Stage 3: Serve via Nginx ─────────────────────────────────────────────────
FROM nginx:stable-alpine AS runner

# Remove default Nginx config and copy our custom one
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy the built React app
COPY --from=builder /app/build /usr/share/nginx/html

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -qO- http://localhost:80 || exit 1

CMD ["nginx", "-g", "daemon off;"]

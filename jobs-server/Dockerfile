# syntax=docker/dockerfile:1

# ---- dependencies -----------------------------------------------------------
FROM node:22-alpine AS deps
WORKDIR /app
RUN apk add --no-cache python3 make g++
COPY package.json package-lock.json ./
RUN npm ci

# ---- build ------------------------------------------------------------------
FROM deps AS build
WORKDIR /app
COPY tsconfig.json tsconfig.build.json ./
COPY prisma ./prisma
COPY src ./src
RUN node node_modules/prisma/build/index.js generate && npm run build

# ---- production dependencies only -------------------------------------------
FROM node:22-alpine AS prod-deps
WORKDIR /app
RUN apk add --no-cache python3 make g++
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY prisma ./prisma

# ---- runtime ----------------------------------------------------------------
FROM node:22-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production

RUN apk add --no-cache tini wget

COPY --from=prod-deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules/.prisma ./node_modules/.prisma
COPY --from=build /app/node_modules/@prisma ./node_modules/@prisma
COPY package.json ./
COPY prisma ./prisma

# The image runs as the unprivileged `node` user that the base image already provides.
USER node

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=20s --retries=3 \
  CMD wget --quiet --tries=1 --spider "http://127.0.0.1:${PORT:-8080}/api/health" || exit 1

# tini reaps zombies and forwards signals, so the graceful shutdown in server.ts runs.
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/server.js"]

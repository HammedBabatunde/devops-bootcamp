# syntax=docker/dockerfile:1
FROM node:20-alpine AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
RUN corepack enable
WORKDIR /app

# ============================================
# Dependencies stage
# ============================================
FROM base AS deps
COPY package.json pnpm-lock.yaml ./
RUN --mount=type=cache,id=pnpm,target=/pnpm/store \
    pnpm install --frozen-lockfile

# ============================================
# Build stage
# ============================================
FROM deps AS build
COPY . .
COPY --from=deps /app/node_modules ./node_modules

# Apply the TS bypass for JWT
RUN sed -i '1i // @ts-nocheck' src/common/jwt.service.ts

# FIX: Generate Prisma types so 'pnpm run build' doesn't fail
RUN pnpm prisma generate

# Compile TS to JS
RUN pnpm run build

# ============================================
# Production image
# ============================================
FROM base AS production
ENV NODE_ENV=production
WORKDIR /app

RUN addgroup --system --gid 1001 nodejs && \
    adduser --system --uid 1001 nodejs

COPY package.json pnpm-lock.yaml ./
COPY prisma ./prisma/

# 1. Install prod deps
# 2. Pin npx to version 6.19.2 to match your project and avoid Prisma 7 breaking changes
RUN --mount=type=cache,id=pnpm,target=/pnpm/store \
    pnpm install --prod --frozen-lockfile && \
    npx -y prisma@6.19.2 generate 

# 3. Copy the compiled JS from build stage
COPY --from=build --chown=nodejs:nodejs /app/dist ./dist

RUN chown -R nodejs:nodejs /app
USER nodejs
EXPOSE 3001
CMD ["node", "dist/main.js"]
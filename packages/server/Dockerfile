FROM node:20-alpine AS deps
WORKDIR /app

# Install pnpm
RUN npm install -g pnpm@9.15.9

# Copy root workspace files
COPY package.json pnpm-workspace.yaml pnpm-lock.yaml ./

# Copy package.jsons for workspace dependencies
COPY packages/server/package.json packages/server/package.json
COPY packages/web/package.json packages/web/package.json

# Install dependencies (frozen lockfile)
RUN pnpm install --frozen-lockfile || pnpm install

# ====== Build stage ======
FROM node:20-alpine AS build
WORKDIR /app

RUN npm install -g pnpm@9.15.9

# Copy deps from previous stage
COPY --from=deps /app/node_modules ./node_modules
COPY --from=deps /app/packages/server/node_modules ./packages/server/node_modules
COPY --from=deps /app/packages/web/node_modules ./packages/web/node_modules

# Copy source files
COPY package.json pnpm-workspace.yaml pnpm-lock.yaml ./
COPY packages/server ./packages/server
COPY packages/web ./packages/web

# Build web frontend (needed for the public/web directory)
RUN cd packages/web && pnpm build

# Build server TypeScript
RUN cd packages/server && pnpm build

# Generate Prisma client
RUN cd packages/server && npx prisma generate

# ====== Runtime stage ======
FROM node:20-alpine AS runtime
WORKDIR /app

ENV NODE_ENV=production
ENV PORT=4000

# Copy only necessary runtime files
COPY --from=build /app/packages/server/package.json ./
COPY --from=build /app/packages/server/package-lock.json ./
COPY --from=build /app/packages/server/dist ./dist
COPY --from=build /app/packages/server/public ./public
COPY --from=build /app/packages/server/prisma ./prisma
COPY --from=build /app/packages/server/node_modules ./node_modules

# Expose port
EXPOSE 4000

# Start server: prisma db push (safe, no reset) + node dist/app.js
CMD ["sh", "-c", "npx prisma@5.22.0 db push && node dist/app.js"]

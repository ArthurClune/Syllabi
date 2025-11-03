FROM node:20-alpine AS base
 
# Install dependencies only when needed
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
 
# Copy package files
COPY package.json package-lock.json* ./
RUN npm ci --legacy-peer-deps
 
# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
 
# Build arguments for environment variables
ARG NEXT_PUBLIC_SUPABASE_URL
ARG NEXT_PUBLIC_SUPABASE_ANON_KEY
ARG NEXT_PUBLIC_BACKEND_URL
ARG NEXT_PUBLIC_APP_URL
 
# Set environment variables for build
ENV NEXT_PUBLIC_SUPABASE_URL=$NEXT_PUBLIC_SUPABASE_URL
ENV NEXT_PUBLIC_SUPABASE_ANON_KEY=$NEXT_PUBLIC_SUPABASE_ANON_KEY
ENV NEXT_PUBLIC_BACKEND_URL=$NEXT_PUBLIC_BACKEND_URL
ENV NEXT_PUBLIC_APP_URL=$NEXT_PUBLIC_APP_URL
 
# Build Next.js with increased memory limit
ENV NODE_OPTIONS="--max-old-space-size=4096"
RUN npm run build
 
# Production image
FROM base AS runner
WORKDIR /app
 
ENV NODE_ENV=production
 
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs
 
# Copy built application
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
 
USER nextjs
 
EXPOSE 3000
 
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"
 
CMD ["node", "server.js"]
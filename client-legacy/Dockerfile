# PayMate PSP + admin (legacy) client — Vite static build served by nginx.
# Multi-stage: build the bundle in-image (no committed dist/ needed), then
# serve it with nginx. All VITE_* must be present at BUILD time (Vite bakes
# them into the static bundle).
FROM node:20-alpine AS build
WORKDIR /app

COPY package.json package-lock.json ./
# npm install (not ci) — tolerant of lockfile drift across npm major versions
# (the runner's node:20-alpine ships npm 10; locks may be generated with 11).
# --ignore-scripts skips native-module compiles (usb/node-hid/keccak, pulled in
# by wallet libs) that need Python + a C toolchain absent from alpine. Those are
# Node-only hardware-wallet transports, unused in the browser bundle. esbuild's
# Linux binary still resolves via its @esbuild/linux-x64 optional package.
RUN npm install --no-fund --no-audit --ignore-scripts

COPY . .

# Vite build-time env (baked into the static bundle). Provided via
# docker-compose build args or CI --build-arg.
ARG VITE_API_URL
ARG VITE_CHAIN_ID
ARG VITE_CHAIN_NAME
ARG VITE_RPC_URL
ARG VITE_STABLECOIN_ADDRESS
ARG VITE_FACTORY_ADDRESS
ARG VITE_WALLETCONNECT_PROJECT_ID

ENV VITE_API_URL=${VITE_API_URL}
ENV VITE_CHAIN_ID=${VITE_CHAIN_ID}
ENV VITE_CHAIN_NAME=${VITE_CHAIN_NAME}
ENV VITE_RPC_URL=${VITE_RPC_URL}
ENV VITE_STABLECOIN_ADDRESS=${VITE_STABLECOIN_ADDRESS}
ENV VITE_FACTORY_ADDRESS=${VITE_FACTORY_ADDRESS}
ENV VITE_WALLETCONNECT_PROJECT_ID=${VITE_WALLETCONNECT_PROJECT_ID}

# Base path for serving under /admin behind the ingress. Defaults to '/'
# so nothing breaks if built standalone; CI passes --build-arg VITE_BASE=/admin/.
ARG VITE_BASE=/
# The wallet-SDK dependency graph (metamask-sdk, reown/appkit, walletconnect,
# coinbase) exhausts Node's default heap during Rollup bundling — raise it.
ENV NODE_OPTIONS=--max-old-space-size=6144
RUN npm run build -- --base=${VITE_BASE}

# ── nginx stage ──────────────────────────────────────────────
FROM nginx:1.25.1

# React Router needs the SPA fallback (serve index.html for unknown routes).
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx/nginx.conf /etc/nginx/conf.d
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

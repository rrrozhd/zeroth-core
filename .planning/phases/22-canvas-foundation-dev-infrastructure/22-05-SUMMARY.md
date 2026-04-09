---
phase: 22-canvas-foundation-dev-infrastructure
plan: 05
subsystem: infra
tags: [docker, nginx, vue, spa, multi-stage-build, reverse-proxy]

requires:
  - phase: 22-01
    provides: Vue 3 + Vite scaffold with build toolchain
  - phase: 22-02
    provides: Interactive canvas and node components to serve

provides:
  - Multi-stage Dockerfile for Studio frontend (Node build + Nginx serve)
  - Nginx configuration with SPA fallback, API proxy, and asset caching
  - Updated docker-compose.yml with Studio-integrated nginx service

affects: [deployment, production-serving, phase-23-inspector, phase-24-websocket]

tech-stack:
  added: [nginx-1.27-alpine, node-22-alpine, docker-multi-stage]
  patterns: [spa-nginx-fallback, reverse-proxy-api, immutable-asset-caching, dockerignore-optimization]

key-files:
  created:
    - apps/studio/Dockerfile
    - apps/studio/.dockerignore
    - docker/nginx/studio.conf
  modified:
    - docker-compose.yml

key-decisions:
  - "Modified existing nginx service rather than adding separate studio-nginx service (cleaner per D-11)"
  - "Nginx config mounted as volume for flexibility rather than baked into image"
  - "Static assets cached with 1y expiry and immutable header (Vite content-hashes filenames)"

patterns-established:
  - "Multi-stage Docker build: Node build stage + Nginx serve stage"
  - "SPA routing: try_files $uri $uri/ /index.html for client-side routing"
  - "API proxy: /api/, /health, /v1/ forwarded to FastAPI backend"

requirements-completed: [INFRA-01]

duration: 1min
completed: 2026-04-09
---

# Phase 22 Plan 05: Docker Multi-Container Deployment Summary

**Multi-stage Docker build serving Vue SPA via Nginx with API reverse proxy to FastAPI backend**

## What Was Built

### Task 1: Studio Dockerfile, Nginx Config, and docker-compose Update

Created the complete Docker deployment infrastructure for serving the Studio frontend:

1. **apps/studio/Dockerfile** - Two-stage build:
   - Stage 1: `node:22-alpine` installs dependencies (`npm ci`) and builds the Vue app (`npm run build`)
   - Stage 2: `nginx:1.27-alpine` copies built assets to `/usr/share/nginx/html/studio`

2. **docker/nginx/studio.conf** - Nginx server configuration:
   - Serves Vue SPA from `/usr/share/nginx/html/studio` at root URL
   - `try_files $uri $uri/ /index.html` for SPA client-side routing on refresh
   - Nested location block caches static assets (js, css, images, fonts) with `expires 1y` and `Cache-Control: public, immutable`
   - Proxies `/api/` requests to `http://zeroth:8000` (FastAPI backend)
   - Passes through `/health` and `/v1/` to FastAPI for existing endpoints
   - Gzip compression for text/css/json/js/xml responses

3. **apps/studio/.dockerignore** - Excludes `node_modules/`, `dist/`, `.git/`, `*.md`, `.env*` to prevent slow Docker builds

4. **docker-compose.yml** - Updated nginx service:
   - Changed from `image: nginx:1.27-alpine` to `build: ./apps/studio` (multi-stage Dockerfile)
   - Volume mounts `studio.conf` instead of `nginx.conf` as default Nginx config
   - Retains same ports (80, 443), depends_on, network, restart policy

## Deviations from Plan

None - plan executed exactly as written.

## Known Stubs

None - all files are complete production configurations.

## Commits

| Task | Commit | Description |
|------|--------|-------------|
| 1 | fc7f8f8 | feat(22-05): add Docker multi-container deployment for Studio SPA |

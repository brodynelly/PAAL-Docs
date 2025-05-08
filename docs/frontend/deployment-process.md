# Frontend Deployment Process

## Overview

The PAAL system frontend follows a structured deployment process to ensure reliable and consistent releases. This document outlines the build process, deployment environments, and CI/CD pipeline used for the frontend application.

## Build Process

### Build Steps

The frontend build process consists of the following steps:

1. **Code Compilation**: TypeScript code is compiled to JavaScript
2. **Bundling**: Next.js bundles the application code and dependencies
3. **Optimization**: Assets are optimized (minification, tree shaking, etc.)
4. **Static Generation**: Static pages are pre-rendered where applicable
5. **Output Generation**: Production-ready files are generated in the `.next` directory

### Build Configuration

The build is configured in `next.config.mjs`:

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  redirects: async () => [
    {
      source: "/",
      destination: "/overview",
      permanent: true,
    },
  ],
  images: {
    remotePatterns: [
      {
        protocol: "https",
        hostname: "images.unsplash.com",
        port: "",
        pathname: "/**",
      },
    ],
  },
  // Configure the server
  serverRuntimeConfig: {
    port: process.env.PORT || 3000,
  },
  // Enable output tracing for better debugging
  output: 'standalone',
  // Add API proxy configuration
  async rewrites() {
    return [
      {
        source: '/api/:path*',
        destination: 'http://backend:5005/api/:path*',
      },
    ];
  },
};

export default nextConfig;
```

### Build Commands

The build process is executed using the following npm scripts defined in `package.json`:

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "analyze": "ANALYZE=true next build"
  }
}
```

## Containerization

The frontend application is containerized using Docker for consistent deployment across environments.

### Development Dockerfile

For development, a multi-stage Dockerfile (`Dockerfile.dev`) is used:

```dockerfile
# Dockerfile.dev
FROM node:20-alpine AS base

# Enable Corepack and install pnpm
RUN corepack enable && corepack prepare pnpm@latest --activate

WORKDIR /usr/src/app

# Copy package files for dependency installation
COPY package*.json pnpm-lock.yaml ./

# Create a development stage for faster rebuilds
FROM base AS dev

# Install all dependencies
RUN pnpm install --no-frozen-lockfile

# Copy only the frontend code
COPY src ./src
COPY public ./public
COPY next.config.mjs ./next.config.mjs
COPY tailwind.config.ts ./tailwind.config.ts
COPY postcss.config.mjs ./postcss.config.mjs
COPY tsconfig.json ./tsconfig.json

#allow for listening on docker network, but don't expose to main server
EXPOSE 3000

# Start the development server
CMD ["pnpm", "run", "dev"]
```

### Production Dockerfile

For production, a separate Dockerfile (`Dockerfile.frontend`) is used:

```dockerfile
FROM node:20-alpine

# Enable Corepack and install pnpm
RUN corepack enable && corepack prepare pnpm@latest --activate

# Set the working directory
WORKDIR /usr/src/app

# Copy package files first for dependency caching
COPY package*.json pnmp-lock.yaml ./

# Install production dependencies
RUN pnpm install --production

# Copy the rest of your application code
COPY . .

# Build the Next.js application
RUN npm run build

# Expose the port Next.js will run on
EXPOSE 3000

# Start Next.js in production mode
CMD ["npm", "start"]
```

## Deployment Environments

The application is deployed to three main environments:

### Development Environment

- **Purpose**: For active development and testing new features
- **URL**: `https://dev.paal-system.example.com`
- **Deployment**: Automatic from the `develop` branch
- **Configuration**: Uses development API endpoints and feature flags

### Staging Environment

- **Purpose**: For pre-production testing and QA
- **URL**: `https://staging.paal-system.example.com`
- **Deployment**: Manual promotion from development
- **Configuration**: Mirrors production configuration but with staging API endpoints

### Production Environment

- **Purpose**: Live environment for end users
- **URL**: `https://paal-system.example.com`
- **Deployment**: Manual promotion from staging
- **Configuration**: Production API endpoints and optimized settings

## Environment Variables

Environment variables are used to configure the application for different environments:

```
# Base configuration
NEXT_PUBLIC_API_URL=http://localhost:8080
NEXT_PUBLIC_SOCKET_URL=http://localhost:8080
NEXT_PUBLIC_APP_ENV=development

# Feature flags
NEXT_PUBLIC_ENABLE_ANALYTICS=false
NEXT_PUBLIC_ENABLE_EXPERIMENTAL_FEATURES=true

# Authentication
NEXT_PUBLIC_AUTH_DOMAIN=auth.example.com
```

Environment variables are managed differently across environments:

- **Local Development**: `.env.local` file (not committed to version control)
- **Docker Development**: Environment variables in `docker-compose.yml`
- **CI/CD Environments**: Environment variables stored in CI/CD platform

## CI/CD Pipeline

The continuous integration and continuous deployment pipeline automates the build, test, and deployment process.

### CI/CD Workflow

1. **Code Commit**: Developer pushes code to a branch
2. **Automated Tests**: CI system runs linting, unit tests, and integration tests
3. **Build**: Application is built and containerized
4. **Deployment**: Container is deployed to the appropriate environment
5. **Smoke Tests**: Basic functionality tests are run post-deployment
6. **Monitoring**: Application is monitored for errors or performance issues

### GitHub Actions Workflow

The CI/CD pipeline is implemented using GitHub Actions:

```yaml
# .github/workflows/frontend-ci.yml
name: Frontend CI/CD

on:
  push:
    branches: [main, develop]
    paths:
      - 'src/**'
      - 'public/**'
      - 'package.json'
      - 'pnpm-lock.yaml'
      - 'next.config.mjs'
      - 'Dockerfile.frontend'
      - '.github/workflows/frontend-ci.yml'
  pull_request:
    branches: [main, develop]
    paths:
      - 'src/**'
      - 'public/**'
      - 'package.json'
      - 'pnpm-lock.yaml'
      - 'next.config.mjs'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'pnpm'
      
      - name: Install pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 8
          run_install: false
      
      - name: Install dependencies
        run: pnpm install
      
      - name: Lint
        run: pnpm run lint
      
      - name: Test
        run: pnpm run test
  
  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ghcr.io/${{ github.repository }}/frontend
          tags: |
            type=ref,event=branch
            type=sha,format=short
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          file: Dockerfile.frontend
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  deploy-dev:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: development
    steps:
      - name: Deploy to Development
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.DEV_HOST }}
          username: ${{ secrets.DEV_USERNAME }}
          key: ${{ secrets.DEV_SSH_KEY }}
          script: |
            cd /opt/paal
            docker-compose pull frontend
            docker-compose up -d frontend
  
  deploy-prod:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - name: Deploy to Production
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USERNAME }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            cd /opt/paal
            docker-compose pull frontend
            docker-compose up -d frontend
```

## Deployment Architecture

The frontend application is deployed as part of a larger system using Docker Compose:

```yaml
# docker-compose.yml (excerpt)
services:
  frontend:
    build:
      context: .
      dockerfile: Dockerfile.dev
      target: dev
    container_name: frontend
    depends_on:
      - backend
    networks:
      - app-net
    ports:
      - "3000:3000"
    env_file: .env
    environment:
      - PORT=3000
      - NEXT_PUBLIC_API_URL=http://localhost:8080
      - HOSTNAME=0.0.0.0
      - NODE_ENV=development
      - NEXT_TELEMETRY_DISABLED=1
    volumes:
      - ./src:/usr/src/app/src
      - ./public:/usr/src/app/public
      - ./next.config.mjs:/usr/src/app/next.config.mjs
      - ./tailwind.config.ts:/usr/src/app/tailwind.config.ts
      - ./postcss.config.mjs:/usr/src/app/postcss.config.mjs
      - ./tsconfig.json:/usr/src/app/tsconfig.json
      - frontend-node-modules:/usr/src/app/node_modules

  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    ports:
      - "8080:80"
    depends_on:
      - backend
      - frontend
    networks:
      - app-net
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
```

The NGINX configuration routes traffic to the appropriate services:

```nginx
# nginx.conf
server {
    listen 80;
    server_name localhost;

    # API requests
    location /api/ {
        proxy_pass http://backend:5005;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # Socket.IO
    location /socket.io/ {
        proxy_pass http://backend:5005;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # Frontend
    location / {
        proxy_pass http://frontend:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

## Deployment Monitoring

After deployment, the application is monitored using several tools:

### Error Tracking

Sentry is used for real-time error tracking:

```tsx
// src/app/layout.tsx
import * as Sentry from '@sentry/nextjs';

if (process.env.NEXT_PUBLIC_SENTRY_DSN) {
  Sentry.init({
    dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
    environment: process.env.NEXT_PUBLIC_APP_ENV,
    tracesSampleRate: 1.0,
  });
}
```

### Performance Monitoring

Performance is monitored using:

- **Lighthouse**: For page performance metrics
- **Web Vitals**: For core web vitals tracking
- **Custom Analytics**: For user interaction metrics

### Health Checks

A health check endpoint is implemented to verify the application is running correctly:

```tsx
// src/app/api/health/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  return NextResponse.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    version: process.env.NEXT_PUBLIC_APP_VERSION || 'unknown',
    environment: process.env.NEXT_PUBLIC_APP_ENV || 'development',
  });
}
```

## Rollback Procedure

In case of deployment issues, the following rollback procedure is implemented:

1. **Identify Issue**: Determine the nature and severity of the problem
2. **Decision**: Decide whether to fix forward or roll back
3. **Rollback Command**: Execute the rollback command to revert to the previous version:

```bash
# Rollback to previous version
docker-compose pull frontend:previous-tag
docker-compose up -d frontend
```

4. **Verification**: Verify the application is functioning correctly after rollback
5. **Root Cause Analysis**: Investigate the cause of the issue
6. **Fix**: Implement and test a fix before attempting redeployment

## Deployment Checklist

Before each production deployment, the following checklist is reviewed:

1. **All tests passing**: Verify all automated tests are passing
2. **Performance budget**: Ensure the application meets performance targets
3. **Accessibility compliance**: Verify accessibility requirements are met
4. **Browser compatibility**: Test on supported browsers
5. **Feature verification**: Manually verify key features
6. **Security scan**: Run security scanning tools
7. **Documentation**: Update documentation if needed
8. **Stakeholder approval**: Get sign-off from relevant stakeholders

## Deployment Schedule

- **Development**: Continuous deployment as features are completed
- **Staging**: Weekly deployments (typically on Tuesdays)
- **Production**: Bi-weekly deployments (typically on Thursdays)
- **Hotfixes**: As needed, following expedited testing and approval

---
name: Directus Development Workflow
description: Use this skill when setting up and maintaining professional Directus development workflows. Master project scaffolding, TypeScript configuration, testing strategies, continuous integration/deployment, Docker containerization, multi-environment management, and development best practices for building scalable Directus applications.
---

# Directus Development Workflow

## Overview

This skill provides comprehensive guidance for setting up and maintaining professional Directus development workflows. Master project scaffolding, TypeScript configuration, testing strategies, continuous integration/deployment, Docker containerization, multi-environment management, and development best practices for building scalable Directus applications.

## When to Use This Skill

* Setting up new Directus projects
* Configuring TypeScript for type safety
* Implementing testing strategies
* Setting up CI/CD pipelines
* Containerizing with Docker
* Managing multiple environments
* Implementing database migrations
* Setting up development tools
* Optimizing build processes
* Deploying to production

## Project Setup

### Step 1: Initialize Directus Project

```
mkdir my-directus-project && cd my-directus-project
npm init -y
npm install directus
npx directus init

# Project structure
my-directus-project/
├── .env
├── .gitignore
├── docker-compose.yml
├── package.json
├── tsconfig.json
├── uploads/
├── extensions/
│   ├── endpoints/
│   ├── hooks/
│   ├── interfaces/
│   ├── displays/
│   ├── layouts/
│   ├── modules/
│   ├── operations/
│   └── panels/
├── migrations/
├── seeders/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── scripts/
├── docs/
└── .github/
    └── workflows/
```

### Step 2: Environment Configuration

```
# .env.example

# Database
DB_CLIENT="pg"
DB_HOST="localhost"
DB_PORT="5432"
DB_DATABASE="directus"
DB_USER="directus"
DB_PASSWORD="directus"

# Security
KEY="your-random-secret-key"
SECRET="your-random-secret"

# Admin
ADMIN_EMAIL="admin@example.com"
ADMIN_PASSWORD="admin"

# Server
PUBLIC_URL="http://localhost:8055"
PORT=8055

# Storage
STORAGE_LOCATIONS="local,s3"
STORAGE_LOCAL_DRIVER="local"
STORAGE_LOCAL_ROOT="./uploads"

# S3 Storage (optional)
STORAGE_S3_DRIVER="s3"
STORAGE_S3_KEY="your-s3-key"
STORAGE_S3_SECRET="your-s3-secret"
STORAGE_S3_BUCKET="your-bucket"
STORAGE_S3_REGION="us-east-1"

# Email
EMAIL_TRANSPORT="sendgrid"
EMAIL_SENDGRID_API_KEY="your-sendgrid-key"
EMAIL_FROM="no-reply@example.com"

# Cache
CACHE_ENABLED="true"
CACHE_STORE="redis"
CACHE_REDIS="redis://localhost:6379"
CACHE_AUTO_PURGE="true"

# Rate Limiting
RATE_LIMITER_ENABLED="true"
RATE_LIMITER_STORE="redis"
RATE_LIMITER_POINTS="100"
RATE_LIMITER_DURATION="60"

# Extensions
EXTENSIONS_AUTO_RELOAD="true"

# Telemetry
TELEMETRY="false"

# AI Integration (custom)
OPENAI_API_KEY="your-openai-key"
ANTHROPIC_API_KEY="your-anthropic-key"
PINECONE_API_KEY="your-pinecone-key"
```

### Step 3: TypeScript Configuration

```
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "moduleResolution": "node",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@extensions/*": ["extensions/*"],
      "@utils/*": ["src/utils/*"],
      "@services/*": ["src/services/*"],
      "@types/*": ["src/types/*"]
    }
  },
  "include": ["src/**/*", "extensions/**/*", "tests/**/*"],
  "exclude": ["node_modules", "dist", "uploads"]
}
```

## Docker Configuration

### Development Docker Setup

```
# docker-compose.yml
version: '3.8'

services:
  database:
    image: postgis/postgis:15-alpine
    container_name: directus_database
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: directus
      POSTGRES_PASSWORD: directus
      POSTGRES_DB: directus
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U directus"]
      interval: 10s
      timeout: 5s
      retries: 5

  cache:
    image: redis:7-alpine
    container_name: directus_cache
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  directus:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: directus_app
    ports:
      - "8055:8055"
    volumes:
      - ./uploads:/directus/uploads
      - ./extensions:/directus/extensions
      - ./migrations:/directus/migrations
      - ./.env:/directus/.env
    environment:
      DB_CLIENT: pg
      DB_HOST: database
      DB_PORT: 5432
      DB_DATABASE: directus
      DB_USER: directus
      DB_PASSWORD: directus
      CACHE_ENABLED: "true"
      CACHE_STORE: redis
      CACHE_REDIS: redis://cache:6379
      EXTENSIONS_AUTO_RELOAD: "true"
    depends_on:
      database:
        condition: service_healthy
      cache:
        condition: service_healthy
    command: >
      sh -c "
        npx directus database install &&
        npx directus database migrate:latest &&
        npx directus start
      "

  adminer:
    image: adminer
    container_name: directus_adminer
    ports:
      - "8080:8080"

  mailhog:
    image: mailhog/mailhog
    container_name: directus_mailhog
    ports:
      - "1025:1025"
      - "8025:8025"

volumes:
  postgres_data:
  redis_data:
```

### Production Dockerfile

```
# Dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json tsconfig.json ./
RUN npm ci
COPY extensions ./extensions
COPY src ./src
RUN npm run build

FROM node:18-alpine
WORKDIR /directus
RUN npm install -g directus
COPY --from=builder /app/dist/extensions ./extensions
COPY --from=builder /app/package*.json ./
RUN npm ci --only=production
RUN mkdir -p uploads

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:8055/server/health', (r) => {r.statusCode === 200 ? process.exit(0) : process.exit(1)})"

EXPOSE 8055
CMD ["npx", "directus", "start"]
```

## Testing Strategy

### Unit Testing with Vitest

```
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import path from 'path';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
    },
    setupFiles: ['./tests/setup.ts'],
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

### Integration Testing

```
// tests/integration/api.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { createDirectus, rest, authentication, createItems, readItems } from '@directus/sdk';

describe('API Integration Tests', () => {
  let client: any;

  beforeAll(async () => {
    client = createDirectus('http://localhost:8055').with(authentication()).with(rest());
    await client.login('admin@example.com', 'admin');
  });

  it('should create and read items', async () => {
    const created = await client.request(createItems('articles', {
      title: 'Test Article',
      content: 'Test content',
      status: 'published',
    }));

    expect(created).toHaveProperty('id');

    const items = await client.request(readItems('articles', {
      filter: { id: { _eq: created.id } },
    }));

    expect(items).toHaveLength(1);
    expect(items[0].title).toBe('Test Article');
  });
});
```

### End-to-End Testing with Playwright

```
// tests/e2e/admin-panel.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Directus Admin Panel', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('http://localhost:8055/admin');
    await page.fill('input[type="email"]', 'admin@example.com');
    await page.fill('input[type="password"]', 'admin');
    await page.click('button[type="submit"]');
    await page.waitForURL('**/content');
  });

  test('should create new article', async ({ page }) => {
    await page.click('text=Articles');
    await page.click('button:has-text("Create Item")');
    await page.fill('input[name="title"]', 'Test Article from E2E');
    await page.fill('textarea[name="content"]', 'This is test content');
    await page.click('button:has-text("Save")');
    await expect(page.locator('text=Item created')).toBeVisible();
  });
});
```

## CI/CD Pipeline

### GitHub Actions Workflow

```
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  release:
    types: [created]

env:
  NODE_VERSION: '18'
  DOCKER_REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: directus
          POSTGRES_PASSWORD: directus
          POSTGRES_DB: directus_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
        ports:
          - 6379:6379
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm test -- --coverage
        env:
          DB_CLIENT: pg
          DB_HOST: localhost
          DB_DATABASE: directus_test
          DB_USER: directus
          DB_PASSWORD: directus

  build:
    needs: [lint, test]
    runs-on: ubuntu-latest
    if: github.event_name == 'push' || github.event_name == 'release'
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.DOCKER_REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.DOCKER_REGISTRY }}/${{ env.IMAGE_NAME }}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-production:
    needs: build
    runs-on: ubuntu-latest
    if: github.event_name == 'release'
    environment:
      name: production
      url: https://example.com
    steps:
      - run: echo "Deploying to production..."
      - run: curl -f https://example.com/server/health
```

## Database Migration Management

### Migration Scripts

```
// migrations/001_create_custom_tables.ts
import { Knex } from 'knex';

export async function up(knex: Knex): Promise<void> {
  await knex.schema.createTable('custom_analytics', (table) => {
    table.uuid('id').primary().defaultTo(knex.raw('gen_random_uuid()'));
    table.string('event_type', 50).notNullable();
    table.string('event_category', 50);
    table.jsonb('event_data');
    table.uuid('user_id').references('id').inTable('directus_users');
    table.timestamp('created_at').defaultTo(knex.fn.now());
    table.index(['event_type', 'created_at']);
    table.index('user_id');
  });

  await knex.schema.createTable('custom_settings', (table) => {
    table.string('key', 100).primary();
    table.jsonb('value').notNullable();
    table.string('type', 20).defaultTo('string');
    table.text('description');
    table.timestamps(true, true);
  });
}

export async function down(knex: Knex): Promise<void> {
  await knex.schema.dropTableIfExists('custom_settings');
  await knex.schema.dropTableIfExists('custom_analytics');
}
```

## Performance Monitoring

```
// src/monitoring/apm.ts
import { performance } from 'perf_hooks';

export class PerformanceMonitor {
  private metrics: Map<string, number[]> = new Map();

  startTimer(operation: string): () => void {
    const start = performance.now();
    return () => {
      const duration = performance.now() - start;
      this.recordMetric(operation, duration);
      if (duration > 1000) console.warn(`Slow operation: ${operation} took ${duration.toFixed(2)}ms`);
    };
  }

  recordMetric(name: string, value: number): void {
    if (!this.metrics.has(name)) this.metrics.set(name, []);
    const values = this.metrics.get(name)!;
    values.push(value);
    if (values.length > 100) values.shift();
  }

  getStats(name: string): { avg: number; min: number; max: number; p95: number } | null {
    const values = this.metrics.get(name);
    if (!values || values.length === 0) return null;
    const sorted = [...values].sort((a, b) => a - b);
    const sum = sorted.reduce((a, b) => a + b, 0);
    return {
      avg: sum / sorted.length,
      min: sorted[0],
      max: sorted[sorted.length - 1],
      p95: sorted[Math.floor(sorted.length * 0.95)],
    };
  }
}
```

## Best Practices

### Security

```
// src/security/security-middleware.ts
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';

export function setupSecurity(app: any): void {
  app.use(helmet({ contentSecurityPolicy: { directives: { defaultSrc: ["'self'"] } } }));

  const limiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 100 });
  app.use('/api/', limiter);

  const authLimiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 5, skipSuccessfulRequests: true });
  app.use('/auth/', authLimiter);
}
```

## Success Metrics

* ✅ Development environment setup < 5 minutes
* ✅ TypeScript compilation with zero errors
* ✅ Test coverage > 80%
* ✅ CI/CD pipeline execution < 10 minutes
* ✅ Docker build size < 500MB
* ✅ Zero-downtime deployments
* ✅ Database migrations rollback capability
* ✅ Monitoring alerts < 1 minute response time
* ✅ Security scanning passes all checks
* ✅ Performance benchmarks meet SLA requirements

## Resources

* [Directus Documentation](https://docs.directus.io)
* [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
* [GitHub Actions](https://docs.github.com/en/actions)
* [Kubernetes Documentation](https://kubernetes.io/docs/)
* [Vitest Testing Framework](https://vitest.dev/)
* [Playwright E2E Testing](https://playwright.dev/)
* [TypeScript Handbook](https://www.typescriptlang.org/docs/)

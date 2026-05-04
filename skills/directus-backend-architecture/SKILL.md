---
name: Directus Backend Architecture
description: Use this skill for deep expertise in Directus backend architecture, covering API endpoint extensions, hook systems, service layers, flows and automation, database operations, authentication, and performance optimization. Master the TypeScript/Node.js backend to build scalable, secure, and efficient Directus applications.
---

# Directus Backend Architecture

## Overview

This skill provides deep expertise in Directus backend architecture, covering API endpoint extensions, hook systems, service layers, flows and automation, database operations, authentication, and performance optimization. Master the TypeScript/Node.js backend to build scalable, secure, and efficient Directus applications.

## When to Use This Skill

* Creating custom API endpoints and routes
* Implementing business logic with hooks
* Building automation workflows with Flows
* Extending authentication and permissions
* Optimizing database queries and performance
* Creating custom services and providers
* Implementing real-time features
* Building data migration and seeding scripts
* Integrating third-party services

## Core Architecture

### Directus Stack

* **Runtime**: Node.js (v18+)
* **Framework**: Express.js
* **Language**: TypeScript
* **Database**: Knex.js query builder
* **ORM**: Custom abstraction layer
* **Cache**: Redis/Memory
* **Queue**: Bull/Redis
* **WebSockets**: Socket.io

### Directory Structure

```
directus/
├── api/
│   └── src/
│       ├── services/
│       ├── controllers/
│       ├── middleware/
│       ├── database/
│       ├── utils/
│       └── types/
├── extensions/
│   ├── endpoints/
│   ├── hooks/
│   ├── operations/
│   └── services/
└── shared/
```

## Process: Creating Custom API Endpoints

### Step 1: Initialize Endpoint Extension

```
npx create-directus-extension@latest
# Select: endpoint > my-custom-api > typescript
```

### Step 2: Implement Endpoint Logic

```
// src/index.ts
import { defineEndpoint } from '@directus/extensions-sdk';
import Joi from 'joi';

export default defineEndpoint((router, context) => {
  const { services, database, getSchema, env, logger, emitter } = context;
  const { ItemsService, MailService } = services;

  const createSchema = Joi.object({
    title: Joi.string().required().min(3).max(255),
    content: Joi.string().required(),
    status: Joi.string().valid('draft', 'published').default('draft'),
    tags: Joi.array().items(Joi.string()),
    metadata: Joi.object(),
  });

  // GET /custom/analytics
  router.get('/analytics', async (req, res, next) => {
    try {
      if (!req.accountability?.user) {
        return res.status(401).json({ error: 'Unauthorized' });
      }

      const results = await database
        .select(
          database.raw('DATE(created_at) as date'),
          database.raw('COUNT(*) as count'),
          database.raw('AVG(amount) as avg_amount')
        )
        .from('orders')
        .where('status', 'completed')
        .whereRaw('created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)')
        .groupBy(database.raw('DATE(created_at)'))
        .orderBy('date', 'desc');

      return res.json({
        data: {
          daily: results,
          total: results.reduce((sum, day) => sum + day.count, 0),
          average: results.reduce((sum, day) => sum + day.avg_amount, 0) / results.length,
          period: '30_days',
        },
      });
    } catch (error) {
      logger.error('Analytics endpoint error:', error);
      return next(error);
    }
  });

  // POST /custom/process
  router.post('/process', async (req, res, next) => {
    try {
      const { error, value } = createSchema.validate(req.body);
      if (error) return res.status(400).json({ error: 'Validation Error', details: error.details });

      const itemsService = new ItemsService('articles', {
        schema: await getSchema(),
        accountability: req.accountability,
      });

      const result = await database.transaction(async (trx) => {
        const article = await itemsService.createOne({
          ...value,
          author: req.accountability?.user,
        }, { emitEvents: false });

        if (value.tags && value.tags.length > 0) {
          const tagsService = new ItemsService('article_tags', {
            schema: await getSchema(),
            accountability: req.accountability,
            knex: trx,
          });
          await tagsService.createMany(value.tags.map(tag => ({ article_id: article.id, tag_name: tag })));
        }

        emitter.emitAction('custom.article.created', { article, user: req.accountability?.user });
        return article;
      });

      if (env.EMAIL_NOTIFICATIONS === 'true') {
        const mailService = new MailService({ schema: await getSchema() });
        await mailService.send({
          to: env.ADMIN_EMAIL,
          subject: `New Article: ${result.title}`,
          html: `<h2>New Article Published</h2><p><strong>Title:</strong> ${result.title}</p>`,
        });
      }

      return res.status(201).json({ data: result });
    } catch (error) {
      logger.error('Process endpoint error:', error);
      return next(error);
    }
  });

  // Server-Sent Events
  router.get('/stream', (req, res) => {
    res.writeHead(200, { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache', 'Connection': 'keep-alive' });
    res.write('data: {"type":"connected"}\n\n');

    const handler = (data: any) => res.write(`data: ${JSON.stringify(data)}\n\n`);
    emitter.onAction('items.create', handler);
    emitter.onAction('items.update', handler);

    req.on('close', () => {
      emitter.offAction('items.create', handler);
      emitter.offAction('items.update', handler);
    });
  });
});
```

## Process: Implementing Hooks

### Hook Types

1. **Filter Hooks** - Modify data before database operations
2. **Action Hooks** - React to events after they occur
3. **Init Hooks** - Run during startup
4. **Schedule Hooks** - Run on cron schedules

### Step 1: Create Hook Extension

```
// src/index.ts
import { defineHook } from '@directus/extensions-sdk';
import axios from 'axios';

export default defineHook(({ filter, action, init, schedule }, context) => {
  const { services, database, getSchema, env, logger } = context;
  const { ItemsService } = services;

  // Filter hook - auto-generate slugs and metadata
  filter('items.create', async (payload, meta, context) => {
    if (meta.collection === 'articles') {
      if (!payload.slug && payload.title) {
        payload.slug = generateSlug(payload.title);
      }
      payload.word_count = countWords(payload.content || '');
      payload.reading_time = Math.ceil(payload.word_count / 200);
      if (!payload.excerpt && payload.content) {
        payload.excerpt = generateExcerpt(payload.content, 160);
      }

      const schema = await getSchema();
      const articlesService = new ItemsService('articles', { schema, knex: database });
      const existing = await articlesService.readByQuery({ filter: { slug: { _eq: payload.slug } }, limit: 1 });
      if (existing.length > 0) payload.slug = `${payload.slug}-${Date.now()}`;
    }
    return payload;
  });

  // Action hook - sync to external services
  action('items.create', async ({ payload, key, collection }, context) => {
    try {
      if (collection === 'orders') {
        if (env.EXTERNAL_API_URL) {
          await axios.post(`${env.EXTERNAL_API_URL}/webhook/order`, { order_id: key, ...payload }, {
            headers: { 'X-API-Key': env.EXTERNAL_API_KEY },
          });
        }
        await updateStatistics('orders', 'created');
      }
    } catch (error) {
      logger.error('Order creation hook error:', error);
    }
  });

  // Filter hook for updates
  filter('items.update', async (payload, meta, context) => {
    const { collection } = meta;
    if (collection === 'users') {
      payload.last_modified = new Date().toISOString();
      payload.modified_by = context.accountability?.user;
      if (payload.email && !validateEmail(payload.email)) {
        throw new Error('Invalid email format');
      }
    }
    return payload;
  });

  // Init hook
  init('app.before', async () => {
    logger.info('Initializing custom hooks...');
    await verifyCustomTables();
    logger.info('Custom hooks initialized successfully');
  });

  // Schedule hook - daily cleanup
  schedule('0 0 * * *', async () => {
    logger.info('Running daily cleanup...');
    const deleted = await database('directus_sessions').where('expires', '<', new Date()).delete();
    logger.info(`Cleaned ${deleted} expired sessions`);
  });

  // Schedule hook - health check every 5 minutes
  schedule('*/5 * * * *', async () => {
    const health = await checkSystemHealth();
    if (!health.healthy) logger.error('System health check failed:', health.issues);
  });

  function generateSlug(title: string): string {
    return title.toLowerCase().replace(/[^\w\s-]/g, '').replace(/[\s_-]+/g, '-').replace(/^-+|-+$/g, '');
  }

  function countWords(text: string): number { return text.trim().split(/\s+/).length; }

  function generateExcerpt(content: string, maxLength: number): string {
    const stripped = content.replace(/<[^>]*>/g, '');
    return stripped.length <= maxLength ? stripped : stripped.substring(0, maxLength).trim() + '...';
  }

  function validateEmail(email: string): boolean { return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email); }

  async function verifyCustomTables(): Promise<void> {
    const requiredTables = ['custom_logs', 'custom_statistics'];
    for (const table of requiredTables) {
      const exists = await database.schema.hasTable(table);
      if (!exists) {
        await database.schema.createTable(table, (t) => {
          t.uuid('id').primary();
          t.jsonb('data');
          t.timestamps(true, true);
        });
      }
    }
  }

  async function updateStatistics(type: string, action: string): Promise<void> {
    await database('custom_statistics')
      .insert({ id: database.raw('gen_random_uuid()'), type, action, count: 1, date: new Date() })
      .onConflict(['type', 'action', database.raw('DATE(date)')])
      .merge({ count: database.raw('custom_statistics.count + 1') });
  }

  async function checkSystemHealth(): Promise<any> {
    const health = { healthy: true, issues: [] as string[], metrics: {} as any };
    try {
      await database.raw('SELECT 1');
      health.metrics.database = 'connected';
    } catch {
      health.healthy = false;
      health.issues.push('Database connection failed');
    }
    const memUsage = process.memoryUsage();
    const heapUsedMB = Math.round(memUsage.heapUsed / 1024 / 1024);
    health.metrics.memory = `${heapUsedMB}MB`;
    if (heapUsedMB > 512) { health.healthy = false; health.issues.push(`High memory usage: ${heapUsedMB}MB`); }
    return health;
  }
});
```

## Process: Building Flows and Operations

### Step 1: Create Custom Operation

```
// src/index.ts
import { defineOperationApi } from '@directus/extensions-sdk';

type Options = {
  collection: string;
  batchSize: number;
  operation: 'archive' | 'delete' | 'export';
  conditions: Record<string, any>;
};

export default defineOperationApi<Options>({
  id: 'batch-processor',
  handler: async ({ collection, batchSize, operation, conditions }, context) => {
    const { services, database, getSchema, logger } = context;
    const { ItemsService } = services;

    const schema = await getSchema();
    const itemsService = new ItemsService(collection, { schema, accountability: { role: 'admin' } });

    let processed = 0;
    let hasMore = true;
    const results = [];

    while (hasMore) {
      const items = await itemsService.readByQuery({ filter: conditions, limit: batchSize, offset: processed });
      if (items.length === 0) { hasMore = false; break; }

      for (const item of items) {
        try {
          switch (operation) {
            case 'archive':
              await itemsService.updateOne(item.id, { status: 'archived', archived_at: new Date() });
              results.push({ id: item.id, status: 'archived' });
              break;
            case 'delete':
              await itemsService.deleteOne(item.id);
              results.push({ id: item.id, status: 'deleted' });
              break;
            case 'export':
              results.push({ id: item.id, title: item.title || 'Untitled', created: new Date(item.created_at).toISOString() });
              break;
          }
          processed++;
        } catch (error) {
          results.push({ id: item.id, status: 'error', error: error.message });
        }
      }

      if (items.length < batchSize) hasMore = false;
      await new Promise(resolve => setTimeout(resolve, 100));
    }

    return {
      processed,
      results,
      summary: { total: processed, successful: results.filter(r => r.status !== 'error').length, failed: results.filter(r => r.status === 'error').length },
    };
  },
});
```

## Service Layer Architecture

### Custom Service Implementation

```
// src/services/analytics.service.ts
export class AnalyticsService {
  private tableName = 'analytics_events';

  constructor(private options: any) {}

  get knex() { return this.options.knex || this.options.database; }
  get accountability() { return this.options.accountability; }

  async trackEvent(event: {
    category: string;
    action: string;
    label?: string;
    value?: number;
    userId?: string;
    metadata?: Record<string, any>;
  }): Promise<void> {
    await this.knex(this.tableName).insert({
      id: this.knex.raw('gen_random_uuid()'),
      category: event.category,
      action: event.action,
      label: event.label,
      value: event.value,
      user_id: event.userId,
      metadata: JSON.stringify(event.metadata || {}),
      created_at: new Date(),
    });
    await this.updateAggregates(event);
  }

  async getMetrics(options: { startDate: Date; endDate: Date; groupBy: 'hour' | 'day' | 'week' | 'month'; category?: string }): Promise<any[]> {
    const query = this.knex(this.tableName)
      .select(
        this.knex.raw(`DATE_TRUNC('${options.groupBy}', created_at) as period`),
        'category', 'action',
        this.knex.raw('COUNT(*) as count'),
        this.knex.raw('COUNT(DISTINCT user_id) as unique_users'),
        this.knex.raw('AVG(value) as avg_value')
      )
      .whereBetween('created_at', [options.startDate, options.endDate])
      .groupBy('period', 'category', 'action')
      .orderBy('period', 'desc');

    if (options.category) query.where('category', options.category);
    return await query;
  }

  private async updateAggregates(event: any): Promise<void> {
    const date = new Date().toISOString().split('T')[0];
    await this.knex('analytics_aggregates')
      .insert({ id: this.knex.raw('gen_random_uuid()'), date, category: event.category, action: event.action, count: 1, sum_value: event.value || 0 })
      .onConflict(['date', 'category', 'action'])
      .merge({ count: this.knex.raw('analytics_aggregates.count + 1'), sum_value: this.knex.raw('analytics_aggregates.sum_value + ?', [event.value || 0]) });
  }

  async cleanupOldEvents(daysToKeep: number): Promise<number> {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - daysToKeep);
    return await this.knex(this.tableName).where('created_at', '<', cutoffDate).delete();
  }
}
```

## Database Operations

### Advanced Query Patterns

```
// Recursive CTE for hierarchical data
async getHierarchy(parentId: string | null): Promise<any[]> {
  const query = `
    WITH RECURSIVE category_tree AS (
      SELECT id, name, parent_id, 0 as level, ARRAY[id] as path
      FROM categories
      WHERE parent_id ${parentId ? '= ?' : 'IS NULL'}

      UNION ALL

      SELECT c.id, c.name, c.parent_id, ct.level + 1, ct.path || c.id
      FROM categories c
      INNER JOIN category_tree ct ON c.parent_id = ct.id
      WHERE ct.level < 10
    )
    SELECT * FROM category_tree ORDER BY path
  `;
  const result = await this.database.raw(query, parentId ? [parentId] : []);
  return result.rows;
}

// Window functions for analytics
async getRunningTotals(startDate: Date, endDate: Date): Promise<any[]> {
  const query = `
    SELECT
      date,
      amount,
      SUM(amount) OVER (ORDER BY date) as running_total,
      AVG(amount) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as moving_avg_7d
    FROM daily_sales
    WHERE date BETWEEN ? AND ?
    ORDER BY date
  `;
  const result = await this.database.raw(query, [startDate, endDate]);
  return result.rows;
}

// Optimized pagination with cursor
async getCursorPagination(options: { table: string; cursor?: string; limit: number; orderBy: string }): Promise<{ data: any[]; nextCursor: string | null }> {
  let query = this.database(options.table).orderBy(options.orderBy).limit(options.limit + 1);
  if (options.cursor) {
    const decodedCursor = Buffer.from(options.cursor, 'base64').toString();
    query = query.where(options.orderBy, '>', decodedCursor);
  }
  const results = await query;
  const hasMore = results.length > options.limit;
  const data = hasMore ? results.slice(0, -1) : results;
  const nextCursor = hasMore ? Buffer.from(data[data.length - 1][options.orderBy]).toString('base64') : null;
  return { data, nextCursor };
}
```

## Authentication & Permissions

### Custom Authentication Provider

```
// src/auth-provider.ts
import jwt from 'jsonwebtoken';
import bcrypt from 'bcrypt';

export class CustomAuthProvider {
  constructor(private knex: any, private secret: string) {}

  async login(credentials: { email: string; password: string }) {
    const user = await this.knex('directus_users').where('email', credentials.email).first();

    if (!user) throw new Error('Invalid credentials');

    const validPassword = await bcrypt.compare(credentials.password, user.password);
    if (!validPassword) throw new Error('Invalid credentials');

    if (user.status !== 'active') throw new Error('Account is not active');

    const accessToken = this.generateAccessToken(user);
    const refreshToken = await this.generateRefreshToken(user);

    await this.knex('directus_users').where('id', user.id).update({ last_access: new Date() });

    const { password, ...sanitizedUser } = user;
    return { accessToken, refreshToken, user: sanitizedUser };
  }

  async verify(token: string) {
    const payload = jwt.verify(token, this.secret) as any;
    const session = await this.knex('directus_sessions').where('token', token).where('expires', '>', new Date()).first();
    if (!session) throw new Error('Session expired');
    return { id: payload.id, role: payload.role, app_access: payload.app_access };
  }

  private generateAccessToken(user: any): string {
    return jwt.sign({ id: user.id, role: user.role, app_access: user.app_access, email: user.email }, this.secret, { expiresIn: '15m', issuer: 'directus' });
  }

  private async generateRefreshToken(user: any): Promise<string> {
    const token = require('crypto').randomBytes(32).toString('hex');
    await this.knex('directus_sessions').insert({
      token, user: user.id,
      expires: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    });
    return token;
  }
}
```

## Performance Optimization

### Caching Strategy

```
// src/cache/cache.service.ts
import Redis from 'ioredis';
import { LRUCache } from 'lru-cache';

export class CacheService {
  private redis: Redis;
  private memoryCache: LRUCache<string, any>;

  constructor() {
    this.redis = new Redis({ host: process.env.REDIS_HOST || 'localhost', port: parseInt(process.env.REDIS_PORT || '6379') });
    this.memoryCache = new LRUCache({ max: 500, ttl: 1000 * 60 * 5 });
  }

  async get(key: string): Promise<any | null> {
    const memResult = this.memoryCache.get(key);
    if (memResult) return memResult;
    const redisResult = await this.redis.get(key);
    if (redisResult) {
      const data = JSON.parse(redisResult);
      this.memoryCache.set(key, data);
      return data;
    }
    return null;
  }

  async set(key: string, value: any, ttl?: number): Promise<void> {
    const serialized = JSON.stringify(value);
    this.memoryCache.set(key, value);
    if (ttl) await this.redis.setex(key, ttl, serialized);
    else await this.redis.set(key, serialized);
  }

  async invalidate(pattern: string): Promise<void> {
    for (const key of this.memoryCache.keys()) {
      if (key.includes(pattern)) this.memoryCache.delete(key);
    }
    const keys = await this.redis.keys(`*${pattern}*`);
    if (keys.length > 0) await this.redis.del(...keys);
  }

  async remember<T>(key: string, ttl: number, callback: () => Promise<T>): Promise<T> {
    const cached = await this.get(key);
    if (cached) return cached;
    const fresh = await callback();
    await this.set(key, fresh, ttl);
    return fresh;
  }
}
```

## Database Migrations

```
// migrations/001_create_analytics_tables.ts
import { Knex } from 'knex';

export async function up(knex: Knex): Promise<void> {
  await knex.schema.createTable('analytics_events', (table) => {
    table.uuid('id').primary().defaultTo(knex.raw('gen_random_uuid()'));
    table.string('category', 50).notNullable();
    table.string('action', 50).notNullable();
    table.string('label', 100);
    table.decimal('value', 10, 2);
    table.uuid('user_id').references('id').inTable('directus_users');
    table.jsonb('metadata');
    table.string('session_id', 100);
    table.string('ip_address', 45);
    table.timestamp('created_at').defaultTo(knex.fn.now());
    table.index(['category', 'action']);
    table.index('user_id');
    table.index('created_at');
  });

  await knex.schema.createTable('analytics_aggregates', (table) => {
    table.uuid('id').primary().defaultTo(knex.raw('gen_random_uuid()'));
    table.date('date').notNullable();
    table.string('category', 50).notNullable();
    table.string('action', 50).notNullable();
    table.integer('count').defaultTo(0);
    table.decimal('sum_value', 12, 2).defaultTo(0);
    table.timestamps(true, true);
    table.unique(['date', 'category', 'action']);
    table.index('date');
  });
}

export async function down(knex: Knex): Promise<void> {
  await knex.schema.dropTableIfExists('analytics_aggregates');
  await knex.schema.dropTableIfExists('analytics_events');
}
```

## Success Metrics

* ✅ API endpoints respond within 200ms for 95% of requests
* ✅ Database queries optimized with proper indexing
* ✅ Hooks execute without blocking main operations
* ✅ Flows process batches efficiently without memory leaks
* ✅ Authentication system secure with proper token management
* ✅ Caching reduces database load by 60%+
* ✅ Error handling prevents data corruption
* ✅ Logging provides complete audit trail
* ✅ Tests achieve 80%+ code coverage
* ✅ Migrations run without data loss

## Resources

* [Directus API Documentation](https://docs.directus.io/reference/introduction.html)
* [Directus Extensions SDK](https://docs.directus.io/extensions/introduction.html)
* [Knex.js Query Builder](https://knexjs.org/)
* [Express.js Middleware](https://expressjs.com/en/guide/using-middleware.html)
* [Node.js Best Practices](https://github.com/goldbergyoni/nodebestpractices)
* [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/)

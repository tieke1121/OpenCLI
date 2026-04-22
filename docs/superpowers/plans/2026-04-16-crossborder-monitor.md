# Cross-Border Product Monitor Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan.

**Goal:** Build a self-hosted product monitoring tool for cross-border e-commerce sellers that tracks 1688 prices, Amazon competitor prices, and Xiaohongshu social media trends.

**Architecture:** Monolithic Node.js application with:
- **OpenCLI** as the scraping engine (browser automation via Chrome Extension)
- **MySQL 8** for data persistence (via Docker, one-click deployment)
- **Express.js** REST API for querying data and triggering scans
- **node-cron** for scheduled monitoring tasks
- **Telegram Bot** for price alert notifications

**Tech Stack:** Node.js >= 21, MySQL 8 (Docker), Express, node-cron, Telegram Bot API

**Docker Support:** Full Docker/Docker Compose setup for one-click VPS deployment

---

## File Structure

```
crossborder-monitor/
├── src/
│   ├── main.ts                    # Entry point, CLI commands
│   ├── db.ts                      # MySQL connection & migrations
│   ├── api.ts                     # Express REST API server
│   ├── scheduler.ts               # Cron job definitions
│   ├── commands/
│   │   ├── scan-1688.ts           # 1688 product price scan
│   │   ├── scan-amazon.ts         # Amazon competitor price scan
│   │   └── scan-xiaohongshu.ts     # Xiaohongshu trending products
│   ├── services/
│   │   ├── opencli.ts             # OpenCLI wrapper (spawn opencli commands)
│   │   ├── notifier.ts            # Telegram notification service
│   │   └── price-detector.ts      # Price change detection logic
│   ├── models/
│   │   ├── product.ts             # Product model
│   │   ├── price-history.ts        # Price history model
│   │   └── alert.ts               # Alert model
│   └── types/
│       └── index.ts                # TypeScript interfaces
├── scripts/
│   └── init-db.ts                  # Database initialization
├── docker/
│   ├── Dockerfile                  # Application container
│   └── nginx.conf                  # Reverse proxy (optional)
├── docker-compose.yml              # One-click deployment
├── .env.example                    # Environment template
├── package.json
├── tsconfig.json
└── README.md
```

---

## Chunk 1: Project Setup

### Task 1: Initialize Project

**Files:**
- Create: `crossborder-monitor/package.json`
- Create: `crossborder-monitor/tsconfig.json`

- [ ] **Step 1: Create project directory**

```bash
mkdir -p crossborder-monitor
cd crossborder-monitor
npm init -y
```

- [ ] **Step 2: Create package.json**

```json
{
  "name": "crossborder-monitor",
  "version": "1.0.0",
  "type": "module",
  "main": "dist/main.js",
  "bin": {
    "monitor": "dist/main.js"
  },
  "scripts": {
    "dev": "tsx src/main.ts",
    "dev:watch": "tsx watch src/main.ts",
    "build": "tsc",
    "start": "node dist/main.js",
    "init-db": "tsx scripts/init-db.ts"
  },
  "dependencies": {
    "express": "^4.18.2",
    "mysql2": "^3.9.1",
    "node-cron": "^3.0.3",
    "yaml": "^2.3.4",
    "undici": "^6.6.2",
    "dotenv": "^16.4.1"
  },
  "devDependencies": {
    "@types/express": "^4.17.21",
    "@types/node": "^20.11.0",
    "@types/node-cron": "^3.0.11",
    "tsx": "^4.19.3",
    "typescript": "^5.3.3"
  }
}
```

- [ ] **Step 3: Create tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

- [ ] **Step 4: Install dependencies**

```bash
npm install
```

- [ ] **Step 5: Commit**

```bash
git init
git add package.json tsconfig.json
git commit -m "feat: initialize crossborder-monitor project"
```

---

### Task 2: Database Setup

**Files:**
- Create: `crossborder-monitor/src/db.ts`
- Create: `crossborder-monitor/scripts/init-db.ts`
- Create: `crossborder-monitor/src/types/index.ts`

- [ ] **Step 1: Create types**

```typescript
// src/types/index.ts
export interface Product {
  id: string;
  platform: '1688' | 'amazon' | 'xiaohongshu';
  external_id: string;
  title: string;
  url: string;
  current_price: number | null;
  currency: string;
  created_at: string;
  updated_at: string;
}

export interface PriceHistory {
  id: string;
  product_id: string;
  price: number;
  currency: string;
  recorded_at: string;
}

export interface Alert {
  id: string;
  product_id: string;
  type: 'price_drop' | 'price_rise' | 'new_product';
  threshold?: number;
  enabled: boolean;
  telegram_chat_id?: string;
  created_at: string;
}

export interface ScanResult {
  product_id: string;
  success: boolean;
  error?: string;
  items_found: number;
}
```

- [ ] **Step 2: Create database module**

```typescript
// src/db.ts
import mysql from 'mysql2/promise';
import { log } from './logger.js';

interface DbConfig {
  host: string;
  port: number;
  user: string;
  password: string;
  database: string;
}

let pool: mysql.Pool | null = null;

export function getDbConfig(): DbConfig {
  return {
    host: process.env.DB_HOST ?? 'localhost',
    port: parseInt(process.env.DB_PORT ?? '3306'),
    user: process.env.DB_USER ?? 'monitor',
    password: process.env.DB_PASSWORD ?? 'monitor_password',
    database: process.env.DB_NAME ?? 'crossborder_monitor',
  };
}

export async function getPool(): Promise<mysql.Pool> {
  if (!pool) {
    const config = getDbConfig();
    pool = mysql.createPool({
      host: config.host,
      port: config.port,
      user: config.user,
      password: config.password,
      database: config.database,
      waitForConnections: true,
      connectionLimit: 10,
      queueLimit: 0,
      enableKeepAlive: true,
      keepAliveInitialDelay: 0,
    });
    log.info(`[db] MySQL pool: ${config.host}:${config.port}/${config.database}`);
  }
  return pool;
}

export async function initDb(): Promise<void> {
  const config = getDbConfig();

  // First connect without database to create it if needed
  const tempPool = mysql.createPool({
    host: config.host,
    port: config.port,
    user: config.user,
    password: config.password,
    waitForConnections: true,
    connectionLimit: 2,
  });

  try {
    await tempPool.execute(`CREATE DATABASE IF NOT EXISTS \`${config.database}\``);
    await tempPool.end();
  } catch (error) {
    await tempPool.end();
    throw error;
  }

  // Now use the actual database
  const db = await getPool();

  await db.execute(`
    CREATE TABLE IF NOT EXISTS products (
      id VARCHAR(36) PRIMARY KEY,
      platform ENUM('1688', 'amazon', 'xiaohongshu') NOT NULL,
      external_id VARCHAR(255) NOT NULL,
      title VARCHAR(1000) NOT NULL,
      url TEXT NOT NULL,
      current_price DECIMAL(12, 2),
      currency VARCHAR(10) DEFAULT 'CNY',
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
      UNIQUE KEY uk_platform_external (platform, external_id),
      INDEX idx_platform (platform),
      INDEX idx_updated (updated_at)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  `);

  await db.execute(`
    CREATE TABLE IF NOT EXISTS price_history (
      id VARCHAR(36) PRIMARY KEY,
      product_id VARCHAR(36) NOT NULL,
      price DECIMAL(12, 2) NOT NULL,
      currency VARCHAR(10) NOT NULL,
      recorded_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
      INDEX idx_product_recorded (product_id, recorded_at)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  `);

  await db.execute(`
    CREATE TABLE IF NOT EXISTS alerts (
      id VARCHAR(36) PRIMARY KEY,
      product_id VARCHAR(36) NOT NULL,
      type ENUM('price_drop', 'price_rise', 'new_product') NOT NULL,
      threshold DECIMAL(5, 2),
      enabled BOOLEAN DEFAULT TRUE,
      telegram_chat_id VARCHAR(100),
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
      INDEX idx_product (product_id)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  `);

  await db.execute(`
    CREATE TABLE IF NOT EXISTS scan_logs (
      id VARCHAR(36) PRIMARY KEY,
      platform ENUM('1688', 'amazon', 'xiaohongshu') NOT NULL,
      started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      completed_at TIMESTAMP NULL,
      items_found INT DEFAULT 0,
      errors TEXT,
      INDEX idx_started (started_at)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  `);

  log.info('[db] All tables created/verified');
}

export async function closeDb(): Promise<void> {
  if (pool) {
    await pool.end();
    pool = null;
    log.info('[db] MySQL pool closed');
  }
}
```

- [ ] **Step 3: Create init script**

```typescript
// scripts/init-db.ts
import 'dotenv/config';
import { initDb } from '../src/db.js';

console.log('Initializing MySQL database...');
await initDb();
console.log('Database initialization complete!');
```

- [ ] **Step 4: Create data directory and init**

```bash
mkdir -p data
npm run init-db
```

- [ ] **Step 5: Commit**

```bash
git add src/db.ts src/types/index.ts scripts/init-db.ts
git commit -m "feat: add MySQL database layer with product/price history/alerts tables"
```

---

### Task 3: OpenCLI Wrapper Service

**Files:**
- Create: `crossborder-monitor/src/services/opencli.ts`

- [ ] **Step 1: Create OpenCLI wrapper**

```typescript
// src/services/opencli.ts
import { spawn } from 'node:child_process';
import { promisify } from 'node:util';
import { log } from '../logger.js';

const execFile = promisify(spawn);

interface OpenCLIResult {
  stdout: string;
  stderr: string;
  exitCode: number;
}

export async function runOpenCLI(args: string[]): Promise<OpenCLIResult> {
  log.info(`[opencli] Running: opencli ${args.join(' ')}`);

  return new Promise((resolve) => {
    const proc = spawn('opencli', args, {
      stdio: ['pipe', 'pipe', 'pipe'],
      shell: true,
    });

    let stdout = '';
    let stderr = '';

    proc.stdout?.on('data', (data) => {
      stdout += data.toString();
    });

    proc.stderr?.on('data', (data) => {
      stderr += data.toString();
    });

    proc.on('close', (code) => {
      resolve({
        stdout,
        stderr,
        exitCode: code ?? 0,
      });
    });

    proc.on('error', (err) => {
      stderr += err.message;
      resolve({ stdout, stderr, exitCode: 1 });
    });
  });
}

export interface ProductItem {
  rank?: number;
  title: string;
  price_text?: string;
  price_min?: number;
  price_max?: number;
  currency?: string;
  moq_text?: string;
  seller_name?: string;
  location?: string;
  item_url: string;
  source_url?: string;
}

export async function scan1688Products(query: string, limit: number = 20): Promise<ProductItem[]> {
  const result = await runOpenCLI([
    '1688', 'search', query,
    '--limit', String(limit),
    '--format', 'json'
  ]);

  if (result.exitCode !== 0) {
    throw new Error(`1688 scan failed: ${result.stderr || result.stdout}`);
  }

  try {
    const data = JSON.parse(result.stdout);
    return data as ProductItem[];
  } catch {
    throw new Error(`Failed to parse 1688 response: ${result.stdout}`);
  }
}

export async function scanAmazonProducts(query: string, limit: number = 20): Promise<ProductItem[]> {
  const result = await runOpenCLI([
    'amazon', 'search', query,
    '--limit', String(limit),
    '--format', 'json'
  ]);

  if (result.exitCode !== 0) {
    throw new Error(`Amazon scan failed: ${result.stderr || result.stdout}`);
  }

  try {
    const data = JSON.parse(result.stdout);
    return data as ProductItem[];
  } catch {
    throw new Error(`Failed to parse Amazon response: ${result.stdout}`);
  }
}

export async function scanXiaohongshu(keyword: string, limit: number = 20): Promise<ProductItem[]> {
  const result = await runOpenCLI([
    'xiaohongshu', 'search', keyword,
    '--limit', String(limit),
    '--format', 'json'
  ]);

  if (result.exitCode !== 0) {
    throw new Error(`Xiaohongshu scan failed: ${result.stderr || result.stdout}`);
  }

  try {
    const data = JSON.parse(result.stdout);
    return data as ProductItem[];
  } catch {
    throw new Error(`Failed to parse Xiaohongshu response: ${result.stdout}`);
  }
}
```

- [ ] **Step 2: Create logger utility**

```typescript
// src/logger.ts
type LogLevel = 'info' | 'warn' | 'error' | 'debug';

const DEBUG = process.env.DEBUG === '1';

function formatTime(): string {
  return new Date().toISOString();
}

function log(level: LogLevel, ...args: unknown[]): void {
  const prefix = `[${formatTime()}] [${level.toUpperCase()}]`;
  if (level === 'debug' && !DEBUG) return;

  const fn = level === 'error' ? console.error
           : level === 'warn'  ? console.warn
           : console.log;

  fn(prefix, ...args);
}

export const logger = {
  info: (...args: unknown[]) => log('info', ...args),
  warn: (...args: unknown[]) => log('warn', ...args),
  error: (...args: unknown[]) => log('error', ...args),
  debug: (...args: unknown[]) => log('debug', ...args),
};

export const log = {
  info: (msg: string, ...meta: unknown[]) => logger.info(msg, ...meta),
  warn: (msg: string, ...meta: unknown[]) => logger.warn(msg, ...meta),
  error: (msg: string, ...meta: unknown[]) => logger.error(msg, ...meta),
  debug: (msg: string, ...meta: unknown[]) => logger.debug(msg, ...meta),
};
```

- [ ] **Step 3: Commit**

```bash
git add src/services/opencli.ts src/logger.ts
git commit -m "feat: add OpenCLI wrapper service for 1688/Amazon/Xiaohongshu scanning"
```

---

## Chunk 2: Core Features

### Task 4: Scan Commands

**Files:**
- Create: `crossborder-monitor/src/commands/scan-1688.ts`
- Create: `crossborder-monitor/src/commands/scan-amazon.ts`
- Create: `crossborder-monitor/src/commands/scan-xiaohongshu.ts`

- [ ] **Step 1: Create 1688 scan command**

```typescript
// src/commands/scan-1688.ts
import { getDb } from '../db.js';
import { scan1688Products, type ProductItem } from '../services/opencli.js';
import { log } from '../logger.js';
import { detectPriceChanges } from '../services/price-detector.js';
import { sendAlert } from '../services/notifier.js';
import { randomId } from '../utils.js';

export interface ScanOptions {
  query: string;
  limit?: number;
  dryRun?: boolean;
}

export async function scan1688(options: ScanOptions): Promise<{ success: boolean; itemsFound: number }> {
  const db = getDb();
  const limit = options.limit ?? 20;

  log.info(`[1688] Starting scan for query: "${options.query}"`);

  const scanLogId = randomId();
  const startedAt = new Date().toISOString();

  try {
    const products = await scan1688Products(options.query, limit);
    log.info(`[1688] Found ${products.length} products`);

    let insertedCount = 0;
    let priceChangeCount = 0;

    const insertProduct = db.prepare(`
      INSERT INTO products (id, platform, external_id, title, url, current_price, currency, updated_at)
      VALUES (?, ?, ?, ?, ?, ?, ?, ?)
      ON CONFLICT(platform, external_id) DO UPDATE SET
        title = excluded.title,
        url = excluded.url,
        current_price = excluded.current_price,
        updated_at = excluded.updated_at
    `);

    const insertHistory = db.prepare(`
      INSERT INTO price_history (id, product_id, price, currency)
      VALUES (?, ?, ?, ?)
    `);

    const getPreviousPrice = db.prepare(`
      SELECT price FROM price_history
      WHERE product_id = ?
      ORDER BY recorded_at DESC
      LIMIT 1
    `);

    for (const item of products) {
      const externalId = extractExternalId(item.item_url);
      if (!externalId) continue;

      const price = item.price_min ?? 0;
      const currency = item.currency ?? 'CNY';

      // Insert or update product
      insertProduct.run(
        randomId(),
        '1688',
        externalId,
        item.title,
        item.item_url,
        price,
        currency,
        new Date().toISOString()
      );

      // Get product id
      const product = db.prepare(`
        SELECT id FROM products WHERE platform = '1688' AND external_id = ?
      `).get(externalId) as { id: string } | undefined;

      if (!product) continue;

      // Record price history
      insertHistory.run(randomId(), product.id, price, currency);

      // Check for price changes
      const previous = getPreviousPrice.get(product.id) as { price: number } | undefined;
      if (previous && previous.price !== price) {
        priceChangeCount++;
        const change = ((price - previous.price) / previous.price) * 100;

        log.info(`[1688] Price change detected: ${item.title} - ${previous.price} → ${price} (${change.toFixed(1)}%)`);

        if (!options.dryRun) {
          await sendAlert({
            productId: product.id,
            type: price < previous.price ? 'price_drop' : 'price_rise',
            oldPrice: previous.price,
            newPrice: price,
            currency,
            title: item.title,
          });
        }
      }

      insertedCount++;
    }

    // Update scan log
    db.prepare(`
      INSERT INTO scan_logs (id, platform, completed_at, items_found)
      VALUES (?, '1688', datetime('now'), ?)
    `).run(scanLogId, insertedCount);

    log.info(`[1688] Scan complete: ${insertedCount} products, ${priceChangeCount} price changes`);

    return { success: true, itemsFound: insertedCount };

  } catch (error) {
    const errorMessage = error instanceof Error ? error.message : String(error);
    log.error(`[1688] Scan failed: ${errorMessage}`);

    db.prepare(`
      INSERT INTO scan_logs (id, platform, completed_at, errors)
      VALUES (?, '1688', datetime('now'), ?)
    `).run(scanLogId, errorMessage);

    return { success: false, itemsFound: 0 };
  }
}

function extractExternalId(url: string): string | null {
  // Extract offer ID from 1688 URL
  const match = url.match(/offer[s]?\/(\d+)/i);
  return match ? match[1] : null;
}
```

- [ ] **Step 2: Create Amazon scan command (similar structure)**

```typescript
// src/commands/scan-amazon.ts
// Same pattern as scan-1688.ts but for Amazon
// Extract ASIN from Amazon URL
// Platform: 'amazon'
```

- [ ] **Step 3: Create Xiaohongshu scan command**

```typescript
// src/commands/scan-xiaohongshu.ts
// Same pattern but for Xiaohongshu
// Platform: 'xiaohongshu'
// Note: Xiaohongshu doesn't have price, so alert type is 'new_product'
```

- [ ] **Step 4: Commit**

```bash
git add src/commands/scan-*.ts
git commit -m "feat: add scan commands for 1688, Amazon, Xiaohongshu"
```

---

### Task 5: Price Change Detection & Notifications

**Files:**
- Create: `crossborder-monitor/src/services/price-detector.ts`
- Create: `crossborder-monitor/src/services/notifier.ts`

- [ ] **Step 1: Create price detector**

```typescript
// src/services/price-detector.ts
export interface PriceChange {
  productId: string;
  type: 'price_drop' | 'price_rise';
  oldPrice: number;
  newPrice: number;
  changePercent: number;
}

export function detectPriceChanges(
  oldPrice: number,
  newPrice: number
): PriceChange | null {
  if (oldPrice === newPrice) return null;

  const changePercent = ((newPrice - oldPrice) / oldPrice) * 100;

  return {
    productId: '', // Will be set by caller
    type: newPrice < oldPrice ? 'price_drop' : 'price_rise',
    oldPrice,
    newPrice,
    changePercent,
  };
}

export function shouldNotify(change: PriceChange, threshold: number = 5): boolean {
  // Only notify if change exceeds threshold (default 5%)
  return Math.abs(change.changePercent) >= threshold;
}
```

- [ ] **Step 2: Create notifier service**

```typescript
// src/services/notifier.ts
import { log } from '../logger.js';

const TELEGRAM_BOT_TOKEN = process.env.TELEGRAM_BOT_TOKEN;
const TELEGRAM_CHAT_ID = process.env.TELEGRAM_CHAT_ID;

interface AlertPayload {
  productId: string;
  type: 'price_drop' | 'price_rise' | 'new_product';
  oldPrice?: number;
  newPrice: number;
  currency: string;
  title: string;
  url?: string;
}

export async function sendAlert(payload: AlertPayload): Promise<boolean> {
  if (!TELEGRAM_BOT_TOKEN || !TELEGRAM_CHAT_ID) {
    log.warn('[notifier] Telegram not configured, skipping notification');
    return false;
  }

  const emoji = payload.type === 'price_drop' ? '🔻' : payload.type === 'price_rise' ? '🔺' : '🆕';
  const priceText = payload.oldPrice
    ? `${payload.oldPrice} → ${payload.newPrice} ${payload.currency}`
    : `${payload.newPrice} ${payload.currency}`;

  const message = `
${emoji} *Price Alert*

*${payload.title}*

${priceText}
${payload.url ? `\n🔗 ${payload.url}` : ''}
  `.trim();

  try {
    const response = await fetch(`https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        chat_id: TELEGRAM_CHAT_ID,
        text: message,
        parse_mode: 'Markdown',
      }),
    });

    if (!response.ok) {
      throw new Error(`Telegram API error: ${response.status}`);
    }

    log.info('[notifier] Alert sent successfully');
    return true;
  } catch (error) {
    log.error('[notifier] Failed to send alert:', error);
    return false;
  }
}

export function isNotifierConfigured(): boolean {
  return Boolean(TELEGRAM_BOT_TOKEN && TELEGRAM_CHAT_ID);
}
```

- [ ] **Step 3: Create utils**

```typescript
// src/utils.ts
export function randomId(): string {
  return Math.random().toString(36).substring(2, 15) +
         Math.random().toString(36).substring(2, 15);
}
```

- [ ] **Step 4: Commit**

```bash
git add src/services/price-detector.ts src/services/notifier.ts src/utils.ts
git commit -m "feat: add price change detection and Telegram notification"
```

---

### Task 6: Scheduler & API Server

**Files:**
- Create: `crossborder-monitor/src/scheduler.ts`
- Create: `crossborder-monitor/src/api.ts`

- [ ] **Step 1: Create scheduler**

```typescript
// src/scheduler.ts
import cron from 'node-cron';
import { scan1688 } from './commands/scan-1688.js';
import { log } from './logger.js';
import { getDb } from './db.js';

interface ScheduledTask {
  name: string;
  cronExpression: string;
  platform: '1688' | 'amazon' | 'xiaohongshu';
  query: string;
  enabled: boolean;
}

const DEFAULT_TASKS: ScheduledTask[] = [
  {
    name: '1688-price-watch',
    cronExpression: '0 */6 * * *', // Every 6 hours
    platform: '1688',
    query: '收纳盒',
    enabled: true,
  },
];

export function initScheduler(customTasks?: ScheduledTask[]): void {
  const tasks = customTasks ?? DEFAULT_TASKS;

  for (const task of tasks) {
    if (!task.enabled) continue;

    log.info(`[scheduler] Registered task: ${task.name} (${task.cronExpression})`);

    cron.schedule(task.cronExpression, async () => {
      log.info(`[scheduler] Running task: ${task.name}`);

      try {
        switch (task.platform) {
          case '1688':
            await scan1688({ query: task.query });
            break;
          // Add other platforms as needed
        }
      } catch (error) {
        log.error(`[scheduler] Task ${task.name} failed:`, error);
      }
    });
  }
}

export function getScheduledTasks(): ScheduledTask[] {
  const db = getDb();
  // Could load from DB in production
  return DEFAULT_TASKS;
}
```

- [ ] **Step 2: Create API server**

```typescript
// src/api.ts
import express, { type Request, type Response } from 'express';
import { getDb } from './db.js';
import { scan1688 } from './commands/scan-1688.js';
import { initScheduler } from './scheduler.js';
import { log } from './logger.js';

const app = express();
app.use(express.json());

const PORT = process.env.PORT ?? 3000;

// Health check
app.get('/health', (_req: Request, res: Response) => {
  res.json({ status: 'ok', timestamp: new Date().toISOString() });
});

// Get all products
app.get('/api/products', (req: Request, res: Response) => {
  const db = getDb();
  const platform = req.query.platform as string | undefined;

  let query = 'SELECT * FROM products';
  const params: string[] = [];

  if (platform) {
    query += ' WHERE platform = ?';
    params.push(platform);
  }

  query += ' ORDER BY updated_at DESC LIMIT 100';

  const products = db.prepare(query).all(...params);
  res.json({ products });
});

// Get price history for a product
app.get('/api/products/:id/history', (req: Request, res: Response) => {
  const db = getDb();
  const history = db.prepare(`
    SELECT * FROM price_history
    WHERE product_id = ?
    ORDER BY recorded_at DESC
    LIMIT 30
  `).all(req.params.id);

  res.json({ history });
});

// Trigger a scan manually
app.post('/api/scan/:platform', async (req: Request, res: Response) => {
  const { platform } = req.params;
  const { query, limit } = req.body;

  if (!query) {
    res.status(400).json({ error: 'query is required' });
    return;
  }

  log.info(`[api] Manual scan triggered: ${platform} - ${query}`);

  try {
    let result;
    switch (platform) {
      case '1688':
        result = await scan1688({ query, limit });
        break;
      default:
        res.status(400).json({ error: `Platform ${platform} not supported` });
        return;
    }

    res.json(result);
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);
    res.status(500).json({ error: message });
  }
});

// Get scan history
app.get('/api/scans', (req: Request, res: Response) => {
  const db = getDb();
  const scans = db.prepare(`
    SELECT * FROM scan_logs
    ORDER BY started_at DESC
    LIMIT 50
  `).all();

  res.json({ scans });
});

export function startApi(): Promise<void> {
  return new Promise((resolve) => {
    initScheduler();
    app.listen(PORT, () => {
      log.info(`[api] Server started on port ${PORT}`);
      resolve();
    });
  });
}
```

- [ ] **Step 3: Create main entry point**

```typescript
// src/main.ts
import { Command } from 'commander';
import { initDb, closeDb } from './db.js';
import { startApi } from './api.js';
import { scan1688 } from './commands/scan-1688.js';
import { log } from './logger.js';

const program = new Command();

program
  .name('monitor')
  .description('Cross-border product monitor')
  .version('1.0.0');

program
  .command('scan')
  .description('Scan products from a platform')
  .argument('<platform>', 'Platform: 1688, amazon, xiaohongshu')
  .argument('<query>', 'Search query')
  .option('-l, --limit <number>', 'Number of results', '20')
  .option('--dry-run', 'Run without sending notifications')
  .action(async (platform, query, opts) => {
    initDb();

    try {
      let result;
      switch (platform) {
        case '1688':
          result = await scan1688({ query, limit: parseInt(opts.limit), dryRun: opts.dryRun });
          break;
        default:
          console.error(`Platform ${platform} not supported yet`);
          process.exit(1);
      }

      console.log(JSON.stringify(result, null, 2));
    } finally {
      closeDb();
    }
  });

program
  .command('api')
  .description('Start the REST API server')
  .option('-p, --port <number>', 'Port', '3000')
  .action(async (opts) => {
    process.env.PORT = opts.port;
    initDb();

    // Handle graceful shutdown
    process.on('SIGINT', () => {
      log.info('Shutting down...');
      closeDb();
      process.exit(0);
    });

    await startApi();
  });

program
  .command('init')
  .description('Initialize the database')
  .action(() => {
    initDb();
    console.log('Database initialized!');
  });

program.parse();
```

- [ ] **Step 4: Commit**

```bash
git add src/scheduler.ts src/api.ts src/main.ts
git commit -m "feat: add scheduler and REST API server"
```

---

## Chunk 3: Testing & Deployment

### Task 7: Local Testing

**Files:**
- Create: `crossborder-monitor/test/scan.test.ts`

- [ ] **Step 1: Verify OpenCLI is installed and working**

```bash
opencli --version
opencli list | head -20
```

- [ ] **Step 2: Test 1688 scan locally**

```bash
npm run dev -- scan 1688 "收纳盒" --limit 5
```

Expected: JSON output with scanned products

- [ ] **Step 3: Test API server**

```bash
npm run dev -- api &
curl http://localhost:3000/health
curl http://localhost:3000/api/products
```

- [ ] **Step 4: Test with Telegram (optional)**

```bash
TELEGRAM_BOT_TOKEN=your_token TELEGRAM_CHAT_ID=your_chat_id npm run dev -- scan 1688 "收纳盒"
```

- [ ] **Step 5: Commit test results**

```bash
git add README.md
git commit -m "docs: add local testing guide"
```

---

### Task 8: VPS Deployment Guide

**Files:**
- Create: `crossborder-monitor/README.md`

- [ ] **Step 1: Create deployment README**

```markdown
# Cross-Border Product Monitor

Self-hosted product monitoring for cross-border e-commerce sellers.

## Features

- Monitor 1688 product prices
- Track Amazon competitor prices
- Xiaohongshu trending product discovery
- Price change alerts via Telegram
- REST API for integrations
- Scheduled scanning with cron

## Quick Start (Docker - Recommended)

### One-Click Deployment

\`\`\`bash
# Clone the repo
git clone <your-repo>
cd crossborder-monitor

# Configure environment
cp .env.example .env
# Edit .env with your settings

# Start everything (MySQL + App + Chrome)
docker-compose up -d

# Initialize database
docker-compose exec app npm run init-db

# Done! API available at http://localhost:3000
\`\`\`

## Manual Setup (Without Docker)

### Requirements

- Node.js >= 21.0.0
- MySQL 8.0+ (or use Docker for MySQL only)
- Chrome/Chromium browser
- OpenCLI (installed globally)
- Chrome Extension (Browser Bridge)

### 1. Install OpenCLI

\`\`\`bash
npm install -g @jackwener/opencli
\`\`\`

### 2. Install Chrome Extension

1. Download `opencli-extension-v{version}.zip` from [Releases](https://github.com/jackwener/opencli/releases)
2. Unzip and load in `chrome://extensions` (Developer mode)
3. Log into your 1688/Amazon accounts in Chrome

### 3. Install Monitor

\`\`\`bash
git clone <your-repo>
cd crossborder-monitor
npm install
npm run build
\`\`\`

### 4. Configure Environment

\`\`\`bash
cp .env.example .env
# Edit .env with your MySQL and Telegram settings
\`\`\`

### 5. Initialize Database

\`\`\`bash
npm run init-db
\`\`\`

### 6. Run

\`\`\`bash
# Start API server
npm start

# Or run a one-time scan
npm run dev -- scan 1688 "收纳盒"
\`\`\`

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /health | Health check |
| GET | /api/products | List all monitored products |
| GET | /api/products/:id/history | Price history for a product |
| POST | /api/scan/:platform | Trigger a manual scan |
| GET | /api/scans | Scan history |

## Docker Deployment (Production)

### Prerequisites

- Docker Engine 24.0+
- Docker Compose v2.20+

### Production Setup

\`\`\`bash
# SSH into your VPS
ssh user@your-vps

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Clone and setup
git clone <your-repo>
cd crossborder-monitor

# Edit production environment
cp .env.example .env
nano .env  # Set secure passwords

# Start in production mode
docker-compose -f docker-compose.yml up -d --build

# View logs
docker-compose logs -f

# Check status
docker-compose ps
\`\`\`

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| DB_HOST | mysql | MySQL container hostname |
| DB_PORT | 3306 | MySQL port |
| DB_USER | monitor | MySQL username |
| DB_PASSWORD | (required) | MySQL password |
| DB_NAME | crossborder_monitor | Database name |
| PORT | 3000 | Application port |
| TELEGRAM_BOT_TOKEN | (optional) | Telegram bot token |
| TELEGRAM_CHAT_ID | (optional) | Telegram chat ID |

## Architecture

\`\`\`
┌─────────────────────────────────────────────────────────────┐
│                     crossborder-monitor                     │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │   CLI API   │  │ REST API    │  │  Scheduler  │         │
│  │  (scan cmds)│  │ (Express)    │  │ (node-cron) │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                │                │                 │
│  ┌──────▼──────────────────────────────────────┐           │
│  │              Service Layer                   │           │
│  │  ┌─────────────┐  ┌─────────────────────┐  │           │
│  │  │  OpenCLI     │  │  Price Detector     │  │           │
│  │  │  (scraping)  │  │  & Notifier         │  │           │
│  │  └─────────────┘  └─────────────────────┘  │           │
│  └────────────────────────┬────────────────────┘           │
│                           │                                 │
│  ┌────────────────────────▼────────────────────┐           │
│  │              MySQL 8 Database                  │           │
│  │  products, price_history, alerts, scan_logs    │           │
│  └───────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │   Chrome Browser       │
              │   (with OpenCLI        │
              │    Extension)          │
              └───────────────────────┘
\`\`\`
```

- [ ] **Step 2: Commit deployment guide**

```bash
git add README.md
git commit -m "docs: add comprehensive deployment guide"
```

---

### Task 9: Docker Files

**Files:**
- Create: `crossborder-monitor/docker/Dockerfile`
- Create: `crossborder-monitor/docker-compose.yml`
- Create: `crossborder-monitor/.env.example`

- [ ] **Step 1: Create .env.example**

```bash
# Database
DB_HOST=mysql
DB_PORT=3306
DB_USER=monitor
DB_PASSWORD=change_me_to_a_secure_password
DB_NAME=crossborder_monitor

# Application
PORT=3000

# Telegram Notifications (optional)
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=

# Debug
DEBUG=0
```

- [ ] **Step 2: Create Dockerfile**

```dockerfile
# docker/Dockerfile
FROM node:21-slim

LABEL maintainer="your-name"
LABEL description="Cross-Border Product Monitor"

# Install dependencies
RUN apt-get update && apt-get install -y \
    chromium \
    chromium-sandbox \
    fonts-noto-cjk \
    && rm -rf /var/lib/apt/lists/*

# Install OpenCLI globally
RUN npm install -g @jackwener/opencli

# Create app directory
WORKDIR /app

# Copy package files
COPY package*.json ./
COPY tsconfig.json ./

# Install dependencies
RUN npm ci --only=production && npm cache clean --force

# Copy source and build
COPY src/ ./src/
RUN npm run build

# Create non-root user
RUN useradd -m -s /bin/bash appuser && \
    mkdir -p /home/appuser/.config/opencli && \
    chown -R appuser:appuser /app

USER appuser

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# Start
CMD ["node", "dist/main.js", "api"]
```

- [ ] **Step 3: Create docker-compose.yml**

```yaml
# docker-compose.yml
version: '3.9'

services:
  mysql:
    image: mysql:8.0
    container_name: monitor-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: crossborder_monitor
      MYSQL_USER: monitor
      MYSQL_PASSWORD: ${DB_PASSWORD:-monitor_password}
    volumes:
      - mysql_data:/var/lib/mysql
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
    container_name: monitor-app
    restart: unless-stopped
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      DB_HOST: mysql
      DB_PORT: 3306
      DB_USER: monitor
      DB_PASSWORD: ${DB_PASSWORD:-monitor_password}
      DB_NAME: crossborder_monitor
      PORT: 3000
      TELEGRAM_BOT_TOKEN: ${TELEGRAM_BOT_TOKEN:-}
      TELEGRAM_CHAT_ID: ${TELEGRAM_CHAT_ID:-}
      DEBUG: ${DEBUG:-0}
    ports:
      - "3000:3000"
    volumes:
      # Mount Chrome profile for logged-in state
      - chrome_data:/home/appuser/.config/google-chrome
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  mysql_data:
  chrome_data:
```

- [ ] **Step 4: Create .dockerignore**

```bash
# .dockerignore
node_modules
dist
.git
*.log
.env
.env.*
!.env.example
```

- [ ] **Step 5: Commit Docker files**

```bash
git add docker/ docker-compose.yml .env.example .dockerignore
git commit -m "feat: add Docker support for one-click deployment"
```

---

## Implementation Notes

### Windows-Specific Considerations

1. **Chrome Path**: On Windows, Chrome is typically at `C:\Program Files\Google\Chrome\Application\chrome.exe`
2. **Shell**: Use `cmd` or `powershell` instead of bash
3. **Path Separator**: Use `path.join()` which handles this automatically (our code uses `node:path`)
4. **Environment Variables**: Use `set KEY=value` in cmd or `$env:KEY=value` in PowerShell

### Cron Expressions

Common patterns for `node-cron`:
- `*/15 * * * *` - Every 15 minutes
- `0 */6 * * *` - Every 6 hours
- `0 0 * * *` - Daily at midnight
- `0 9,18 * * *` - Twice daily at 9 AM and 6 PM

### Error Handling

All commands should:
1. Catch errors and log them
2. Update scan_logs with error details
3. Return appropriate exit codes
4. Not crash the scheduler

### Extending to New Platforms

To add a new platform:

1. Create `src/commands/scan-{platform}.ts`
2. Implement `scanPlatform()` function
3. Add platform case in `src/api.ts`
4. Add example cron task in `src/scheduler.ts`
```

---

## Next Steps

1. **Review this plan** - Make sure the architecture fits your needs
2. **Approve the design** - Confirm the file structure and approach
3. **Start implementation** - We'll build incrementally, testing as we go

Does this technical plan look right? Should I proceed with implementation?

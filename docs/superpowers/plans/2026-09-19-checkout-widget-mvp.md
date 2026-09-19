# Checkout Widget MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an end-to-end, Stripe-test-mode demo of an embeddable checkout widget that lets a merchant accept a payment while the platform automatically collects a small `application_fee_amount` via Stripe Connect.

**Architecture:** An npm-workspaces monorepo with three packages — `server` (Express API handling merchant auth, Stripe Connect onboarding, PaymentIntent creation, and webhooks), `widget` (a vanilla-JS embeddable checkout form built with Stripe Elements), and `dashboard` (a React app for merchant signup, Stripe onboarding, and viewing transactions). Stripe is the only party that ever moves money; the platform only ever reads/writes its own SQLite records and calls the Stripe API.

**Tech Stack:** Node.js, Express, `better-sqlite3`, Stripe Node SDK, `bcrypt`, `jsonwebtoken`, Vite, vanilla JS + `@stripe/stripe-js` (widget), React + `react-router-dom` (dashboard), Vitest + `supertest` + `@testing-library/react` for tests.

**Spec:** `docs/superpowers/specs/2026-09-19-checkout-widget-design.md`

## Global Constraints

- Everything runs in **Stripe test mode** only — no live keys, no real money movement.
- Monorepo uses **npm workspaces** with packages under `packages/*`.
- Stripe Connect account type is **Express**; onboarding uses Stripe-hosted **Account Links** (the current Stripe-recommended flow for Express accounts), not raw OAuth redirect URLs.
- Platform fee formula: `max(1, round(amountCents * 0.001))` — 0.1% of the transaction, minimum 1 cent (Stripe only supports whole-cent amounts, so "1/1000th of a cent" isn't representable; this is the closest tiny-fee equivalent).
- No production onboarding, legal ToS, fraud detection, or currency conversion — explicitly out of scope per the spec.
- Auth between dashboard and server uses a **Bearer JWT** (not cookies/sessions) to avoid cross-origin cookie complications between the Vite dev server and the API.

---

## File Structure

```
Income/
├── package.json                  # npm workspaces root
├── .gitignore
├── .env.example
├── packages/
│   ├── server/
│   │   ├── package.json
│   │   ├── src/
│   │   │   ├── app.js            # Express app (exported, no listen())
│   │   │   ├── index.js          # Starts the server (listen())
│   │   │   ├── db.js             # better-sqlite3 setup + schema
│   │   │   ├── stripeClient.js   # Stripe SDK singleton
│   │   │   ├── auth.js           # JWT sign/verify + bcrypt helpers
│   │   │   └── routes/
│   │   │       ├── authRoutes.js       # POST /api/signup, /api/login
│   │   │       ├── connectRoutes.js    # POST /api/connect/start
│   │   │       ├── checkoutRoutes.js   # POST /api/checkout
│   │   │       ├── transactionRoutes.js# GET /api/transactions
│   │   │       └── webhookRoutes.js    # POST /webhooks/stripe
│   │   └── test/
│   │       ├── health.test.js
│   │       ├── db.test.js
│   │       ├── authRoutes.test.js
│   │       ├── connectRoutes.test.js
│   │       ├── checkoutRoutes.test.js
│   │       ├── transactionRoutes.test.js
│   │       └── webhookRoutes.test.js
│   ├── widget/
│   │   ├── package.json
│   │   ├── vite.config.js
│   │   ├── src/
│   │   │   └── widget.js         # window.IncomeCheckout.mount(...)
│   │   └── test/
│   │       └── widget.test.js
│   └── dashboard/
│       ├── package.json
│       ├── vite.config.js
│       ├── index.html
│       ├── src/
│       │   ├── main.jsx
│       │   ├── App.jsx
│       │   ├── api.js            # fetch wrapper for the server API
│       │   └── pages/
│       │       ├── Signup.jsx
│       │       ├── Login.jsx
│       │       ├── ConnectStripe.jsx
│       │       ├── EmbedSnippet.jsx
│       │       └── Transactions.jsx
│       └── test/
│           ├── Signup.test.jsx
│           └── Transactions.test.jsx
└── test/
    └── e2e/
        └── test-merchant-site.html  # static page for manual E2E test
```

---

### Task 1: Monorepo scaffolding

**Files:**
- Create: `package.json` (root)
- Create: `.gitignore`
- Create: `.env.example`

**Interfaces:**
- Produces: root workspaces config that later tasks' `packages/*/package.json` register into.

- [ ] **Step 1: Create the root `package.json`**

```json
{
  "name": "income",
  "private": true,
  "version": "0.0.0",
  "workspaces": [
    "packages/*"
  ],
  "scripts": {
    "test": "npm test --workspaces --if-present"
  }
}
```

- [ ] **Step 2: Create `.gitignore`**

```
node_modules/
dist/
*.db
.env
.DS_Store
```

- [ ] **Step 3: Create `.env.example`**

```
STRIPE_SECRET_KEY=sk_test_...
STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
JWT_SECRET=dev-secret-change-me
PORT=3000
```

- [ ] **Step 4: Verify workspaces resolve**

Run: `npm install` (from repo root)
Expected: completes with no errors (no packages exist yet, so it just sets up the root `node_modules`).

- [ ] **Step 5: Commit**

```bash
git add package.json .gitignore .env.example
git commit -m "chore: scaffold npm workspaces monorepo"
```

---

### Task 2: Server bootstrap + health check

**Files:**
- Create: `packages/server/package.json`
- Create: `packages/server/src/app.js`
- Create: `packages/server/src/index.js`
- Test: `packages/server/test/health.test.js`

**Interfaces:**
- Produces: `app.js` exports `app` (an Express instance) for tests and for `index.js` to listen on.

- [ ] **Step 1: Create `packages/server/package.json`**

```json
{
  "name": "@income/server",
  "version": "0.0.0",
  "type": "module",
  "main": "src/index.js",
  "scripts": {
    "dev": "node src/index.js",
    "test": "vitest run"
  },
  "dependencies": {
    "express": "^4.19.2",
    "dotenv": "^16.4.5",
    "better-sqlite3": "^11.3.0",
    "bcrypt": "^5.1.1",
    "jsonwebtoken": "^9.0.2",
    "stripe": "^16.12.0",
    "cors": "^2.8.5"
  },
  "devDependencies": {
    "vitest": "^2.1.1",
    "supertest": "^7.0.0"
  }
}
```

- [ ] **Step 2: Write the failing test**

```javascript
// packages/server/test/health.test.js
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import app from '../src/app.js';

describe('GET /health', () => {
  it('returns 200 and status ok', async () => {
    const res = await request(app).get('/health');
    expect(res.status).toBe(200);
    expect(res.body).toEqual({ status: 'ok' });
  });
});
```

- [ ] **Step 3: Install deps and run test to verify it fails**

Run: `cd packages/server && npm install && npx vitest run`
Expected: FAIL — `Cannot find module '../src/app.js'`

- [ ] **Step 4: Write minimal implementation**

```javascript
// packages/server/src/app.js
import express from 'express';
import cors from 'cors';

const app = express();
app.use(cors());

app.get('/health', (req, res) => {
  res.json({ status: 'ok' });
});

export default app;
```

```javascript
// packages/server/src/index.js
import 'dotenv/config';
import app from './app.js';

const port = process.env.PORT || 3000;
app.listen(port, () => {
  console.log(`Server listening on port ${port}`);
});
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npx vitest run` (from `packages/server`)
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add packages/server/package.json packages/server/src/app.js packages/server/src/index.js packages/server/test/health.test.js
git commit -m "feat(server): bootstrap Express app with health check"
```

---

### Task 3: SQLite database + schema

**Files:**
- Create: `packages/server/src/db.js`
- Test: `packages/server/test/db.test.js`

**Interfaces:**
- Produces: `createDb(path)` returning a `better-sqlite3` `Database` instance with `merchants` and `transactions` tables created. `path === ':memory:'` for tests.
- Produces: default export `db` — a singleton `Database` instance created from `process.env.DB_PATH || 'income.db'`, for route modules to import directly.

- [ ] **Step 1: Write the failing test**

```javascript
// packages/server/test/db.test.js
import { describe, it, expect } from 'vitest';
import { createDb } from '../src/db.js';

describe('createDb', () => {
  it('creates a merchants table and allows inserting a row', () => {
    const db = createDb(':memory:');
    const info = db.prepare(
      `INSERT INTO merchants (email, password_hash) VALUES (?, ?)`
    ).run('merchant@example.com', 'hashed');
    expect(info.changes).toBe(1);

    const row = db.prepare('SELECT * FROM merchants WHERE id = ?').get(info.lastInsertRowid);
    expect(row.email).toBe('merchant@example.com');
    expect(row.stripe_account_id).toBeNull();
    expect(row.onboarded).toBe(0);
  });

  it('creates a transactions table linked to merchants', () => {
    const db = createDb(':memory:');
    const { lastInsertRowid: merchantId } = db.prepare(
      `INSERT INTO merchants (email, password_hash) VALUES (?, ?)`
    ).run('m@example.com', 'hashed');

    const info = db.prepare(
      `INSERT INTO transactions (merchant_id, amount_cents, fee_cents, stripe_payment_intent_id, status)
       VALUES (?, ?, ?, ?, ?)`
    ).run(merchantId, 1000, 1, 'pi_123', 'succeeded');
    expect(info.changes).toBe(1);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/db.test.js`
Expected: FAIL — `createDb is not a function` / module not found

- [ ] **Step 3: Write minimal implementation**

```javascript
// packages/server/src/db.js
import Database from 'better-sqlite3';

export function createDb(path) {
  const db = new Database(path);
  db.pragma('journal_mode = WAL');

  db.exec(`
    CREATE TABLE IF NOT EXISTS merchants (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      email TEXT UNIQUE NOT NULL,
      password_hash TEXT NOT NULL,
      stripe_account_id TEXT,
      onboarded INTEGER NOT NULL DEFAULT 0,
      created_at TEXT NOT NULL DEFAULT (datetime('now'))
    );

    CREATE TABLE IF NOT EXISTS transactions (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      merchant_id INTEGER NOT NULL REFERENCES merchants(id),
      amount_cents INTEGER NOT NULL,
      fee_cents INTEGER NOT NULL,
      stripe_payment_intent_id TEXT NOT NULL,
      status TEXT NOT NULL,
      created_at TEXT NOT NULL DEFAULT (datetime('now'))
    );
  `);

  return db;
}

const db = createDb(process.env.DB_PATH || 'income.db');
export default db;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/db.test.js`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/server/src/db.js packages/server/test/db.test.js
git commit -m "feat(server): add SQLite schema for merchants and transactions"
```

---

### Task 4: Auth helpers (password hashing + JWT)

**Files:**
- Create: `packages/server/src/auth.js`
- Test: `packages/server/test/auth.test.js`

**Interfaces:**
- Produces: `hashPassword(plain): Promise<string>`, `verifyPassword(plain, hash): Promise<boolean>`, `signToken(merchantId): string`, `verifyToken(token): { merchantId: number }` (throws on invalid/expired).

- [ ] **Step 1: Write the failing test**

```javascript
// packages/server/test/auth.test.js
import { describe, it, expect } from 'vitest';
import { hashPassword, verifyPassword, signToken, verifyToken } from '../src/auth.js';

describe('auth helpers', () => {
  it('hashes and verifies a password', async () => {
    const hash = await hashPassword('correct-horse');
    expect(hash).not.toBe('correct-horse');
    expect(await verifyPassword('correct-horse', hash)).toBe(true);
    expect(await verifyPassword('wrong', hash)).toBe(false);
  });

  it('signs and verifies a JWT carrying the merchant id', () => {
    const token = signToken(42);
    const decoded = verifyToken(token);
    expect(decoded.merchantId).toBe(42);
  });

  it('throws on an invalid token', () => {
    expect(() => verifyToken('not-a-real-token')).toThrow();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/auth.test.js`
Expected: FAIL — module not found

- [ ] **Step 3: Write minimal implementation**

```javascript
// packages/server/src/auth.js
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';

const SECRET = process.env.JWT_SECRET || 'dev-secret-change-me';

export async function hashPassword(plain) {
  return bcrypt.hash(plain, 10);
}

export async function verifyPassword(plain, hash) {
  return bcrypt.compare(plain, hash);
}

export function signToken(merchantId) {
  return jwt.sign({ merchantId }, SECRET, { expiresIn: '7d' });
}

export function verifyToken(token) {
  return jwt.verify(token, SECRET);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/auth.test.js`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/server/src/auth.js packages/server/test/auth.test.js
git commit -m "feat(server): add password hashing and JWT helpers"
```

---

### Task 5: Signup/login routes

**Files:**
- Create: `packages/server/src/routes/authRoutes.js`
- Modify: `packages/server/src/app.js` (mount the router, accept an injectable `db`)
- Test: `packages/server/test/authRoutes.test.js`

**Interfaces:**
- Consumes: `createDb` from Task 3, `hashPassword`/`verifyPassword`/`signToken` from Task 4.
- Produces: `createApp(db)` — refactor of `app.js` to accept a database instance (so tests use an in-memory DB instead of the real file). Routes: `POST /api/signup { email, password } -> { token }` (409 if email taken), `POST /api/login { email, password } -> { token }` (401 on bad credentials).

- [ ] **Step 1: Write the failing test**

```javascript
// packages/server/test/authRoutes.test.js
import { describe, it, expect, beforeEach } from 'vitest';
import request from 'supertest';
import { createDb } from '../src/db.js';
import { createApp } from '../src/app.js';

let app;

beforeEach(() => {
  const db = createDb(':memory:');
  app = createApp(db);
});

describe('POST /api/signup', () => {
  it('creates a merchant and returns a token', async () => {
    const res = await request(app)
      .post('/api/signup')
      .send({ email: 'm@example.com', password: 'secret123' });
    expect(res.status).toBe(201);
    expect(res.body.token).toBeTypeOf('string');
  });

  it('rejects a duplicate email with 409', async () => {
    await request(app).post('/api/signup').send({ email: 'm@example.com', password: 'secret123' });
    const res = await request(app).post('/api/signup').send({ email: 'm@example.com', password: 'other' });
    expect(res.status).toBe(409);
  });
});

describe('POST /api/login', () => {
  it('logs in with correct credentials', async () => {
    await request(app).post('/api/signup').send({ email: 'm@example.com', password: 'secret123' });
    const res = await request(app).post('/api/login').send({ email: 'm@example.com', password: 'secret123' });
    expect(res.status).toBe(200);
    expect(res.body.token).toBeTypeOf('string');
  });

  it('rejects wrong password with 401', async () => {
    await request(app).post('/api/signup').send({ email: 'm@example.com', password: 'secret123' });
    const res = await request(app).post('/api/login').send({ email: 'm@example.com', password: 'wrong' });
    expect(res.status).toBe(401);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/authRoutes.test.js`
Expected: FAIL — `createApp is not exported`

- [ ] **Step 3: Write minimal implementation**

```javascript
// packages/server/src/routes/authRoutes.js
import { Router } from 'express';
import { hashPassword, verifyPassword, signToken } from '../auth.js';

export function authRoutes(db) {
  const router = Router();

  router.post('/signup', async (req, res) => {
    const { email, password } = req.body;
    const existing = db.prepare('SELECT id FROM merchants WHERE email = ?').get(email);
    if (existing) return res.status(409).json({ error: 'email already registered' });

    const passwordHash = await hashPassword(password);
    const info = db
      .prepare('INSERT INTO merchants (email, password_hash) VALUES (?, ?)')
      .run(email, passwordHash);

    res.status(201).json({ token: signToken(info.lastInsertRowid) });
  });

  router.post('/login', async (req, res) => {
    const { email, password } = req.body;
    const merchant = db.prepare('SELECT * FROM merchants WHERE email = ?').get(email);
    if (!merchant) return res.status(401).json({ error: 'invalid credentials' });

    const valid = await verifyPassword(password, merchant.password_hash);
    if (!valid) return res.status(401).json({ error: 'invalid credentials' });

    res.json({ token: signToken(merchant.id) });
  });

  return router;
}
```

```javascript
// packages/server/src/app.js
import express from 'express';
import cors from 'cors';
import { authRoutes } from './routes/authRoutes.js';

export function createApp(db) {
  const app = express();
  app.use(cors());
  app.use(express.json());

  app.get('/health', (req, res) => {
    res.json({ status: 'ok' });
  });

  app.use('/api', authRoutes(db));

  return app;
}

export default createApp;
```

```javascript
// packages/server/src/index.js
import 'dotenv/config';
import db from './db.js';
import { createApp } from './app.js';

const app = createApp(db);
const port = process.env.PORT || 3000;
app.listen(port, () => {
  console.log(`Server listening on port ${port}`);
});
```

- [ ] **Step 4: Update the health check test to use `createApp`**

```javascript
// packages/server/test/health.test.js
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import { createDb } from '../src/db.js';
import { createApp } from '../src/app.js';

describe('GET /health', () => {
  it('returns 200 and status ok', async () => {
    const app = createApp(createDb(':memory:'));
    const res = await request(app).get('/health');
    expect(res.status).toBe(200);
    expect(res.body).toEqual({ status: 'ok' });
  });
});
```

- [ ] **Step 5: Run all server tests to verify they pass**

Run: `npx vitest run`
Expected: PASS (health.test.js and authRoutes.test.js)

- [ ] **Step 6: Commit**

```bash
git add packages/server/src/app.js packages/server/src/index.js packages/server/src/routes/authRoutes.js packages/server/test/authRoutes.test.js packages/server/test/health.test.js
git commit -m "feat(server): add merchant signup/login routes"
```

---

### Task 6: Auth middleware for protected routes

**Files:**
- Create: `packages/server/src/requireAuth.js`
- Test: `packages/server/test/requireAuth.test.js`

**Interfaces:**
- Consumes: `verifyToken` from Task 4.
- Produces: `requireAuth(req, res, next)` Express middleware. On success, sets `req.merchantId`. On missing/invalid `Authorization: Bearer <token>` header, responds `401 { error: 'unauthorized' }`.

- [ ] **Step 1: Write the failing test**

```javascript
// packages/server/test/requireAuth.test.js
import { describe, it, expect } from 'vitest';
import express from 'express';
import request from 'supertest';
import { signToken } from '../src/auth.js';
import { requireAuth } from '../src/requireAuth.js';

function buildTestApp() {
  const app = express();
  app.get('/protected', requireAuth, (req, res) => {
    res.json({ merchantId: req.merchantId });
  });
  return app;
}

describe('requireAuth', () => {
  it('rejects requests with no Authorization header', async () => {
    const res = await request(buildTestApp()).get('/protected');
    expect(res.status).toBe(401);
  });

  it('rejects requests with an invalid token', async () => {
    const res = await request(buildTestApp())
      .get('/protected')
      .set('Authorization', 'Bearer garbage');
    expect(res.status).toBe(401);
  });

  it('allows requests with a valid token and sets req.merchantId', async () => {
    const token = signToken(7);
    const res = await request(buildTestApp())
      .get('/protected')
      .set('Authorization', `Bearer ${token}`);
    expect(res.status).toBe(200);
    expect(res.body.merchantId).toBe(7);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/requireAuth.test.js`
Expected: FAIL — module not found

- [ ] **Step 3: Write minimal implementation**

```javascript
// packages/server/src/requireAuth.js
import { verifyToken } from './auth.js';

export function requireAuth(req, res, next) {
  const header = req.headers.authorization || '';
  const [scheme, token] = header.split(' ');

  if (scheme !== 'Bearer' || !token) {
    return res.status(401).json({ error: 'unauthorized' });
  }

  try {
    const { merchantId } = verifyToken(token);
    req.merchantId = merchantId;
    next();
  } catch {
    res.status(401).json({ error: 'unauthorized' });
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run test/requireAuth.test.js`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/server/src/requireAuth.js packages/server/test/requireAuth.test.js
git commit -m "feat(server): add requireAuth middleware"
```

---

### Task 7: Stripe client wrapper + Connect onboarding route

**Files:**
- Create: `packages/server/src/stripeClient.js`
- Create: `packages/server/src/routes/connectRoutes.js`
- Modify: `packages/server/src/app.js` (mount router, accept injectable `stripe`)
- Test: `packages/server/test/connectRoutes.test.js`

**Interfaces:**
- Consumes: `requireAuth` from Task 6.
- Produces: `createApp(db, stripe)` (extends `createApp` to accept a Stripe client, defaulting to the real one). `POST /api/connect/start` (auth required) → creates a Stripe Express account if the merchant doesn't have one, creates an Account Link, returns `{ url }`.

- [ ] **Step 1: Write the failing test**

```javascript
// packages/server/test/connectRoutes.test.js
import { describe, it, expect, vi, beforeEach } from 'vitest';
import request from 'supertest';
import { createDb } from '../src/db.js';
import { createApp } from '../src/app.js';
import { signToken } from '../src/auth.js';

let db, app, stripe, token, merchantId;

beforeEach(() => {
  db = createDb(':memory:');
  const info = db
    .prepare('INSERT INTO merchants (email, password_hash) VALUES (?, ?)')
    .run('m@example.com', 'hash');
  merchantId = info.lastInsertRowid;
  token = signToken(merchantId);

  stripe = {
    accounts: { create: vi.fn().mockResolvedValue({ id: 'acct_123' }) },
    accountLinks: { create: vi.fn().mockResolvedValue({ url: 'https://connect.stripe.com/setup/acct_123' }) },
  };

  app = createApp(db, stripe);
});

describe('POST /api/connect/start', () => {
  it('requires auth', async () => {
    const res = await request(app).post('/api/connect/start');
    expect(res.status).toBe(401);
  });

  it('creates a Stripe Express account and returns an onboarding URL', async () => {
    const res = await request(app)
      .post('/api/connect/start')
      .set('Authorization', `Bearer ${token}`);

    expect(res.status).toBe(200);
    expect(res.body.url).toBe('https://connect.stripe.com/setup/acct_123');
    expect(stripe.accounts.create).toHaveBeenCalledWith({ type: 'express' });

    const row = db.prepare('SELECT stripe_account_id FROM merchants WHERE id = ?').get(merchantId);
    expect(row.stripe_account_id).toBe('acct_123');
  });

  it('reuses an existing Stripe account instead of creating a new one', async () => {
    db.prepare('UPDATE merchants SET stripe_account_id = ? WHERE id = ?').run('acct_existing', merchantId);

    const res = await request(app)
      .post('/api/connect/start')
      .set('Authorization', `Bearer ${token}`);

    expect(res.status).toBe(200);
    expect(stripe.accounts.create).not.toHaveBeenCalled();
    expect(stripe.accountLinks.create).toHaveBeenCalledWith(
      expect.objectContaining({ account: 'acct_existing' })
    );
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/connectRoutes.test.js`
Expected: FAIL — `createApp` doesn't accept a `stripe` argument / route missing

- [ ] **Step 3: Write minimal implementation**

```javascript
// packages/server/src/stripeClient.js
import Stripe from 'stripe';

export function createStripeClient() {
  return new Stripe(process.env.STRIPE_SECRET_KEY);
}

const stripe = createStripeClient();
export default stripe;
```

```javascript
// packages/server/src/routes/connectRoutes.js
import { Router } from 'express';
import { requireAuth } from '../requireAuth.js';

export function connectRoutes(db, stripe) {
  const router = Router();

  router.post('/connect/start', requireAuth, async (req, res) => {
    const merchant = db.prepare('SELECT * FROM merchants WHERE id = ?').get(req.merchantId);

    let accountId = merchant.stripe_account_id;
    if (!accountId) {
      const account = await stripe.accounts.create({ type: 'express' });
      accountId = account.id;
      db.prepare('UPDATE merchants SET stripe_account_id = ? WHERE id = ?').run(accountId, merchant.id);
    }

    const accountLink = await stripe.accountLinks.create({
      account: accountId,
      refresh_url: `${process.env.APP_URL || 'http://localhost:5173'}/connect`,
      return_url: `${process.env.APP_URL || 'http://localhost:5173'}/embed`,
      type: 'account_onboarding',
    });

    res.json({ url: accountLink.url });
  });

  return router;
}
```

```javascript
// packages/server/src/app.js
import express from 'express';
import cors from 'cors';
import { authRoutes } from './routes/authRoutes.js';
import { connectRoutes } from './routes/connectRoutes.js';
import defaultStripe from './stripeClient.js';

export function createApp(db, stripe = defaultStripe) {
  const app = express();
  app.use(cors());
  app.use(express.json());

  app.get('/health', (req, res) => {
    res.json({ status: 'ok' });
  });

  app.use('/api', authRoutes(db));
  app.use('/api', connectRoutes(db, stripe));

  return app;
}

export default createApp;
```

```javascript
// packages/server/src/index.js
import 'dotenv/config';
import db from './db.js';
import { createApp } from './app.js';

const app = createApp(db);
const port = process.env.PORT || 3000;
app.listen(port, () => {
  console.log(`Server listening on port ${port}`);
});
```

- [ ] **Step 4: Run all server tests to verify they pass**

Run: `npx vitest run`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/server/src/stripeClient.js packages/server/src/routes/connectRoutes.js packages/server/src/app.js packages/server/src/index.js packages/server/test/connectRoutes.test.js
git commit -m "feat(server): add Stripe Connect Express account onboarding"
```

---

### Task 8: Stripe webhook handler (account.updated, payment_intent events)

**Files:**
- Create: `packages/server/src/routes/webhookRoutes.js`
- Modify: `packages/server/src/app.js` (mount webhook router; must read raw body for signature verification, so it's mounted *before* `express.json()`)
- Test: `packages/server/test/webhookRoutes.test.js`

**Interfaces:**
- Produces: `POST /webhooks/stripe`. On `account.updated` with `charges_enabled: true`, sets `merchants.onboarded = 1` for the matching `stripe_account_id`. On `payment_intent.succeeded`, inserts a row into `transactions` using metadata (`merchantId`) and amounts from the event, with `status = 'succeeded'`.

- [ ] **Step 1: Write the failing test**

```javascript
// packages/server/test/webhookRoutes.test.js
import { describe, it, expect, vi, beforeEach } from 'vitest';
import request from 'supertest';
import { createDb } from '../src/db.js';
import { createApp } from '../src/app.js';

let db, app, stripe, merchantId;

beforeEach(() => {
  db = createDb(':memory:');
  const info = db
    .prepare('INSERT INTO merchants (email, password_hash, stripe_account_id) VALUES (?, ?, ?)')
    .run('m@example.com', 'hash', 'acct_123');
  merchantId = info.lastInsertRowid;

  stripe = {
    webhooks: {
      // Test double: skips real signature verification, just parses the body we sent.
      constructEvent: vi.fn((rawBody) => JSON.parse(rawBody)),
    },
  };

  app = createApp(db, stripe);
});

describe('POST /webhooks/stripe', () => {
  it('marks a merchant onboarded on account.updated with charges_enabled', async () => {
    const event = {
      type: 'account.updated',
      data: { object: { id: 'acct_123', charges_enabled: true } },
    };

    const res = await request(app)
      .post('/webhooks/stripe')
      .set('stripe-signature', 'test-sig')
      .send(event);

    expect(res.status).toBe(200);
    const row = db.prepare('SELECT onboarded FROM merchants WHERE id = ?').get(merchantId);
    expect(row.onboarded).toBe(1);
  });

  it('records a transaction on payment_intent.succeeded', async () => {
    const event = {
      type: 'payment_intent.succeeded',
      data: {
        object: {
          id: 'pi_456',
          amount: 1000,
          application_fee_amount: 1,
          metadata: { merchantId: String(merchantId) },
        },
      },
    };

    const res = await request(app)
      .post('/webhooks/stripe')
      .set('stripe-signature', 'test-sig')
      .send(event);

    expect(res.status).toBe(200);
    const row = db.prepare('SELECT * FROM transactions WHERE stripe_payment_intent_id = ?').get('pi_456');
    expect(row.amount_cents).toBe(1000);
    expect(row.fee_cents).toBe(1);
    expect(row.status).toBe('succeeded');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/webhookRoutes.test.js`
Expected: FAIL — route not found (404)

- [ ] **Step 3: Write minimal implementation**

```javascript
// packages/server/src/routes/webhookRoutes.js
import { Router } from 'express';
import express from 'express';

export function webhookRoutes(db, stripe) {
  const router = Router();

  router.post('/stripe', express.raw({ type: 'application/json' }), (req, res) => {
    let event;
    try {
      event = stripe.webhooks.constructEvent(
        req.body,
        req.headers['stripe-signature'],
        process.env.STRIPE_WEBHOOK_SECRET
      );
    } catch (err) {
      return res.status(400).send(`Webhook signature verification failed: ${err.message}`);
    }

    if (event.type === 'account.updated') {
      const account = event.data.object;
      if (account.charges_enabled) {
        db.prepare('UPDATE merchants SET onboarded = 1 WHERE stripe_account_id = ?').run(account.id);
      }
    }

    if (event.type === 'payment_intent.succeeded') {
      const pi = event.data.object;
      db.prepare(
        `INSERT INTO transactions (merchant_id, amount_cents, fee_cents, stripe_payment_intent_id, status)
         VALUES (?, ?, ?, ?, 'succeeded')`
      ).run(Number(pi.metadata.merchantId), pi.amount, pi.application_fee_amount, pi.id);
    }

    res.json({ received: true });
  });

  return router;
}
```

```javascript
// packages/server/src/app.js
import express from 'express';
import cors from 'cors';
import { authRoutes } from './routes/authRoutes.js';
import { connectRoutes } from './routes/connectRoutes.js';
import { webhookRoutes } from './routes/webhookRoutes.js';
import defaultStripe from './stripeClient.js';

export function createApp(db, stripe = defaultStripe) {
  const app = express();
  app.use(cors());

  // Mounted before express.json() so the raw body is available for signature verification.
  app.use('/webhooks', webhookRoutes(db, stripe));

  app.use(express.json());

  app.get('/health', (req, res) => {
    res.json({ status: 'ok' });
  });

  app.use('/api', authRoutes(db));
  app.use('/api', connectRoutes(db, stripe));

  return app;
}

export default createApp;
```

Note: in this test, supertest's `.send(event)` sends JSON with `Content-Type: application/json`, and `express.raw({ type: 'application/json' })` captures it as a `Buffer` in `req.body`. The test double's `constructEvent` calls `JSON.parse(rawBody)` on that buffer, which works because `JSON.parse` accepts a `Buffer` (it's coerced to a string).

- [ ] **Step 4: Run all server tests to verify they pass**

Run: `npx vitest run`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/server/src/routes/webhookRoutes.js packages/server/src/app.js packages/server/test/webhookRoutes.test.js
git commit -m "feat(server): add Stripe webhook handler for onboarding and payments"
```

---

### Task 9: Checkout route (PaymentIntent with application fee)

**Files:**
- Create: `packages/server/src/routes/checkoutRoutes.js`
- Modify: `packages/server/src/app.js` (mount router)
- Test: `packages/server/test/checkoutRoutes.test.js`

**Interfaces:**
- Produces: `POST /api/checkout { merchantId, amountCents } -> { clientSecret }` (no auth — called by the public widget on a customer's page). Looks up the merchant's `stripe_account_id`; 404s if the merchant doesn't exist or isn't onboarded. Computes `feeCents = max(1, round(amountCents * 0.001))` and creates a PaymentIntent via `stripe.paymentIntents.create(..., { stripeAccount: merchant.stripe_account_id })`.

- [ ] **Step 1: Write the failing test**

```javascript
// packages/server/test/checkoutRoutes.test.js
import { describe, it, expect, vi, beforeEach } from 'vitest';
import request from 'supertest';
import { createDb } from '../src/db.js';
import { createApp } from '../src/app.js';

let db, app, stripe, merchantId;

beforeEach(() => {
  db = createDb(':memory:');

  stripe = {
    paymentIntents: {
      create: vi.fn().mockResolvedValue({ client_secret: 'pi_secret_abc' }),
    },
  };
  app = createApp(db, stripe);
});

describe('POST /api/checkout', () => {
  it('returns 404 for an unknown merchant', async () => {
    const res = await request(app).post('/api/checkout').send({ merchantId: 999, amountCents: 1000 });
    expect(res.status).toBe(404);
  });

  it('returns 404 for a merchant that has not finished onboarding', async () => {
    const info = db
      .prepare('INSERT INTO merchants (email, password_hash, stripe_account_id, onboarded) VALUES (?, ?, ?, 0)')
      .run('m@example.com', 'hash', 'acct_123');
    const res = await request(app).post('/api/checkout').send({ merchantId: info.lastInsertRowid, amountCents: 1000 });
    expect(res.status).toBe(404);
  });

  it('creates a PaymentIntent on the connected account with the platform fee', async () => {
    const info = db
      .prepare('INSERT INTO merchants (email, password_hash, stripe_account_id, onboarded) VALUES (?, ?, ?, 1)')
      .run('m@example.com', 'hash', 'acct_123');
    merchantId = info.lastInsertRowid;

    const res = await request(app)
      .post('/api/checkout')
      .send({ merchantId, amountCents: 1000 });

    expect(res.status).toBe(200);
    expect(res.body.clientSecret).toBe('pi_secret_abc');

    expect(stripe.paymentIntents.create).toHaveBeenCalledWith(
      {
        amount: 1000,
        currency: 'usd',
        application_fee_amount: 1,
        metadata: { merchantId: String(merchantId) },
        automatic_payment_methods: { enabled: true },
      },
      { stripeAccount: 'acct_123' }
    );
  });

  it('applies a minimum fee of 1 cent even for very small amounts', async () => {
    const info = db
      .prepare('INSERT INTO merchants (email, password_hash, stripe_account_id, onboarded) VALUES (?, ?, ?, 1)')
      .run('m2@example.com', 'hash', 'acct_456');

    await request(app).post('/api/checkout').send({ merchantId: info.lastInsertRowid, amountCents: 50 });

    expect(stripe.paymentIntents.create).toHaveBeenCalledWith(
      expect.objectContaining({ application_fee_amount: 1 }),
      { stripeAccount: 'acct_456' }
    );
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/checkoutRoutes.test.js`
Expected: FAIL — route not found (404 for the "success" cases too, since nothing exists yet)

- [ ] **Step 3: Write minimal implementation**

```javascript
// packages/server/src/routes/checkoutRoutes.js
import { Router } from 'express';

export function platformFeeCents(amountCents) {
  return Math.max(1, Math.round(amountCents * 0.001));
}

export function checkoutRoutes(db, stripe) {
  const router = Router();

  router.post('/checkout', async (req, res) => {
    const { merchantId, amountCents } = req.body;
    const merchant = db.prepare('SELECT * FROM merchants WHERE id = ?').get(merchantId);

    if (!merchant || !merchant.onboarded) {
      return res.status(404).json({ error: 'merchant not found or not onboarded' });
    }

    const feeCents = platformFeeCents(amountCents);

    const paymentIntent = await stripe.paymentIntents.create(
      {
        amount: amountCents,
        currency: 'usd',
        application_fee_amount: feeCents,
        metadata: { merchantId: String(merchant.id) },
        automatic_payment_methods: { enabled: true },
      },
      { stripeAccount: merchant.stripe_account_id }
    );

    res.json({ clientSecret: paymentIntent.client_secret });
  });

  return router;
}
```

```javascript
// packages/server/src/app.js
import express from 'express';
import cors from 'cors';
import { authRoutes } from './routes/authRoutes.js';
import { connectRoutes } from './routes/connectRoutes.js';
import { webhookRoutes } from './routes/webhookRoutes.js';
import { checkoutRoutes } from './routes/checkoutRoutes.js';
import defaultStripe from './stripeClient.js';

export function createApp(db, stripe = defaultStripe) {
  const app = express();
  app.use(cors());

  app.use('/webhooks', webhookRoutes(db, stripe));

  app.use(express.json());

  app.get('/health', (req, res) => {
    res.json({ status: 'ok' });
  });

  app.use('/api', authRoutes(db));
  app.use('/api', connectRoutes(db, stripe));
  app.use('/api', checkoutRoutes(db, stripe));

  return app;
}

export default createApp;
```

- [ ] **Step 4: Run all server tests to verify they pass**

Run: `npx vitest run`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/server/src/routes/checkoutRoutes.js packages/server/src/app.js packages/server/test/checkoutRoutes.test.js
git commit -m "feat(server): add checkout route creating PaymentIntents with platform fee"
```

---

### Task 10: Transactions list route

**Files:**
- Create: `packages/server/src/routes/transactionRoutes.js`
- Modify: `packages/server/src/app.js` (mount router)
- Test: `packages/server/test/transactionRoutes.test.js`

**Interfaces:**
- Consumes: `requireAuth` from Task 6.
- Produces: `GET /api/transactions` (auth required) → `{ transactions: [{ id, amountCents, feeCents, status, createdAt }, ...] }`, scoped to `req.merchantId`, newest first.

- [ ] **Step 1: Write the failing test**

```javascript
// packages/server/test/transactionRoutes.test.js
import { describe, it, expect, beforeEach } from 'vitest';
import request from 'supertest';
import { createDb } from '../src/db.js';
import { createApp } from '../src/app.js';
import { signToken } from '../src/auth.js';

let db, app, token, merchantId;

beforeEach(() => {
  db = createDb(':memory:');
  const info = db
    .prepare('INSERT INTO merchants (email, password_hash) VALUES (?, ?)')
    .run('m@example.com', 'hash');
  merchantId = info.lastInsertRowid;
  token = signToken(merchantId);
  app = createApp(db, {});
});

describe('GET /api/transactions', () => {
  it('requires auth', async () => {
    const res = await request(app).get('/api/transactions');
    expect(res.status).toBe(401);
  });

  it('returns only the authenticated merchant\'s transactions, newest first', async () => {
    const other = db.prepare('INSERT INTO merchants (email, password_hash) VALUES (?, ?)').run('other@example.com', 'hash');

    db.prepare(
      `INSERT INTO transactions (merchant_id, amount_cents, fee_cents, stripe_payment_intent_id, status, created_at)
       VALUES (?, 1000, 1, 'pi_1', 'succeeded', '2026-01-01 00:00:00')`
    ).run(merchantId);
    db.prepare(
      `INSERT INTO transactions (merchant_id, amount_cents, fee_cents, stripe_payment_intent_id, status, created_at)
       VALUES (?, 2000, 2, 'pi_2', 'succeeded', '2026-01-02 00:00:00')`
    ).run(merchantId);
    db.prepare(
      `INSERT INTO transactions (merchant_id, amount_cents, fee_cents, stripe_payment_intent_id, status)
       VALUES (?, 5000, 5, 'pi_3', 'succeeded')`
    ).run(other.lastInsertRowid);

    const res = await request(app).get('/api/transactions').set('Authorization', `Bearer ${token}`);

    expect(res.status).toBe(200);
    expect(res.body.transactions).toHaveLength(2);
    expect(res.body.transactions[0].stripePaymentIntentId).toBe('pi_2');
    expect(res.body.transactions[1].stripePaymentIntentId).toBe('pi_1');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/transactionRoutes.test.js`
Expected: FAIL — route not found

- [ ] **Step 3: Write minimal implementation**

```javascript
// packages/server/src/routes/transactionRoutes.js
import { Router } from 'express';
import { requireAuth } from '../requireAuth.js';

export function transactionRoutes(db) {
  const router = Router();

  router.get('/transactions', requireAuth, (req, res) => {
    const rows = db
      .prepare('SELECT * FROM transactions WHERE merchant_id = ? ORDER BY created_at DESC, id DESC')
      .all(req.merchantId);

    res.json({
      transactions: rows.map((row) => ({
        id: row.id,
        amountCents: row.amount_cents,
        feeCents: row.fee_cents,
        stripePaymentIntentId: row.stripe_payment_intent_id,
        status: row.status,
        createdAt: row.created_at,
      })),
    });
  });

  return router;
}
```

```javascript
// packages/server/src/app.js
import express from 'express';
import cors from 'cors';
import { authRoutes } from './routes/authRoutes.js';
import { connectRoutes } from './routes/connectRoutes.js';
import { webhookRoutes } from './routes/webhookRoutes.js';
import { checkoutRoutes } from './routes/checkoutRoutes.js';
import { transactionRoutes } from './routes/transactionRoutes.js';
import defaultStripe from './stripeClient.js';

export function createApp(db, stripe = defaultStripe) {
  const app = express();
  app.use(cors());

  app.use('/webhooks', webhookRoutes(db, stripe));

  app.use(express.json());

  app.get('/health', (req, res) => {
    res.json({ status: 'ok' });
  });

  app.use('/api', authRoutes(db));
  app.use('/api', connectRoutes(db, stripe));
  app.use('/api', checkoutRoutes(db, stripe));
  app.use('/api', transactionRoutes(db));

  return app;
}

export default createApp;
```

- [ ] **Step 4: Run all server tests to verify they pass**

Run: `npx vitest run`
Expected: PASS (all server tests)

- [ ] **Step 5: Commit**

```bash
git add packages/server/src/routes/transactionRoutes.js packages/server/src/app.js packages/server/test/transactionRoutes.test.js
git commit -m "feat(server): add transactions list route"
```

---

### Task 11: Embeddable widget

**Files:**
- Create: `packages/widget/package.json`
- Create: `packages/widget/vite.config.js`
- Create: `packages/widget/src/widget.js`
- Test: `packages/widget/test/widget.test.js`

**Interfaces:**
- Produces: `window.IncomeCheckout.mount(elementId, { merchantId, amountCents, publishableKey, serverUrl })`. On submit, POSTs `{ merchantId, amountCents }` to `${serverUrl}/api/checkout`, then calls `stripe.confirmPayment` with the returned `clientSecret`.

- [ ] **Step 1: Create `packages/widget/package.json`**

```json
{
  "name": "@income/widget",
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "build": "vite build",
    "test": "vitest run"
  },
  "dependencies": {
    "@stripe/stripe-js": "^4.6.0"
  },
  "devDependencies": {
    "vite": "^5.4.6",
    "vitest": "^2.1.1",
    "jsdom": "^25.0.0"
  }
}
```

- [ ] **Step 2: Create `packages/widget/vite.config.js`**

```javascript
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    lib: {
      entry: 'src/widget.js',
      name: 'IncomeCheckoutBundle',
      fileName: () => 'widget.js',
      formats: ['iife'],
    },
  },
  test: {
    environment: 'jsdom',
  },
});
```

- [ ] **Step 3: Write the failing test**

```javascript
// packages/widget/test/widget.test.js
import { describe, it, expect, vi, beforeEach } from 'vitest';

const confirmPayment = vi.fn().mockResolvedValue({ error: null });
const mountCardElement = vi.fn();
const elementsMock = { create: vi.fn(() => ({ mount: mountCardElement })) };

vi.mock('@stripe/stripe-js', () => ({
  loadStripe: vi.fn().mockResolvedValue({
    elements: () => elementsMock,
    confirmPayment,
  }),
}));

import { mount } from '../src/widget.js';

beforeEach(() => {
  document.body.innerHTML = '<div id="checkout"></div>';
  global.fetch = vi.fn().mockResolvedValue({
    json: () => Promise.resolve({ clientSecret: 'pi_secret_abc' }),
  });
});

describe('mount', () => {
  it('renders a form with a submit button into the target element', async () => {
    await mount('checkout', {
      merchantId: 1,
      amountCents: 1000,
      publishableKey: 'pk_test_123',
      serverUrl: 'http://localhost:3000',
    });

    const container = document.getElementById('checkout');
    expect(container.querySelector('form')).not.toBeNull();
    expect(container.querySelector('button[type="submit"]')).not.toBeNull();
  });

  it('calls the checkout endpoint and confirms payment on submit', async () => {
    await mount('checkout', {
      merchantId: 1,
      amountCents: 1000,
      publishableKey: 'pk_test_123',
      serverUrl: 'http://localhost:3000',
    });

    const form = document.getElementById('checkout').querySelector('form');
    form.dispatchEvent(new Event('submit', { cancelable: true }));
    await new Promise((resolve) => setTimeout(resolve, 0));

    expect(global.fetch).toHaveBeenCalledWith(
      'http://localhost:3000/api/checkout',
      expect.objectContaining({
        method: 'POST',
        body: JSON.stringify({ merchantId: 1, amountCents: 1000 }),
      })
    );
    expect(confirmPayment).toHaveBeenCalled();
  });
});
```

- [ ] **Step 4: Run test to verify it fails**

Run: `cd packages/widget && npm install && npx vitest run`
Expected: FAIL — `../src/widget.js` doesn't exist

- [ ] **Step 5: Write minimal implementation**

```javascript
// packages/widget/src/widget.js
import { loadStripe } from '@stripe/stripe-js';

export async function mount(elementId, { merchantId, amountCents, publishableKey, serverUrl }) {
  const container = document.getElementById(elementId);
  container.innerHTML = '';

  const form = document.createElement('form');
  const cardMount = document.createElement('div');
  cardMount.id = `${elementId}-card`;
  const submitButton = document.createElement('button');
  submitButton.type = 'submit';
  submitButton.textContent = 'Pay';

  form.appendChild(cardMount);
  form.appendChild(submitButton);
  container.appendChild(form);

  const stripe = await loadStripe(publishableKey);
  const elements = stripe.elements();
  const card = elements.create('card');
  card.mount(`#${cardMount.id}`);

  form.addEventListener('submit', async (event) => {
    event.preventDefault();

    const response = await fetch(`${serverUrl}/api/checkout`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ merchantId, amountCents }),
    });
    const { clientSecret } = await response.json();

    await stripe.confirmPayment({
      clientSecret,
      confirmParams: { payment_method: { card } },
      redirect: 'if_required',
    });
  });
}

if (typeof window !== 'undefined') {
  window.IncomeCheckout = { mount };
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `npx vitest run`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add packages/widget/package.json packages/widget/vite.config.js packages/widget/src/widget.js packages/widget/test/widget.test.js
git commit -m "feat(widget): add embeddable checkout widget"
```

---

### Task 12: Dashboard scaffold + signup/login

**Files:**
- Create: `packages/dashboard/package.json`
- Create: `packages/dashboard/vite.config.js`
- Create: `packages/dashboard/index.html`
- Create: `packages/dashboard/src/main.jsx`
- Create: `packages/dashboard/src/App.jsx`
- Create: `packages/dashboard/src/api.js`
- Create: `packages/dashboard/src/pages/Signup.jsx`
- Create: `packages/dashboard/src/pages/Login.jsx`
- Test: `packages/dashboard/test/Signup.test.jsx`

**Interfaces:**
- Produces: `api.js` exports `signup(email, password)` and `login(email, password)`, both POSTing to `VITE_SERVER_URL` (default `http://localhost:3000`) and returning `{ token }`. On success, `Signup`/`Login` store the token in `localStorage` under `income_token` and navigate to `/connect`.

- [ ] **Step 1: Create `packages/dashboard/package.json`**

```json
{
  "name": "@income/dashboard",
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "test": "vitest run"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.26.2"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.1",
    "vite": "^5.4.6",
    "vitest": "^2.1.1",
    "jsdom": "^25.0.0",
    "@testing-library/react": "^16.0.1",
    "@testing-library/jest-dom": "^6.5.0",
    "@testing-library/user-event": "^14.5.2"
  }
}
```

- [ ] **Step 2: Create `packages/dashboard/vite.config.js`**

```javascript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    setupFiles: './test/setup.js',
  },
});
```

```javascript
// packages/dashboard/test/setup.js
import '@testing-library/jest-dom/vitest';
```

- [ ] **Step 3: Create `packages/dashboard/index.html`**

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Income Dashboard</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

- [ ] **Step 4: Write the failing test**

```jsx
// packages/dashboard/test/Signup.test.jsx
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MemoryRouter } from 'react-router-dom';
import Signup from '../src/pages/Signup.jsx';

vi.mock('../src/api.js', () => ({
  signup: vi.fn().mockResolvedValue({ token: 'jwt-abc' }),
}));

import { signup } from '../src/api.js';

beforeEach(() => {
  localStorage.clear();
  vi.clearAllMocks();
});

describe('Signup page', () => {
  it('submits email and password and stores the returned token', async () => {
    render(
      <MemoryRouter>
        <Signup />
      </MemoryRouter>
    );

    await userEvent.type(screen.getByLabelText(/email/i), 'm@example.com');
    await userEvent.type(screen.getByLabelText(/password/i), 'secret123');
    await userEvent.click(screen.getByRole('button', { name: /sign up/i }));

    await waitFor(() => {
      expect(signup).toHaveBeenCalledWith('m@example.com', 'secret123');
      expect(localStorage.getItem('income_token')).toBe('jwt-abc');
    });
  });
});
```

- [ ] **Step 5: Run test to verify it fails**

Run: `cd packages/dashboard && npm install && npx vitest run`
Expected: FAIL — `../src/pages/Signup.jsx` doesn't exist

- [ ] **Step 6: Write minimal implementation**

```javascript
// packages/dashboard/src/api.js
const SERVER_URL = import.meta.env.VITE_SERVER_URL || 'http://localhost:3000';

async function postJson(path, body) {
  const res = await fetch(`${SERVER_URL}${path}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });
  return res.json();
}

export function signup(email, password) {
  return postJson('/api/signup', { email, password });
}

export function login(email, password) {
  return postJson('/api/login', { email, password });
}
```

```jsx
// packages/dashboard/src/pages/Signup.jsx
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { signup } from '../api.js';

export default function Signup() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const navigate = useNavigate();

  async function handleSubmit(event) {
    event.preventDefault();
    const { token } = await signup(email, password);
    localStorage.setItem('income_token', token);
    navigate('/connect');
  }

  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="email">Email</label>
      <input id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} />

      <label htmlFor="password">Password</label>
      <input id="password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} />

      <button type="submit">Sign up</button>
    </form>
  );
}
```

```jsx
// packages/dashboard/src/pages/Login.jsx
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { login } from '../api.js';

export default function Login() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const navigate = useNavigate();

  async function handleSubmit(event) {
    event.preventDefault();
    const { token } = await login(email, password);
    localStorage.setItem('income_token', token);
    navigate('/connect');
  }

  return (
    <form onSubmit={handleSubmit}>
      <label htmlFor="email">Email</label>
      <input id="email" type="email" value={email} onChange={(e) => setEmail(e.target.value)} />

      <label htmlFor="password">Password</label>
      <input id="password" type="password" value={password} onChange={(e) => setPassword(e.target.value)} />

      <button type="submit">Log in</button>
    </form>
  );
}
```

```jsx
// packages/dashboard/src/App.jsx
import { Routes, Route, Navigate } from 'react-router-dom';
import Signup from './pages/Signup.jsx';
import Login from './pages/Login.jsx';

export default function App() {
  return (
    <Routes>
      <Route path="/signup" element={<Signup />} />
      <Route path="/login" element={<Login />} />
      <Route path="*" element={<Navigate to="/signup" replace />} />
    </Routes>
  );
}
```

```jsx
// packages/dashboard/src/main.jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import App from './App.jsx';

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </React.StrictMode>
);
```

- [ ] **Step 7: Run test to verify it passes**

Run: `npx vitest run`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add packages/dashboard
git commit -m "feat(dashboard): scaffold React app with signup/login pages"
```

---

### Task 13: Dashboard - Connect Stripe + Embed snippet pages

**Files:**
- Create: `packages/dashboard/src/pages/ConnectStripe.jsx`
- Create: `packages/dashboard/src/pages/EmbedSnippet.jsx`
- Modify: `packages/dashboard/src/api.js` (add authenticated request helper)
- Modify: `packages/dashboard/src/App.jsx` (register routes)
- Test: `packages/dashboard/test/ConnectStripe.test.jsx`

**Interfaces:**
- Consumes: `income_token` from `localStorage` (set in Task 12).
- Produces: `api.js` adds `connectStart()` → `POST /api/connect/start` with `Authorization: Bearer <token>`, returns `{ url }`. `ConnectStripe` renders a button that calls it and redirects `window.location` to the returned URL. `EmbedSnippet` reads `merchantId` from the JWT payload (decoded client-side, no verification needed — it's just for display) and renders the `<script>` snippet text.

- [ ] **Step 1: Write the failing test**

```jsx
// packages/dashboard/test/ConnectStripe.test.jsx
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MemoryRouter } from 'react-router-dom';
import ConnectStripe from '../src/pages/ConnectStripe.jsx';

vi.mock('../src/api.js', () => ({
  connectStart: vi.fn().mockResolvedValue({ url: 'https://connect.stripe.com/setup/acct_123' }),
}));

import { connectStart } from '../src/api.js';

beforeEach(() => {
  vi.clearAllMocks();
  delete window.location;
  window.location = { href: '' };
});

describe('ConnectStripe page', () => {
  it('redirects to the Stripe onboarding URL on click', async () => {
    render(
      <MemoryRouter>
        <ConnectStripe />
      </MemoryRouter>
    );

    await userEvent.click(screen.getByRole('button', { name: /connect with stripe/i }));

    await waitFor(() => {
      expect(connectStart).toHaveBeenCalled();
      expect(window.location.href).toBe('https://connect.stripe.com/setup/acct_123');
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/ConnectStripe.test.jsx`
Expected: FAIL — module not found

- [ ] **Step 3: Write minimal implementation**

```javascript
// packages/dashboard/src/api.js
const SERVER_URL = import.meta.env.VITE_SERVER_URL || 'http://localhost:3000';

async function postJson(path, body) {
  const res = await fetch(`${SERVER_URL}${path}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });
  return res.json();
}

async function authedPost(path) {
  const token = localStorage.getItem('income_token');
  const res = await fetch(`${SERVER_URL}${path}`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${token}` },
  });
  return res.json();
}

export function signup(email, password) {
  return postJson('/api/signup', { email, password });
}

export function login(email, password) {
  return postJson('/api/login', { email, password });
}

export function connectStart() {
  return authedPost('/api/connect/start');
}

export function decodeMerchantId() {
  const token = localStorage.getItem('income_token');
  if (!token) return null;
  const payload = JSON.parse(atob(token.split('.')[1]));
  return payload.merchantId;
}
```

```jsx
// packages/dashboard/src/pages/ConnectStripe.jsx
import { connectStart } from '../api.js';

export default function ConnectStripe() {
  async function handleClick() {
    const { url } = await connectStart();
    window.location.href = url;
  }

  return (
    <div>
      <p>Connect your Stripe account to start accepting payments.</p>
      <button onClick={handleClick}>Connect with Stripe</button>
    </div>
  );
}
```

```jsx
// packages/dashboard/src/pages/EmbedSnippet.jsx
import { decodeMerchantId } from '../api.js';

export default function EmbedSnippet() {
  const merchantId = decodeMerchantId();
  const serverUrl = import.meta.env.VITE_SERVER_URL || 'http://localhost:3000';
  const publishableKey = import.meta.env.VITE_STRIPE_PUBLISHABLE_KEY || 'pk_test_replace_me';

  const snippet = `<script src="${serverUrl}/widget.js"></script>
<div id="income-checkout"></div>
<script>
  IncomeCheckout.mount('income-checkout', {
    merchantId: ${merchantId},
    amountCents: 1000,
    publishableKey: '${publishableKey}',
    serverUrl: '${serverUrl}',
  });
</script>`;

  return (
    <div>
      <p>Paste this snippet into your site:</p>
      <pre>{snippet}</pre>
    </div>
  );
}
```

```jsx
// packages/dashboard/src/App.jsx
import { Routes, Route, Navigate } from 'react-router-dom';
import Signup from './pages/Signup.jsx';
import Login from './pages/Login.jsx';
import ConnectStripe from './pages/ConnectStripe.jsx';
import EmbedSnippet from './pages/EmbedSnippet.jsx';

export default function App() {
  return (
    <Routes>
      <Route path="/signup" element={<Signup />} />
      <Route path="/login" element={<Login />} />
      <Route path="/connect" element={<ConnectStripe />} />
      <Route path="/embed" element={<EmbedSnippet />} />
      <Route path="*" element={<Navigate to="/signup" replace />} />
    </Routes>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/dashboard/src/pages/ConnectStripe.jsx packages/dashboard/src/pages/EmbedSnippet.jsx packages/dashboard/src/api.js packages/dashboard/src/App.jsx packages/dashboard/test/ConnectStripe.test.jsx
git commit -m "feat(dashboard): add Stripe Connect and embed snippet pages"
```

---

### Task 14: Dashboard - Transactions page

**Files:**
- Create: `packages/dashboard/src/pages/Transactions.jsx`
- Modify: `packages/dashboard/src/api.js` (add `getTransactions()`)
- Modify: `packages/dashboard/src/App.jsx` (register route)
- Test: `packages/dashboard/test/Transactions.test.jsx`

**Interfaces:**
- Produces: `api.js` adds `getTransactions()` → authenticated `GET /api/transactions`, returns `{ transactions }`. `Transactions` page renders a table with amount, fee, status, and date, converting cents to dollars for display.

- [ ] **Step 1: Write the failing test**

```jsx
// packages/dashboard/test/Transactions.test.jsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen, waitFor } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import Transactions from '../src/pages/Transactions.jsx';

vi.mock('../src/api.js', () => ({
  getTransactions: vi.fn().mockResolvedValue({
    transactions: [
      { id: 1, amountCents: 1000, feeCents: 1, status: 'succeeded', createdAt: '2026-01-02 00:00:00' },
    ],
  }),
}));

describe('Transactions page', () => {
  it('renders a row per transaction with dollar amounts', async () => {
    render(
      <MemoryRouter>
        <Transactions />
      </MemoryRouter>
    );

    await waitFor(() => {
      expect(screen.getByText('$10.00')).toBeInTheDocument();
      expect(screen.getByText('$0.01')).toBeInTheDocument();
      expect(screen.getByText('succeeded')).toBeInTheDocument();
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run test/Transactions.test.jsx`
Expected: FAIL — module not found

- [ ] **Step 3: Write minimal implementation**

```javascript
// packages/dashboard/src/api.js (add this function; keep the rest from Task 13)
async function authedGet(path) {
  const token = localStorage.getItem('income_token');
  const res = await fetch(`${SERVER_URL}${path}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  return res.json();
}

export function getTransactions() {
  return authedGet('/api/transactions');
}
```

```jsx
// packages/dashboard/src/pages/Transactions.jsx
import { useEffect, useState } from 'react';
import { getTransactions } from '../api.js';

function centsToDollars(cents) {
  return `$${(cents / 100).toFixed(2)}`;
}

export default function Transactions() {
  const [transactions, setTransactions] = useState([]);

  useEffect(() => {
    getTransactions().then((data) => setTransactions(data.transactions));
  }, []);

  return (
    <table>
      <thead>
        <tr>
          <th>Amount</th>
          <th>Platform fee</th>
          <th>Status</th>
          <th>Date</th>
        </tr>
      </thead>
      <tbody>
        {transactions.map((tx) => (
          <tr key={tx.id}>
            <td>{centsToDollars(tx.amountCents)}</td>
            <td>{centsToDollars(tx.feeCents)}</td>
            <td>{tx.status}</td>
            <td>{tx.createdAt}</td>
          </tr>
        ))}
      </tbody>
    </table>
  );
}
```

```jsx
// packages/dashboard/src/App.jsx
import { Routes, Route, Navigate } from 'react-router-dom';
import Signup from './pages/Signup.jsx';
import Login from './pages/Login.jsx';
import ConnectStripe from './pages/ConnectStripe.jsx';
import EmbedSnippet from './pages/EmbedSnippet.jsx';
import Transactions from './pages/Transactions.jsx';

export default function App() {
  return (
    <Routes>
      <Route path="/signup" element={<Signup />} />
      <Route path="/login" element={<Login />} />
      <Route path="/connect" element={<ConnectStripe />} />
      <Route path="/embed" element={<EmbedSnippet />} />
      <Route path="/transactions" element={<Transactions />} />
      <Route path="*" element={<Navigate to="/signup" replace />} />
    </Routes>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add packages/dashboard/src/pages/Transactions.jsx packages/dashboard/src/api.js packages/dashboard/src/App.jsx packages/dashboard/test/Transactions.test.jsx
git commit -m "feat(dashboard): add transactions list page"
```

---

### Task 15: Manual end-to-end verification

**Files:**
- Create: `test/e2e/test-merchant-site.html`
- Create: `docs/superpowers/plans/2026-09-19-checkout-widget-mvp-e2e-checklist.md`

**Interfaces:**
- Produces: a static HTML page simulating a merchant's website, embedding the built widget from Task 11 against a merchant ID obtained via the real dashboard flow.

- [ ] **Step 1: Create the static test page**

```html
<!-- test/e2e/test-merchant-site.html -->
<!doctype html>
<html>
  <head>
    <title>Test Merchant Site</title>
  </head>
  <body>
    <h1>Example Merchant — Buy a Widget ($10.00)</h1>
    <script src="../../packages/widget/dist/widget.js"></script>
    <div id="income-checkout"></div>
    <script>
      // Replace MERCHANT_ID and PUBLISHABLE_KEY with real values from the dashboard's /embed page.
      IncomeCheckout.mount('income-checkout', {
        merchantId: MERCHANT_ID,
        amountCents: 1000,
        publishableKey: 'PUBLISHABLE_KEY',
        serverUrl: 'http://localhost:3000',
      });
    </script>
  </body>
</html>
```

- [ ] **Step 2: Write the manual verification checklist**

```markdown
<!-- docs/superpowers/plans/2026-09-19-checkout-widget-mvp-e2e-checklist.md -->
# Checkout Widget MVP — Manual E2E Checklist

Prerequisites:
- Stripe test-mode API keys in `.env` (copy from `.env.example`).
- `stripe listen --forward-to localhost:3000/webhooks/stripe` running,
  with its printed `whsec_...` value set as `STRIPE_WEBHOOK_SECRET`.

Steps:
1. [ ] `npm install` at the repo root.
2. [ ] `npm run build --workspace=packages/widget` to produce `packages/widget/dist/widget.js`.
3. [ ] `npm run dev --workspace=packages/server` (starts on port 3000).
4. [ ] `npm run dev --workspace=packages/dashboard` (starts on port 5173).
5. [ ] Open the dashboard, sign up a test merchant.
6. [ ] Click "Connect with Stripe" and complete Stripe's test-mode onboarding
       (use Stripe's test onboarding data, e.g. any test SSN/DOB it accepts).
7. [ ] Confirm the `account.updated` webhook fires (visible in the `stripe listen` log)
       and the merchant's `onboarded` flag is set (check `income.db` or the `/embed` page loading).
8. [ ] On the `/embed` page, copy the merchant ID and publishable key into
       `test/e2e/test-merchant-site.html`.
9. [ ] Open `test/e2e/test-merchant-site.html` directly in a browser.
10. [ ] Complete a test payment with card `4242 4242 4242 4242`, any future
        expiry, any CVC.
11. [ ] Confirm the payment succeeds in the browser (no error shown).
12. [ ] In the Stripe test dashboard, confirm the PaymentIntent shows the
        $10.00 charge split with a $0.01 application fee to the platform account.
13. [ ] On the dashboard's `/transactions` page, confirm the transaction
        appears with the correct amount, fee, and "succeeded" status.
```

- [ ] **Step 3: Commit**

```bash
git add test/e2e/test-merchant-site.html docs/superpowers/plans/2026-09-19-checkout-widget-mvp-e2e-checklist.md
git commit -m "docs: add manual E2E verification checklist for checkout widget MVP"
```

This task has no automated test — its "test" *is* the manual checklist, which is the spec's MVP definition of done.

---

## Self-Review Notes

- **Spec coverage:** Architecture (3 packages) → Tasks 2–14. Data flow (signup → connect → embed → checkout → webhook → transactions) → Tasks 5, 7–10. Error handling (declines, webhook verification, incomplete onboarding) → Tasks 8 (signature verification), 9 (404 on unonboarded merchant); card-decline UI feedback is Stripe.js's built-in behavior via `confirmPayment`'s returned `error`, surfaced to the merchant's customer. Testing (Stripe test mode, Stripe CLI) → Task 15. MVP definition of done → Task 15 checklist maps 1:1 to the spec's five DoD bullets.
- **Placeholder scan:** no TBD/TODO; all steps have runnable code.
- **Type consistency:** `feeCents`/`amountCents` naming and the `{ merchantId, amountCents }` checkout payload shape are consistent across Tasks 9, 11, and 13's embed snippet. `stripe_account_id` / `onboarded` column names match between Task 3's schema and every route that reads them (Tasks 7–10).

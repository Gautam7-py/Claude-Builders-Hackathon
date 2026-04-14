# Node.js + Express Patterns
<!-- Parent skill: backend-skill/SKILL.md -->
<!-- Load when: architecture.stack.backend.framework is "express" -->
<!-- Last updated: 2026-04-11 -->

## Table of contents
- [Middleware mounting order](#middleware-mounting-order)
- [Route file pattern](#route-file-pattern)
- [Controller pattern](#controller-pattern)
- [Prisma integration](#prisma-integration)
- [JWT auth middleware](#jwt-auth-middleware)

---

## Middleware mounting order

Mount middleware in this exact order in `index.js`. Order matters — CORS
and body parsing must come before routes, and the error handler must come last:

```javascript
import express from 'express';
import cors from './middleware/cors.js';
import logger from './middleware/logger.js';
import { errorHandler } from './middleware/errorHandler.js';
import userRoutes from './routes/users.routes.js';

const app = express();

app.use(cors);                        // 1. CORS — must be first
app.use(express.json());              // 2. Body parsing
app.use(logger);                      // 3. Request logging
app.use('/api/users', userRoutes);    // 4. Routes (one line per resource)
app.use(errorHandler);                // 5. Error handler — must be last

export { app };                       // Always export app for testability

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

---

## Route file pattern

One route file per entity. Routes delegate to controllers — no logic here:

```javascript
// src/routes/users.routes.js
import { Router } from 'express';
import * as usersController from '../controllers/users.controller.js';
import { requireAuth } from '../middleware/auth.js'; // only if auth required

const router = Router();

router.get('/',       usersController.getAll);
router.get('/:id',    usersController.getById);
router.post('/',      usersController.create);
router.put('/:id',    requireAuth, usersController.update);  // auth on protected routes
router.delete('/:id', requireAuth, usersController.remove);

export default router;
```

---

## Controller pattern

One controller file per entity. All business logic lives here:

```javascript
// src/controllers/users.controller.js
import { prisma } from '../db/connection.js';

export const getAll = async (req, res, next) => {
  try {
    const users = await prisma.user.findMany();
    res.json({ success: true, data: users });
  } catch (err) {
    next(err);
  }
};

export const getById = async (req, res, next) => {
  try {
    const user = await prisma.user.findUnique({ where: { id: req.params.id } });
    if (!user) return res.status(404).json({ success: false, error: 'User not found' });
    res.json({ success: true, data: user });
  } catch (err) {
    next(err);
  }
};

export const create = async (req, res, next) => {
  try {
    const { email, name } = req.body;
    if (!email) return res.status(400).json({ success: false, error: 'email is required' });
    const user = await prisma.user.create({ data: { email, name } });
    res.status(201).json({ success: true, data: user });
  } catch (err) {
    next(err);
  }
};
```

---

## Prisma integration

### Connection singleton (`src/db/connection.js`)
```javascript
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis;
export const prisma = globalForPrisma.prisma ?? new PrismaClient();
if (process.env.NODE_ENV !== 'production') globalForPrisma.prisma = prisma;
```

### Schema template (`prisma/schema.prisma`)
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  posts     Post[]
}

model Post {
  id        String   @id @default(uuid())
  title     String
  content   String?
  authorId  String
  author    User     @relation(fields: [authorId], references: [id])
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

---

## JWT auth middleware

```javascript
// src/middleware/auth.js
import jwt from 'jsonwebtoken';

export const requireAuth = (req, res, next) => {
  const authHeader = req.headers.authorization;
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json({ success: false, error: 'No token provided' });
  }
  try {
    const token = authHeader.split(' ')[1];
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    res.status(401).json({ success: false, error: 'Invalid or expired token' });
  }
};
```

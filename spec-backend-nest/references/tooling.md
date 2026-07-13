# Tooling, Configuração e CI/CD

## tsconfig.json

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2021",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": ".",
    "incremental": true,
    "skipLibCheck": true,
    "strictNullChecks": true,
    "noImplicitAny": true,
    "strictBindCallApply": true,
    "forceConsistentCasingInFileNames": true,
    "noFallthroughCasesInSwitch": true,
    "strict": true,
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

---

## ESLint — eslint.config.mjs

```javascript
import tseslint from '@typescript-eslint/eslint-plugin';
import tsparser from '@typescript-eslint/parser';

export default [
  {
    files: ['src/**/*.ts'],
    plugins: { '@typescript-eslint': tseslint },
    languageOptions: {
      parser: tsparser,
      parserOptions: { project: './tsconfig.json' },
    },
    rules: {
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-unused-vars': [
        'error',
        { argsIgnorePattern: '^_' },
      ],
      '@typescript-eslint/explicit-function-return-type': 'warn',
      '@typescript-eslint/consistent-type-imports': 'error',
      '@typescript-eslint/no-floating-promises': 'error',
      'no-console': 'error', // usar Logger do NestJS
      'prefer-const': 'error',
      'no-var': 'error',
    },
  },
  {
    // Relaxar em testes
    files: ['**/*.spec.ts', 'test/**/*.ts'],
    rules: {
      '@typescript-eslint/no-explicit-any': 'off',
      '@typescript-eslint/explicit-function-return-type': 'off',
    },
  },
];
```

---

## Validação de Env Vars

```typescript
// config/env.validation.ts
import * as Joi from 'joi';

export const envValidationSchema = Joi.object({
  NODE_ENV: Joi.string()
    .valid('development', 'test', 'production')
    .default('development'),
  PORT: Joi.number().default(3000),

  // Database
  DATABASE_URL: Joi.string().uri().required(),

  // JWT
  JWT_SECRET: Joi.string().min(32).required(),
  JWT_EXPIRES_IN: Joi.string().default('15m'),
  JWT_REFRESH_SECRET: Joi.string().min(32).required(),
  JWT_REFRESH_EXPIRES_IN: Joi.string().default('7d'),

  // Redis
  REDIS_URL: Joi.string().uri().required(),

  // Logs
  LOG_LEVEL: Joi.string()
    .valid('fatal', 'error', 'warn', 'info', 'debug', 'trace')
    .default('info'),

  // Externo
  STRIPE_SECRET_KEY: Joi.string().when('NODE_ENV', {
    is: 'production',
    then: Joi.required(),
    otherwise: Joi.optional(),
  }),

  // CORS
  ALLOWED_ORIGINS: Joi.string().default('http://localhost:3000'),
  API_DOCS_URL: Joi.string().uri().default('https://api.example.com/errors'),
});

// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      validationSchema: envValidationSchema,
      validationOptions: { abortEarly: false },
    }),
  ],
})
export class AppModule {}
```

---

## Dockerfile — Multi-stage

```dockerfile
# Dockerfile
FROM node:20-alpine AS base
WORKDIR /app
COPY package*.json ./

# ── Dependências de build ─────────────────────────────────────────────────────
FROM base AS deps
RUN npm ci

# ── Build ─────────────────────────────────────────────────────────────────────
FROM base AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# ── Produção — sem devDependencies ────────────────────────────────────────────
FROM base AS prod-deps
RUN npm ci --omit=dev

# ── Runtime final ─────────────────────────────────────────────────────────────
FROM node:20-alpine AS runner
WORKDIR /app

# Criar utilizador não-root
RUN addgroup --system --gid 1001 nodejs \
 && adduser  --system --uid 1001 nestjs

COPY --from=builder   --chown=nestjs:nodejs /app/dist        ./dist
COPY --from=prod-deps --chown=nestjs:nodejs /app/node_modules ./node_modules
COPY --chown=nestjs:nodejs package.json ./

USER nestjs

EXPOSE 3000
ENV NODE_ENV=production

CMD ["node", "dist/main.js"]
```

```yaml
# docker-compose.yml (desenvolvimento)
services:
  api:
    build: { context: ., target: builder }
    command: npm run start:dev
    volumes:
      - .:/app
      - /app/node_modules
    ports: ['3000:3000']
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/myapp
      REDIS_URL: redis://redis:6379
    depends_on: [db, redis]
    env_file: .env.local

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: myapp
    ports: ['5432:5432']
    volumes: [postgres_data:/var/lib/postgresql/data]

  redis:
    image: redis:7-alpine
    ports: ['6379:6379']

volumes:
  postgres_data:
```

---

## package.json — Scripts Obrigatórios

```json
{
  "scripts": {
    "build": "nest build",
    "start": "node dist/main",
    "start:dev": "nest start --watch",
    "start:debug": "nest start --debug --watch",
    "start:prod": "node dist/main",

    "lint": "eslint src --ext .ts",
    "lint:fix": "eslint src --ext .ts --fix",
    "format": "prettier --write \"src/**/*.ts\" \"test/**/*.ts\"",
    "format:check": "prettier --check \"src/**/*.ts\"",
    "typecheck": "tsc --noEmit",

    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:e2e": "jest --config jest-e2e.config.ts --forceExit",

    "db:migrate": "typeorm migration:run -d dist/shared/infrastructure/database/typeorm.config.js",
    "db:migrate:revert": "typeorm migration:revert -d dist/shared/infrastructure/database/typeorm.config.js",
    "db:generate": "typeorm migration:generate -d dist/shared/infrastructure/database/typeorm.config.js",
    "db:seed": "ts-node prisma/seed.ts",

    "check": "npm run typecheck && npm run lint && npm run format:check && npm run test:coverage"
  }
}
```

---

## CI/CD — GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main, develop]

jobs:
  quality:
    name: Quality Gates
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm" }

      - name: Install
        run: npm ci

      - name: Type check
        run: npm run typecheck

      - name: Lint
        run: npm run lint

      - name: Format check
        run: npm run format:check

      - name: Unit tests + coverage
        run: npm run test:coverage
        env:
          NODE_ENV: test

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true

  security:
    name: Security
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm" }
      - run: npm ci
      - name: Dependency audit
        run: npm audit --audit-level=high
      - name: Secret scan
        uses: trufflesecurity/trufflehog@main
        with: { path: ./, base: ${{ github.event.repository.default_branch }} }

  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [quality]
    steps:
      - uses: actions/checkout@v4
      - name: Build image
        run: docker build --target runner -t myapp:${{ github.sha }} .

  integration:
    name: Integration + E2E Tests
    runs-on: ubuntu-latest
    needs: [quality]
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        ports: ["5432:5432"]
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm" }
      - run: npm ci
      - run: npm run build
      - name: Run migrations
        run: npm run db:migrate
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
      - name: E2E tests
        run: npm run test:e2e
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test-secret-min-32-chars-long-enough
          JWT_REFRESH_SECRET: test-refresh-secret-min-32-chars-long
          NODE_ENV: test
```

---

## .env.example

```bash
NODE_ENV=development
PORT=3000

# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/myapp

# JWT
JWT_SECRET=gerar-com-openssl-rand-base64-32-minimo
JWT_EXPIRES_IN=15m
JWT_REFRESH_SECRET=gerar-outro-segredo-diferente-do-anterior
JWT_REFRESH_EXPIRES_IN=7d

# Redis (BullMQ + cache)
REDIS_URL=redis://localhost:6379

# Logs
LOG_LEVEL=info

# CORS
ALLOWED_ORIGINS=http://localhost:3000,http://localhost:3001

# API Docs (para links em error responses)
API_DOCS_URL=https://api.bpc.ao/errors

# Externo
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=

# Email
RESEND_API_KEY=
EMAIL_FROM=noreply@bpc.ao
```

---

## nest-cli.json

```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "deleteOutDir": true,
    "plugins": [
      {
        "name": "@nestjs/swagger",
        "options": {
          "introspectComments": true,
          "classValidatorShim": true
        }
      }
    ]
  }
}
```

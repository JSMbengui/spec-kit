# Tooling, Configuração e CI/CD

## tsconfig.json — Configuração Estrita

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": false,
    "forceConsistentCasingInFileNames": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [{ "name": "next" }],
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

---

## ESLint — eslint.config.mjs

```javascript
// eslint.config.mjs
import { dirname } from "path";
import { fileURLToPath } from "url";
import { FlatCompat } from "@eslint/eslintrc";

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);
const compat = new FlatCompat({ baseDirectory: __dirname });

export default [
  ...compat.extends("next/core-web-vitals", "next/typescript"),
  {
    rules: {
      // TypeScript
      "@typescript-eslint/no-unused-vars": ["error", { argsIgnorePattern: "^_" }],
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/consistent-type-imports": "error",   // import type {}

      // React
      "react/self-closing-comp": "error",
      "react/jsx-no-useless-fragment": "error",

      // Next.js best practices
      "@next/next/no-img-element": "error",

      // Imports
      "no-console": ["warn", { allow: ["warn", "error"] }],
      "prefer-const": "error",
    },
  },
  {
    // Relaxar regras em ficheiros de teste
    files: ["**/*.test.ts", "**/*.test.tsx", "tests/**/*"],
    rules: {
      "@typescript-eslint/no-explicit-any": "off",
    },
  },
];
```

---

## Prettier — .prettierrc

```json
{
  "semi": true,
  "singleQuote": false,
  "quoteProps": "as-needed",
  "jsxSingleQuote": false,
  "trailingComma": "all",
  "bracketSpacing": true,
  "bracketSameLine": false,
  "arrowParens": "always",
  "printWidth": 100,
  "tabWidth": 2,
  "useTabs": false,
  "endOfLine": "lf",
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

```
# .prettierignore
.next/
node_modules/
prisma/migrations/
public/
```

---

## Husky + lint-staged — Pre-commit

```bash
# Instalação
npm install --save-dev husky lint-staged
npx husky init
```

```json
// package.json
{
  "scripts": {
    "prepare": "husky"
  },
  "lint-staged": {
    "*.{ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,css}": [
      "prettier --write"
    ]
  }
}
```

```bash
# .husky/pre-commit
#!/bin/sh
npx lint-staged

# .husky/commit-msg (Conventional Commits)
#!/bin/sh
npx --no -- commitlint --edit "$1"
```

```javascript
// commitlint.config.js
export default {
  extends: ["@commitlint/config-conventional"],
  rules: {
    "type-enum": [2, "always", [
      "feat", "fix", "docs", "style", "refactor",
      "test", "chore", "perf", "ci", "revert"
    ]],
    "subject-max-length": [2, "always", 72],
  },
};
```

---

## next.config.ts

```typescript
import type { NextConfig } from "next";

const config: NextConfig = {
  // Habilitar App Router features
  experimental: {
    // ppr: true,          // Partial Prerendering (experimental)
    // reactCompiler: true, // React Compiler (experimental)
  },

  // Imagens remotas permitidas
  images: {
    remotePatterns: [
      { protocol: "https", hostname: "cdn.bpc.ao" },
      { protocol: "https", hostname: "avatars.githubusercontent.com" },
    ],
    formats: ["image/avif", "image/webp"],
  },

  // Headers de segurança
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: [
          { key: "X-Frame-Options", value: "DENY" },
          { key: "X-Content-Type-Options", value: "nosniff" },
          { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
          {
            key: "Content-Security-Policy",
            value: [
              "default-src 'self'",
              "script-src 'self' 'unsafe-eval' 'unsafe-inline'",
              "style-src 'self' 'unsafe-inline'",
              "img-src 'self' blob: data: https://cdn.bpc.ao",
            ].join("; "),
          },
        ],
      },
    ];
  },

  // Redirects
  async redirects() {
    return [
      { source: "/home", destination: "/", permanent: true },
    ];
  },
};

export default config;
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
    name: Type Check + Lint + Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npx tsc --noEmit

      - name: Lint
        run: npm run lint

      - name: Format check
        run: npx prettier --check "src/**/*.{ts,tsx}"

      - name: Unit + Integration tests
        run: npm run test:coverage
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Dependency audit
        run: npm audit --audit-level=high
      - name: CodeQL Analysis
        uses: github/codeql-action/init@v3
        with:
          languages: typescript

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [quality]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm" }
      - run: npm ci
      - name: Build
        run: npm run build
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          NEXTAUTH_SECRET: ${{ secrets.NEXTAUTH_SECRET }}
          NEXTAUTH_URL: ${{ secrets.NEXTAUTH_URL }}

  e2e:
    name: E2E Tests (apenas em main)
    runs-on: ubuntu-latest
    needs: [build]
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm" }
      - run: npm ci
      - name: Install Playwright
        run: npx playwright install --with-deps chromium
      - name: Run E2E
        run: npx playwright test
        env:
          BASE_URL: ${{ secrets.STAGING_URL }}
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
```

---

## package.json — Scripts Obrigatórios

```json
{
  "scripts": {
    "dev": "next dev --turbopack",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "lint:fix": "next lint --fix",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "typecheck": "tsc --noEmit",
    "test": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "db:generate": "prisma generate",
    "db:migrate": "prisma migrate dev",
    "db:migrate:prod": "prisma migrate deploy",
    "db:seed": "tsx prisma/seed.ts",
    "db:studio": "prisma studio",
    "prepare": "husky",
    "check": "npm run typecheck && npm run lint && npm run format:check && npm run test:coverage"
  }
}
```

---

## .env.example — Template de Variáveis

```bash
# Base de dados
DATABASE_URL="postgresql://user:password@localhost:5432/myapp"

# Auth.js
NEXTAUTH_SECRET="gere-com-openssl-rand-base64-32"
NEXTAUTH_URL="http://localhost:3000"

# App
NEXT_PUBLIC_APP_URL="http://localhost:3000"
NEXT_PUBLIC_APP_NAME="BPC Portal"

# Serviços externos (preencher conforme necessário)
STRIPE_SECRET_KEY=""
STRIPE_PUBLISHABLE_KEY=""
STRIPE_WEBHOOK_SECRET=""

# Email (ex: Resend)
RESEND_API_KEY=""
EMAIL_FROM="noreply@bpc.ao"

# Storage (ex: AWS S3)
AWS_ACCESS_KEY_ID=""
AWS_SECRET_ACCESS_KEY=""
AWS_REGION="eu-west-1"
AWS_S3_BUCKET=""
```

```bash
# .gitignore (adicionar às entradas padrão)
.env
.env.local
.env.production
.env.staging
# Manter apenas .env.example no git
```
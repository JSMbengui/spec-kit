# Estrutura de Pastas e Naming Conventions

## Estrutura Completa

```
my-app/
├── src/
│   ├── app/                          # Next.js App Router — APENAS routing
│   │   ├── (marketing)/              # Route group público
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx              # /
│   │   │   ├── about/
│   │   │   │   └── page.tsx          # /about
│   │   │   └── pricing/
│   │   │       └── page.tsx          # /pricing
│   │   ├── (dashboard)/              # Route group protegido
│   │   │   ├── layout.tsx            # Layout com Sidebar + auth check
│   │   │   ├── dashboard/
│   │   │   │   ├── page.tsx          # /dashboard
│   │   │   │   ├── loading.tsx       # Skeleton automático
│   │   │   │   └── error.tsx         # Error boundary automático
│   │   │   ├── payments/
│   │   │   │   ├── page.tsx          # /payments (lista)
│   │   │   │   ├── [id]/
│   │   │   │   │   ├── page.tsx      # /payments/[id] (detalhe)
│   │   │   │   │   └── edit/
│   │   │   │   │       └── page.tsx  # /payments/[id]/edit
│   │   │   │   └── new/
│   │   │   │       └── page.tsx      # /payments/new
│   │   │   └── settings/
│   │   │       └── page.tsx
│   │   ├── api/                      # Route Handlers (API pública/webhooks)
│   │   │   ├── auth/
│   │   │   │   └── [...nextauth]/
│   │   │   │       └── route.ts
│   │   │   ├── webhooks/
│   │   │   │   └── stripe/
│   │   │   │       └── route.ts
│   │   │   └── health/
│   │   │       └── route.ts
│   │   ├── layout.tsx                # Root layout (html, body, providers)
│   │   ├── not-found.tsx             # Página 404 global
│   │   ├── error.tsx                 # Error boundary global ("use client")
│   │   └── globals.css
│   │
│   ├── features/                     # ← NÚCLEO — domínios de negócio
│   │   ├── auth/
│   │   │   ├── components/
│   │   │   │   ├── LoginForm.tsx
│   │   │   │   ├── RegisterForm.tsx
│   │   │   │   └── UserMenu.tsx
│   │   │   ├── hooks/
│   │   │   │   └── useCurrentUser.ts
│   │   │   ├── actions/
│   │   │   │   ├── login.ts          # "use server"
│   │   │   │   └── logout.ts         # "use server"
│   │   │   ├── queries/
│   │   │   │   └── get-user.ts       # Funções server-side de fetch
│   │   │   ├── schemas/
│   │   │   │   └── auth.schema.ts    # Zod schemas
│   │   │   └── types.ts
│   │   │
│   │   ├── payments/
│   │   │   ├── components/
│   │   │   │   ├── PaymentList.tsx
│   │   │   │   ├── PaymentCard.tsx
│   │   │   │   ├── PaymentForm.tsx   # "use client"
│   │   │   │   └── PaymentStatus.tsx
│   │   │   ├── hooks/
│   │   │   │   ├── usePaymentFilter.ts
│   │   │   │   └── usePaymentStatus.ts
│   │   │   ├── actions/
│   │   │   │   ├── create-payment.ts
│   │   │   │   ├── update-payment.ts
│   │   │   │   └── cancel-payment.ts
│   │   │   ├── queries/
│   │   │   │   ├── get-payments.ts
│   │   │   │   └── get-payment-by-id.ts
│   │   │   ├── schemas/
│   │   │   │   └── payment.schema.ts
│   │   │   └── types.ts
│   │   │
│   │   └── dashboard/
│   │       ├── components/
│   │       │   ├── StatsCard.tsx
│   │       │   └── RecentActivity.tsx
│   │       └── queries/
│   │           └── get-dashboard-stats.ts
│   │
│   ├── components/                   # Componentes verdadeiramente partilhados
│   │   ├── ui/                       # shadcn/ui + primitivos
│   │   │   ├── button.tsx
│   │   │   ├── input.tsx
│   │   │   ├── dialog.tsx
│   │   │   ├── table.tsx
│   │   │   └── ... (gerado pelo shadcn CLI)
│   │   ├── layout/
│   │   │   ├── Navbar.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   ├── Footer.tsx
│   │   │   └── PageHeader.tsx
│   │   └── providers/
│   │       ├── QueryProvider.tsx     # React Query
│   │       ├── ThemeProvider.tsx
│   │       └── index.tsx             # Compõe todos os providers
│   │
│   ├── lib/
│   │   ├── db/
│   │   │   └── index.ts              # Prisma singleton
│   │   ├── auth/
│   │   │   └── index.ts              # Auth.js config
│   │   ├── validations/
│   │   │   └── common.ts             # Schemas Zod partilhados (email, uuid, etc.)
│   │   └── utils.ts                  # cn(), formatDate(), formatCurrency()
│   │
│   ├── hooks/                        # Hooks globais
│   │   ├── useDebounce.ts
│   │   ├── useLocalStorage.ts
│   │   └── useMediaQuery.ts
│   │
│   ├── types/
│   │   ├── index.ts                  # Re-exports de tipos globais
│   │   └── api.ts                    # Tipos de resposta de API externa
│   │
│   ├── constants/
│   │   ├── routes.ts                 # ROUTES.dashboard, ROUTES.payments, etc.
│   │   └── config.ts                 # APP_NAME, ITEMS_PER_PAGE, etc.
│   │
│   └── styles/
│       └── globals.css
│
├── prisma/
│   ├── schema.prisma
│   └── migrations/
│
├── public/
│   ├── images/
│   └── icons/
│
├── tests/
│   ├── e2e/                          # Playwright
│   └── fixtures/                     # Dados de teste partilhados
│
├── .env.local                        # Variáveis locais (no .gitignore)
├── .env.example                      # Template de env (no git)
├── next.config.ts
├── tsconfig.json
├── tailwind.config.ts
├── eslint.config.mjs
└── package.json
```

---

## Naming Conventions

### Ficheiros e Pastas
```
Pastas de features:          kebab-case          payments, user-accounts
Ficheiros de componentes:    PascalCase.tsx       PaymentCard.tsx
Ficheiros de hooks:          camelCase.ts         usePaymentFilter.ts
Ficheiros de actions:        kebab-case.ts        create-payment.ts
Ficheiros de queries:        kebab-case.ts        get-payments.ts
Ficheiros de schemas:        kebab-case.schema.ts payment.schema.ts
Ficheiros de tipos:          camelCase.ts         types.ts
Utilitários:                 camelCase.ts         utils.ts
Constantes:                  camelCase.ts         routes.ts
```

### Componentes
```typescript
// ✅ PascalCase, nome descritivo com contexto
export function PaymentCard({ payment }: PaymentCardProps) {}
export function DashboardStatsCard({ stats }: StatsCardProps) {}
export function UserAccountMenu({ user }: UserMenuProps) {}

// ❌ Genérico, sem contexto de domínio
export function Card() {}
export function Menu() {}
```

### Props Interfaces
```typescript
// ✅ NomeDoComponenteProps
interface PaymentCardProps {
  payment: Payment;
  onSelect?: (id: string) => void;
  isSelected?: boolean;
  className?: string;       // sempre aceitar className para flexibilidade
}

// ❌ Props genérico
interface Props {}
```

### Server Actions
```typescript
// ✅ verbo + substantivo, descreve a acção
export async function createPayment(formData: FormData) {}
export async function updateUserProfile(id: string, data: UpdateProfileDto) {}
export async function cancelSubscription(subscriptionId: string) {}

// ❌ nomes vagos
export async function handleForm() {}
export async function doStuff() {}
```

### Queries (server-side fetch)
```typescript
// ✅ get + o que retorna
export async function getPaymentById(id: string): Promise<Payment | null> {}
export async function getPaymentsByUser(userId: string): Promise<Payment[]> {}
export async function getDashboardStats(): Promise<DashboardStats> {}
```

### Constantes de Rotas
```typescript
// src/constants/routes.ts
export const ROUTES = {
  home: "/",
  dashboard: "/dashboard",
  payments: {
    list: "/payments",
    new: "/payments/new",
    detail: (id: string) => `/payments/${id}`,
    edit: (id: string) => `/payments/${id}/edit`,
  },
  settings: "/settings",
} as const;

// Uso:
import { ROUTES } from "@/constants/routes";
<Link href={ROUTES.payments.detail(payment.id)}>Ver pagamento</Link>
```

---

## Barrel Exports — Quando usar e quando evitar

### Usar barrel exports apenas em `features/*/index.ts`
```typescript
// features/payments/index.ts — expõe apenas a API pública da feature
export { PaymentList } from "./components/PaymentList";
export { PaymentCard } from "./components/PaymentCard";
export { createPayment, updatePayment } from "./actions/create-payment";
export type { Payment, PaymentStatus } from "./types";

// Não expõe: queries internas, helpers privados, sub-componentes internos
```

### Evitar barrel exports em `components/ui/`
```typescript
// ❌ Evitar — barrel em UI causa problemas de tree-shaking e bundle
// components/ui/index.ts
export * from "./button";
export * from "./input";
// ... 50 componentes

// ✅ Importar directamente
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
```

---

## Path Aliases — Configuração Obrigatória

```json
// tsconfig.json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

```typescript
// ✅ Imports absolutos — nunca imports relativos além de 1 nível
import { PaymentCard } from "@/features/payments/components/PaymentCard";
import { Button } from "@/components/ui/button";
import { db } from "@/lib/db";
import { ROUTES } from "@/constants/routes";

// ❌ Imports relativos longos
import { PaymentCard } from "../../../../features/payments/components/PaymentCard";
```
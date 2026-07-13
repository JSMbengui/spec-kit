---
name: nextjs-standards
description: >
  Guia de padrões de organização e arquitectura para projectos Next.js (App Router, TypeScript).
  Usar SEMPRE que o utilizador pedir para: organizar, estruturar, refactorizar ou rever um projecto
  Next.js; criar novos ficheiros, componentes, páginas, hooks, actions ou serviços em Next.js;
  decidir onde colocar código (componente Server vs Client, action vs route handler, etc.);
  aplicar Clean Architecture, Feature-First, SOLID ou boas práticas de grandes empresas em Next.js;
  rever ou melhorar código Next.js existente; configurar ESLint, Prettier, Tailwind ou CI/CD em
  projectos Next.js; ou sempre que o utilizador mencionar "App Router", "server components",
  "server actions", "next.js", "nextjs" ou perguntar sobre estrutura de pastas React/Next.
---

# Next.js Standards Skill

Este skill define os padrões exactos a seguir ao organizar, criar ou rever qualquer projecto Next.js.
Aplica estes padrões de forma consistente e proactiva — não esperes que o utilizador os peça
explicitamente.

## Como usar este skill

1. Lê este SKILL.md completo antes de qualquer resposta sobre estrutura Next.js
2. Para detalhes de uma área específica, lê o ficheiro de referência correspondente:
   - Estrutura de pastas e naming → `references/structure.md`
   - Componentes (Server vs Client, padrões) → `references/components.md`
   - Data fetching, Server Actions, Route Handlers → `references/data.md`
   - Estado, formulários, validação → `references/state-forms.md`
   - Performance, SEO, optimização → `references/performance.md`
   - Testes → `references/testing.md`
   - CI/CD, configuração, tooling → `references/tooling.md`
3. Aplica sempre **todos** os padrões relevantes, mesmo que o utilizador só peça "cria um componente"

---

## Princípios Fundamentais (NUNCA violar)

### 1. Server Components por defeito
Todo o componente é Server Component a menos que precise explicitamente de:
- `useState` / `useReducer` / `useEffect`
- Event listeners (`onClick`, `onChange`, etc.)
- Browser APIs (`localStorage`, `window`, `navigator`)
- Hooks de terceiros que exijam contexto de cliente

**Se precisar de interactividade, isola o mínimo possível em `"use client"`.**

### 2. Feature-First, não Layer-First
```
✅  src/features/payments/components/PaymentList.tsx
❌  src/components/PaymentList.tsx  (genérico, sem contexto de domínio)
```
Cada feature é um módulo auto-contido. Código partilhado vai para `src/components/ui/` ou `src/lib/`.

### 3. Tipagem estrita — sem `any`
- `tsconfig.json` com `"strict": true` sempre
- Nunca usar `as any` — usar `unknown` + type guard se necessário
- Todos os props, retornos de função e dados de API tipados explicitamente

### 4. Validação na fronteira
- Inputs de formulários → Zod schema **antes** de chegar ao servidor
- Variáveis de ambiente → validar no boot com Zod + `@t3-oss/env-nextjs`
- Dados externos de API → parse com Zod, nunca confiar cegamente

### 5. Erros explícitos, nunca silenciosos
- Server Actions devolvem `{ data, error }` — nunca lançam excepções para o cliente
- `error.tsx` em cada segmento de rota crítico
- Logging de erros inesperados sempre (nunca `catch {}` vazio)

---

## Decisões Rápidas (cheat-sheet)

| Situação | Solução |
|----------|---------|
| Buscar dados para renderizar | Server Component + `async/await` directo |
| Mutação (criar, editar, apagar) | Server Action |
| API pública ou webhook externo | Route Handler (`app/api/`) |
| Estado local de UI | `useState` em Client Component |
| Estado global do utilizador | Zustand store |
| Formulário simples | Server Action + `useActionState` |
| Formulário complexo com validação reactiva | React Hook Form + Zod + Server Action |
| Dados que mudam frequentemente no cliente | React Query (`useQuery`) |
| SEO dinâmico | `generateMetadata()` na page.tsx |
| Página estática (não muda) | `export const dynamic = 'force-static'` |
| Página dinâmica por utilizador | `export const dynamic = 'force-dynamic'` |
| Autenticação | Auth.js (NextAuth v5) |
| Upload de ficheiros | Route Handler + `formidable` / S3 presigned URL |

---

## Estrutura de Pastas — Resumo

```
src/
  app/                    # Apenas routing e layouts
    (marketing)/          # Route group — sem prefixo na URL
    (dashboard)/          # Route group — protegido por auth
      layout.tsx          # Layout do dashboard
      page.tsx            # /dashboard
    api/                  # Route handlers
    layout.tsx            # Root layout (html, body, providers)
    page.tsx              # / (home)

  features/               # ← NÚCLEO DO PROJECTO
    [feature-name]/
      components/         # Client components desta feature
      hooks/              # Hooks cliente específicos
      actions/            # Server Actions desta feature
      queries/            # Funções de fetch (server-side)
      schemas/            # Zod schemas
      types.ts            # Types da feature
      index.ts            # Barrel export (opcional)

  components/
    ui/                   # shadcn/ui + primitivos (Button, Input, etc.)
    layout/               # Navbar, Sidebar, Footer, PageHeader
    providers/            # Context providers (QueryProvider, ThemeProvider)

  lib/
    db/                   # Prisma client singleton
    auth/                 # Auth.js config e helpers
    validations/          # Zod schemas partilhados
    utils.ts              # cn(), formatDate(), etc.

  hooks/                  # Hooks globais reutilizáveis
  types/                  # Types e interfaces globais
  constants/              # Constantes da aplicação
  styles/                 # globals.css, variáveis CSS
```

> Para detalhes completos de cada pasta → lê `references/structure.md`

---

## Padrão de Cada Tipo de Ficheiro

### page.tsx (Server Component)
```typescript
import type { Metadata } from "next";

// SEO sempre presente
export const metadata: Metadata = {
  title: "Título | App",
  description: "Descrição",
};

// Props tipadas (searchParams é sempre Promise em Next 15)
interface Props {
  params: Promise<{ id: string }>;
  searchParams: Promise<{ filter?: string }>;
}

export default async function Page({ params, searchParams }: Props) {
  const { id } = await params;
  const { filter } = await searchParams;
  const data = await myFeatureQuery(id, filter);
  return <MyFeatureView data={data} />;
}
```

### Server Action
```typescript
"use server";
import { z } from "zod";
import { revalidatePath } from "next/cache";

const Schema = z.object({ name: z.string().min(2) });

export async function createItem(formData: FormData) {
  const parsed = Schema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors };
  }
  try {
    const item = await db.item.create({ data: parsed.data });
    revalidatePath("/items");
    return { data: item };
  } catch {
    return { error: { general: ["Erro ao criar item. Tenta novamente."] } };
  }
}
```

### Client Component
```typescript
"use client";
// Só presente quando NECESSÁRIO (interactividade, hooks, browser APIs)

interface Props { initialData: Item[] }

export function ItemList({ initialData }: Props) {
  const [filter, setFilter] = useState("");
  const filtered = useMemo(
    () => initialData.filter(i => i.name.includes(filter)),
    [initialData, filter]
  );
  return (
    <>
      <input value={filter} onChange={e => setFilter(e.target.value)} />
      {filtered.map(item => <ItemCard key={item.id} item={item} />)}
    </>
  );
}
```

> Para todos os padrões de componentes → lê `references/components.md`
> Para data fetching e actions → lê `references/data.md`

---

## O que NUNCA fazer em Next.js

```typescript
// ❌ fetch() em useEffect (usa Server Component ou React Query)
useEffect(() => { fetch("/api/data").then(setData) }, []);

// ❌ <img> (usa next/image)
<img src="/photo.jpg" />

// ❌ <a href> interno (usa next/link)
<a href="/about">Sobre</a>

// ❌ "use client" no layout.tsx ou page.tsx raiz
"use client"; export default function Layout(...) // ERRADO

// ❌ any
const data: any = await fetch(...)

// ❌ Variáveis de ambiente sem validação
process.env.DATABASE_URL // pode ser undefined em runtime

// ❌ console.log em produção (usa logger estruturado)
console.log("user:", user)

// ❌ Lógica de negócio no controller (page.tsx)
// A page.tsx chama queries/actions, nunca contém lógica directamente

// ❌ Mutation em GET (Route Handler ou Server Component)
// Mutações sempre via POST/PATCH/DELETE
```

---

## Referências disponíveis

Lê o ficheiro relevante quando precisares de detalhes:

| Ficheiro | Conteúdo |
|----------|----------|
| `references/structure.md` | Estrutura completa de pastas, naming conventions, barrel exports |
| `references/components.md` | Server vs Client, composição, padrões de UI, acessibilidade |
| `references/data.md` | Fetch, cache, Server Actions, Route Handlers, React Query |
| `references/state-forms.md` | Zustand, React Hook Form, Zod, useActionState |
| `references/performance.md` | Streaming, Suspense, Image, Font, bundle, Core Web Vitals |
| `references/testing.md` | Vitest, Testing Library, Playwright, MSW, cobertura |
| `references/tooling.md` | ESLint, Prettier, Husky, tsconfig, env validation, CI/CD |
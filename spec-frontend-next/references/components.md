# Componentes — Server vs Client, Padrões e Composição

## A Decisão mais Importante: Server vs Client

### Árvore de decisão
```
Preciso de useState / useReducer?          → "use client"
Preciso de useEffect?                      → "use client"
Preciso de event listeners?                → "use client"
Preciso de browser APIs?                   → "use client"
Preciso de hooks de libs terceiras?        → provavelmente "use client"
Preciso de submeter dados (mutação)?       → Server Action ("use server") — ver secção dedicada

Só renderizo dados?                        → Server Component ✅
Só acedo a DB/APIs privadas?               → Server Component ✅
Só componho outros componentes?            → Server Component ✅
```

### Padrão de composição — isola o mínimo em "use client"

```typescript
// ✅ Server Component envolve Client Component
// app/(dashboard)/payments/page.tsx — Server Component
import { getPayments } from "@/features/payments/queries/get-payments";
import { PaymentListClient } from "@/features/payments/components/PaymentListClient";

export default async function PaymentsPage() {
  const payments = await getPayments(); // acesso directo a DB
  return (
    <div>
      <h1>Pagamentos</h1>
      {/* Passa dados já fetchados para o client component */}
      <PaymentListClient initialPayments={payments} />
    </div>
  );
}

// features/payments/components/PaymentListClient.tsx — Client Component
"use client";
import { useState, useMemo } from "react";

interface Props { initialPayments: Payment[] }

export function PaymentListClient({ initialPayments }: Props) {
  const [filter, setFilter] = useState("");
  // interactividade aqui
}
```

---

## Server Actions — Mutações sem Route Handler

Para forms e mutações simples, **Server Actions são hoje o padrão por defeito** no App Router — preferíveis a um Route Handler dedicado, exceto quando o endpoint precisa de ser consumido por algo externo (webhook, app mobile, API pública).

```typescript
// features/payments/actions/create-payment.ts
"use server";

import { revalidatePath } from "next/cache";
import { paymentSchema } from "@/features/payments/types";

export async function createPayment(formData: FormData) {
  const parsed = paymentSchema.safeParse({
    amount: formData.get("amount"),
    description: formData.get("description"),
  });

  if (!parsed.success) {
    return { success: false, errors: parsed.error.flatten().fieldErrors };
  }

  await db.payment.create({ data: parsed.data });
  revalidatePath("/payments");
  return { success: true };
}
```

```typescript
// Uso num Client Component com useActionState (React 19 / Next 15+)
"use client";
import { useActionState } from "react";
import { createPayment } from "@/features/payments/actions/create-payment";

export function PaymentForm() {
  const [state, formAction, isPending] = useActionState(createPayment, null);

  return (
    <form action={formAction}>
      <input name="amount" type="number" />
      <input name="description" />
      {state?.errors?.amount && <p role="alert">{state.errors.amount}</p>}
      <button type="submit" disabled={isPending}>Submeter</button>
    </form>
  );
}
```

**Quando NÃO usar Server Action:** quando o consumidor não é um form/componente React desta app (ex: webhook do BNA, app mobile separada, integração externa) — nesse caso, Route Handler (`route.ts`) continua correto.

---

## Estrutura Interna de um Componente — Ordem Obrigatória

```typescript
// 1. Directiva (se client component)
"use client";

// 2. Imports externos (React primeiro, depois libs, depois Next.js)
import { useState, useCallback, useMemo } from "react";
import { useRouter } from "next/navigation";
import { toast } from "sonner";

// 3. Imports internos (types → hooks → utils → components)
import type { Payment } from "@/features/payments/types";
import { usePaymentFilter } from "@/features/payments/hooks/usePaymentFilter";
import { formatCurrency } from "@/lib/utils";
import { Button } from "@/components/ui/button";
import { PaymentCard } from "./PaymentCard";

// 4. Types do componente (só os que são LOCAIS — ver secção "Separação de Tipos")
interface PaymentListProps {
  initialPayments: Payment[];
  onPaymentSelect?: (id: string) => void;
  className?: string;
}

// 5. Componente principal (named export, não default export em features)
export function PaymentList({ initialPayments, onPaymentSelect, className }: PaymentListProps) {
  // 5a. State (useState antes de tudo)
  const [selectedId, setSelectedId] = useState<string | null>(null);

  // 5b. Hooks (useRouter, custom hooks)
  const router = useRouter();
  const { filtered, setFilter } = usePaymentFilter(initialPayments);

  // 5c. Computed values — só usar useMemo se o cálculo for realmente caro
  // (soma simples não justifica; ver secção "Memoização Criteriosa")
  const totalAmount = filtered.reduce((sum, p) => sum + p.amount, 0);

  // 5d. Handlers — só usar useCallback se o filho usar React.memo
  // (ver secção "Memoização Criteriosa")
  const handleSelect = (id: string) => {
    setSelectedId(id);
    onPaymentSelect?.(id);
  };

  // 5e. Effects (por último, antes do return)
  // useEffect apenas quando realmente necessário

  // 5f. Early returns (loading, error, empty states)
  if (filtered.length === 0) {
    return <EmptyState message="Sem pagamentos encontrados" />;
  }

  // 5g. Render
  return (
    <div className={cn("space-y-4", className)}>
      {filtered.map(payment => (
        <PaymentCard
          key={payment.id}
          payment={payment}
          isSelected={selectedId === payment.id}
          onSelect={handleSelect}
        />
      ))}
      <div className="text-sm text-muted-foreground">
        Total: {formatCurrency(totalAmount)}
      </div>
    </div>
  );
}

// 6. Sub-componentes pequenos no mesmo ficheiro (se usados só aqui)
function EmptyState({ message }: { message: string }) {
  return (
    <div className="flex h-32 items-center justify-center text-muted-foreground">
      {message}
    </div>
  );
}
```

---

## Memoização Criteriosa — `useCallback` / `useMemo` / `React.memo`

Estes hooks **não são grátis**: cada um adiciona overhead de comparação de dependências e complexidade de manutenção. Usar por defeito ("memoizar tudo") é um anti-padrão comum mas incorreto.

### Regra prática

```
useMemo   → só se o cálculo for genuinamente caro (loops grandes, ordenações,
            transformações pesadas) E houver re-renders frequentes do componente.
            ❌ NÃO usar para: somas simples, filtros de arrays pequenos,
               concatenação de strings.

useCallback → só compensa se a função for passada a um componente filho
              envolvido em React.memo(). Sem isso, useCallback sozinho
              NÃO evita re-renders — só adiciona overhead de memoização.

React.memo  → só vale para componentes que:
              (a) re-renderizam com frequência, E
              (b) recebem as mesmas props na maioria das vezes, E
              (c) o re-render é visivelmente caro (medido, não assumido).
```

```typescript
// ❌ Anti-padrão: memoizar sem motivo medido
const handleSelect = useCallback((id: string) => {
  setSelectedId(id);
}, []); // PaymentCard não está em memo() → isto não evita nada

// ✅ Só vale a pena combinado:
const PaymentCard = React.memo(function PaymentCard({ payment, isSelected, onSelect }: Props) {
  // ...
});

const handleSelect = useCallback((id: string) => {
  setSelectedId(id);
}, []); // agora sim, evita re-render do PaymentCard quando o pai re-renderiza
```

**Nota:** se o projeto já usa o **React Compiler** (Next.js 15+ com `experimental.reactCompiler`), a memoização é feita automaticamente em build-time — `useCallback`/`useMemo`/`React.memo` manuais tornam-se desnecessários na maioria dos casos.

---

## Tamanho de Componente — Quando Extrair

Não existe uma lei rígida, mas sinais práticos de que um componente precisa de ser dividido:

```
Sinal                                                    → Ação
─────────────────────────────────────────────────────────────────────────
> ~200 linhas no ficheiro                                → candidato a split
Precisas de scroll para ver o return ao ler as props     → candidato a split
> 2 responsabilidades distintas (fetch + filtro +
  paginação + seleção, por exemplo)                      → extrair hook(s)
Lógica de estado/efeitos reutilizada ou complexa          → custom hook (useX.ts)
Bloco de JSX condicional grande (>20 linhas)              → sub-componente
Cálculo sem dependência de estado/JSX                     → função utilitária pura
```

```typescript
// ❌ Tudo misturado num componente só
export function PaymentList({ initialPayments }: Props) {
  const [filter, setFilter] = useState("");
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const [page, setPage] = useState(1);
  // ... 150 linhas de lógica de filtro + paginação + seleção misturadas no JSX
}

// ✅ Responsabilidades extraídas
export function PaymentList({ initialPayments }: Props) {
  const { filtered, setFilter } = usePaymentFilter(initialPayments);
  const { page, setPage, paginated } = usePagination(filtered);
  const { selectedId, select } = usePaymentSelection();

  return (/* JSX limpo, só orquestração */);
}
```

---

## Separação de Código — TS, Tipos e Estilo

Esta é a divisão que evita ficheiros `.tsx` inchados e tipos duplicados entre componentes.

### a) Tipos — locais vs partilhados

```
Tipo usado por 1 componente só          → define dentro do próprio .tsx
Tipo usado por 2+ ficheiros (domínio)   → features/<feature>/types.ts
```

```typescript
// ✅ features/payments/types.ts — tipo de domínio partilhado
export interface Payment {
  id: string;
  amount: number;
  status: PaymentStatus;
  description: string;
}
export type PaymentStatus = "pending" | "completed" | "failed";

// ✅ PaymentList.tsx — tipo local, só usado aqui
interface PaymentListProps {
  initialPayments: Payment[]; // importado de types.ts
  onPaymentSelect?: (id: string) => void;
}

// ❌ Evitar: redefinir `Payment` dentro de cada componente que o consome
```

### b) Estilo — `cn()` inline vs `cva` para variantes

```typescript
// ✅ cn() inline — aceitável até ~2 condições
<div className={cn("rounded-lg border p-4", isActive && "border-primary")} />

// ❌ Quando o className cresce demasiado (3+ condições), fica ilegível
<div className={cn(
  "flex items-center justify-between rounded-lg border p-4 shadow-sm transition-colors",
  isActive && "border-primary bg-primary/5",
  isDisabled && "opacity-50 cursor-not-allowed",
  isCompact && "p-2 text-sm",
  className
)}>

// ✅ Extrair com cva (class-variance-authority) — mesma ferramenta usada
// internamente pelo shadcn/ui (ver button.tsx gerado)
import { cva, type VariantProps } from "class-variance-authority";

const cardVariants = cva(
  "flex items-center justify-between rounded-lg border p-4 shadow-sm transition-colors",
  {
    variants: {
      active: { true: "border-primary bg-primary/5", false: "" },
      disabled: { true: "opacity-50 cursor-not-allowed", false: "" },
      size: { default: "p-4", compact: "p-2 text-sm" },
    },
    defaultVariants: { active: false, disabled: false, size: "default" },
  }
);

interface CardProps extends VariantProps<typeof cardVariants> {
  className?: string;
}

<div className={cn(cardVariants({ active, disabled, size }), className)} />
```

### c) Lógica pura — o que NUNCA deve estar dentro de um `.tsx`

```
Componente (.tsx)    → só JSX + orquestração de hooks. Sem lógica de cálculo
                        complexa, sem transformação de dados pesada.
Hook (use*.ts)        → estado e efeitos reutilizáveis. Pode usar React,
                        mas não retorna JSX.
Util (*.ts)           → funções puras, sem estado, sem hooks, sem JSX.
                        Testáveis sem importar React.
Types (types.ts)      → contratos de dados partilhados entre ficheiros.
```

**Teste prático:** se uma função não usa `useState`/`useEffect`/JSX e pode ser testada com `expect(fn(input)).toBe(output)` sem importar React, **não devia estar dentro do `.tsx`** — pertence a `lib/` ou `utils.ts`.

```typescript
// ✅ lib/utils.ts — função pura, testável isoladamente, sem React
export function formatCurrency(amount: number, currency = "AOA"): string {
  return new Intl.NumberFormat("pt-AO", { style: "currency", currency }).format(amount);
}

// ❌ Evitar: lógica de formatação/transformação pesada inline no JSX
return <div>{new Intl.NumberFormat("pt-AO", { style: "currency", currency: "AOA" }).format(amount)}</div>;
```

### d) Ficheiros separados por preocupação — Comportamento vs Tipagem vs Estilo

Assim que um componente deixa de ser trivial (tem variantes de estilo, ou o tipo de props é reutilizado), divide em três ficheiros lado a lado:

```
features/payments/components/
├── PaymentCard.tsx          # comportamento — só JSX + hooks + orquestração
├── PaymentCard.types.ts     # tipagem — props e tipos locais do componente
└── PaymentCard.styles.ts    # estilo — variantes cva, tokens de classe
```

```typescript
// PaymentCard.types.ts — tipagem isolada
import type { Payment } from "@/features/payments/types";

export interface PaymentCardProps {
  payment: Payment;
  isSelected?: boolean;
  onSelect?: (id: string) => void;
  className?: string;
}

// PaymentCard.styles.ts — estilo isolado (cva)
import { cva } from "class-variance-authority";

export const paymentCardVariants = cva(
  "flex items-center justify-between rounded-lg border p-4 shadow-sm transition-colors",
  {
    variants: {
      selected: { true: "border-primary bg-primary/5", false: "" },
    },
    defaultVariants: { selected: false },
  }
);

// PaymentCard.tsx — só comportamento/JSX
import { cn } from "@/lib/utils";
import type { PaymentCardProps } from "./PaymentCard.types";
import { paymentCardVariants } from "./PaymentCard.styles";

export function PaymentCard({ payment, isSelected, onSelect, className }: PaymentCardProps) {
  return (
    <div
      className={cn(paymentCardVariants({ selected: isSelected }), className)}
      onClick={() => onSelect?.(payment.id)}
    >
      {/* ... */}
    </div>
  );
}
```

**Quando NÃO dividir:** componentes pequenos e triviais (sem variantes de estilo, prop única, tipo usado só ali) podem manter tipo e estilo inline no próprio `.tsx` — dividir por dividir aumenta o número de ficheiros sem ganho real. A divisão compensa quando `.types.ts` ou `.styles.ts` teriam mais do que ~3-4 linhas de conteúdo genuíno.

---

## Princípios SOLID em Componentes e Hooks

### S — Single Responsibility
Um componente/hook tem uma única razão para mudar. Se `PaymentList` faz fetch, filtro, paginação e selecção, tem 4 razões para mudar — extrai um hook por responsabilidade (ver secção "Tamanho de Componente — Quando Extrair").

```typescript
// ❌ Uma responsabilidade a mais: PaymentCard também formata e valida
export function PaymentCard({ payment }: Props) {
  const formatted = new Intl.NumberFormat("pt-AO", { style: "currency", currency: "AOA" }).format(payment.amount);
  const isValid = payment.amount > 0 && payment.recipientAccountNumber.length === 11;
  // ...
}

// ✅ Formatação e validação delegadas a funções/hooks dedicados
import { formatCurrency } from "@/lib/utils";
import { usePaymentValidation } from "../hooks/usePaymentValidation";

export function PaymentCard({ payment }: Props) {
  const { isValid } = usePaymentValidation(payment);
  return <div>{formatCurrency(payment.amount)}</div>;
}
```

### O — Open/Closed
Componentes devem ser extensíveis via props/composição sem editar o componente a cada novo caso de uso.

```typescript
// ❌ Fechado — cada novo tipo de botão exige editar o componente
function Button({ variant }: { variant: "danger" | "primary" }) {
  if (variant === "danger") return <button className="bg-red-500">...</button>;
  if (variant === "primary") return <button className="bg-blue-500">...</button>;
}

// ✅ Aberto para extensão via cva — adicionar variante não exige tocar na lógica
const buttonVariants = cva("...", { variants: { variant: { danger: "bg-red-500", primary: "bg-blue-500" } } });
function Button({ variant, ...props }: ButtonProps) {
  return <button className={buttonVariants({ variant })} {...props} />;
}
```

### L — Liskov Substitution
Uma variante/wrapper de um componente (ex: `PrimaryButton` sobre `Button`) deve poder substituir o componente base sem quebrar quem o consome.

```typescript
// ✅ PrimaryButton estende Button sem alterar o contrato — pode ser usado onde Button é esperado
interface PrimaryButtonProps extends React.ComponentProps<typeof Button> {
  isLoading?: boolean;
}
export function PrimaryButton({ isLoading, ...props }: PrimaryButtonProps) {
  return <Button disabled={isLoading || props.disabled} {...props} />;
}
```

### I — Interface Segregation
Props/interfaces pequenas e específicas por caso de uso — não uma prop gigante partilhada por componentes não relacionados.

```typescript
// ❌ Interface inchada — força PaymentCard a saber de campos que não usa
interface SharedEntityProps {
  payment?: Payment;
  user?: User;
  invoice?: Invoice;
  onPaymentSelect?: (id: string) => void;
  onUserSelect?: (id: string) => void;
}

// ✅ Interface segregada — cada componente só recebe o que precisa
interface PaymentCardProps {
  payment: Payment;
  onSelect?: (id: string) => void;
}
```

### D — Dependency Inversion
Componentes e Server Actions dependem de abstrações (queries, gateways, interfaces), nunca de implementações concretas de infraestrutura.

```typescript
// ❌ Componente/action depende directamente do SDK externo
import Stripe from "stripe";
export async function createPayment(data: PaymentInput) {
  const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
  await stripe.paymentIntents.create({ amount: data.amount });
}

// ✅ Depende de uma abstração (gateway/interface); a implementação concreta é injectada
interface PaymentGateway {
  charge(input: PaymentInput): Promise<{ id: string }>;
}
export async function createPayment(data: PaymentInput, gateway: PaymentGateway) {
  return gateway.charge(data);
}
```

---

## Padrões de UI com shadcn/ui

### Instalação e organização
```bash
# Adicionar componentes shadcn/ui
npx shadcn@latest add button input dialog table form

# Ficam em src/components/ui/ — não editar directamente
# Extender com wrapper components em src/components/
```

### Wrapper para customização
```typescript
// src/components/ui/button.tsx — shadcn original (não tocar)
// src/components/PrimaryButton.tsx — wrapper customizado
import { Button } from "@/components/ui/button";
import { Loader2 } from "lucide-react";

interface PrimaryButtonProps extends React.ComponentProps<typeof Button> {
  isLoading?: boolean;
}

export function PrimaryButton({ isLoading, children, disabled, ...props }: PrimaryButtonProps) {
  return (
    <Button disabled={isLoading || disabled} {...props}>
      {isLoading && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
      {children}
    </Button>
  );
}
```

### cn() — merge de classes Tailwind
```typescript
// src/lib/utils.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

// Uso nos componentes:
<div className={cn("base-classes", isActive && "active-class", className)} />
```

---

## Loading States — Padrões

### loading.tsx — Skeleton automático do Next.js
```typescript
// app/(dashboard)/payments/loading.tsx
import { Skeleton } from "@/components/ui/skeleton";

export default function Loading() {
  return (
    <div className="space-y-4">
      {Array.from({ length: 5 }).map((_, i) => (
        <Skeleton key={i} className="h-20 w-full rounded-lg" />
      ))}
    </div>
  );
}
```

### Suspense granular (streaming)
```typescript
// page.tsx com múltiplas secções independentes
import { Suspense } from "react";

export default function DashboardPage() {
  return (
    <div className="grid grid-cols-2 gap-4">
      {/* Cada secção carrega independentemente */}
      <Suspense fallback={<StatsSkeleton />}>
        <DashboardStats />        {/* async Server Component */}
      </Suspense>
      <Suspense fallback={<ActivitySkeleton />}>
        <RecentActivity />        {/* async Server Component */}
      </Suspense>
    </div>
  );
}
```

---

## Error States — Padrões

### error.tsx — DEVE ser "use client"
```typescript
// app/(dashboard)/payments/error.tsx
"use client";

import { useEffect } from "react";
import { AlertCircle } from "lucide-react";
import { Button } from "@/components/ui/button";

interface ErrorProps {
  error: Error & { digest?: string };
  reset: () => void;
}

export default function Error({ error, reset }: ErrorProps) {
  // Log no cliente (Sentry capta automaticamente)
  useEffect(() => {
    console.error("Payment page error:", error);
  }, [error]);

  return (
    <div className="flex flex-col items-center gap-4 p-8">
      <AlertCircle className="h-12 w-12 text-destructive" />
      <h2 className="text-lg font-semibold">Algo correu mal</h2>
      <p className="text-muted-foreground">{error.message}</p>
      <Button onClick={reset}>Tentar novamente</Button>
    </div>
  );
}
```

---

## Acessibilidade — Checklist Mínimo

```typescript
// ✅ Sempre presente em elementos interactivos
<button
  aria-label="Fechar diálogo"
  aria-expanded={isOpen}
  onClick={handleClose}
>
  <X className="h-4 w-4" aria-hidden="true" />
</button>

// ✅ Formulários com labels associadas
<label htmlFor="email">Email</label>
<input id="email" type="email" aria-describedby="email-error" />
{error && <p id="email-error" role="alert">{error}</p>}

// ✅ Imagens com alt descritivo
<Image src="/logo.png" alt="Logo BPC — Banco de Poupança e Crédito" />

// ✅ Landmarks semânticos
<header>, <nav>, <main>, <aside>, <footer>
// Não usar <div> para estrutura semântica
```

---

## Default vs Named Exports

```typescript
// ✅ Named exports em features (facilita refactoring e barrel exports)
// features/payments/components/PaymentCard.tsx
export function PaymentCard(...) {}

// ✅ Default exports APENAS em ficheiros exigidos pelo Next.js
// app/.../page.tsx, layout.tsx, error.tsx, loading.tsx, not-found.tsx
export default function Page() {}
export default function Layout(...) {}
```

---

## Resumo — Checklist Rápido

```
[ ] Server Component por defeito; "use client" só no mínimo necessário
[ ] Mutação simples (form) → Server Action; endpoint externo → Route Handler
[ ] Ordem interna do componente respeitada (state → hooks → computed → handlers → effects → return)
[ ] useMemo/useCallback só com motivo medido; React.memo só com (a)+(b)+(c)
[ ] Componente > ~200 linhas ou > 2 responsabilidades → extrair hook/sub-componente
[ ] Tipo usado em 2+ ficheiros → features/<feature>/types.ts, não duplicado
[ ] className com 3+ condições → migrar para cva
[ ] Função sem useState/useEffect/JSX → lib/utils.ts, não dentro do .tsx
[ ] shadcn/ui original intocado; customização via wrapper component
[ ] error.tsx com "use client" e imports completos (useEffect incluído)
[ ] Acessibilidade: aria-label, labels associadas, alt descritivo, landmarks semânticos
[ ] Named exports em features; default export só onde o Next.js exige
[ ] Identificadores (vars, funções, tipos) em inglês; só UI copy em português
[ ] Componente não-trivial dividido em .tsx / .types.ts / .styles.ts
[ ] SOLID: responsabilidade única, extensão via composição, props segregadas, dependência de abstrações
```

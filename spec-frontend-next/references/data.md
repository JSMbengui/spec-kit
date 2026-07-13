# Data Fetching, Server Actions e Route Handlers

## Hierarquia de Padrões de Dados

```
Renderizar dados numa page   →  Server Component + query function
Mutação (criar/editar/apagar) →  Server Action
API pública ou webhook        →  Route Handler (app/api/)
Dados reactivos no cliente    →  React Query (useQuery)
Polling / real-time           →  React Query + refetchInterval / WebSocket
```

---

## Query Functions (fetch server-side)

### Padrão base
```typescript
// features/payments/queries/get-payments.ts
import { db } from "@/lib/db";
import { auth } from "@/lib/auth";
import { cache } from "react";

export type GetPaymentsOptions = {
  page?: number;
  limit?: number;
  status?: PaymentStatus;
};

// cache() deduplica chamadas no mesmo render tree (React cache)
export const getPayments = cache(async (options: GetPaymentsOptions = {}) => {
  const session = await auth();
  if (!session?.user) throw new Error("Unauthorized");

  const { page = 1, limit = 20, status } = options;
  const skip = (page - 1) * limit;

  const [payments, total] = await Promise.all([
    db.payment.findMany({
      where: { userId: session.user.id, ...(status && { status }) },
      orderBy: { createdAt: "desc" },
      skip,
      take: limit,
      select: {              // nunca SELECT * — seleccionar apenas o necessário
        id: true,
        amount: true,
        status: true,
        createdAt: true,
        recipient: { select: { name: true, accountNumber: true } },
      },
    }),
    db.payment.count({ where: { userId: session.user.id } }),
  ]);

  return { payments, total, page, totalPages: Math.ceil(total / limit) };
});
```

### Com tratamento de erro
```typescript
// Para queries que podem falhar graciosamente
export async function getPaymentById(id: string) {
  try {
    const session = await auth();
    if (!session?.user) return null;

    return await db.payment.findFirst({
      where: { id, userId: session.user.id },
    });
  } catch (err) {
    // Log mas não crash — a page.tsx lida com null
    logger.error({ err, id }, "Failed to fetch payment");
    return null;
  }
}

// page.tsx usa notFound() para null
export default async function PaymentDetailPage({ params }: Props) {
  const { id } = await params;
  const payment = await getPaymentById(id);
  if (!payment) notFound(); // renderiza not-found.tsx
  return <PaymentDetail payment={payment} />;
}
```

---

## Server Actions — Padrão Completo

### Action básica
```typescript
// features/payments/actions/create-payment.ts
"use server";

import { z } from "zod";
import { revalidatePath } from "next/cache";
import { auth } from "@/lib/auth";
import { db } from "@/lib/db";
import { logger } from "@/lib/logger";

// Schema Zod — validação robusta
const CreatePaymentSchema = z.object({
  amount: z
    .number({ invalid_type_error: "Montante deve ser um número" })
    .positive("Montante deve ser positivo")
    .max(1_000_000, "Montante máximo excedido"),
  recipientAccountNumber: z
    .string()
    .regex(/^\d{11}$/, "Número de conta inválido (11 dígitos)"),
  description: z.string().max(200).optional(),
});

// Tipo do resultado — sempre retorna { data } ou { error }, nunca lança
export type CreatePaymentResult =
  | { data: { id: string }; error?: never }
  | { data?: never; error: Record<string, string[]> };

export async function createPayment(
  formData: FormData
): Promise<CreatePaymentResult> {
  // 1. Auth
  const session = await auth();
  if (!session?.user) {
    return { error: { general: ["Sessão expirada. Faz login novamente."] } };
  }

  // 2. Validação
  const parsed = CreatePaymentSchema.safeParse({
    amount: Number(formData.get("amount")),
    recipientAccountNumber: formData.get("recipientAccountNumber"),
    description: formData.get("description") || undefined,
  });

  if (!parsed.success) {
    return { error: parsed.error.flatten().fieldErrors };
  }

  // 3. Lógica de negócio
  try {
    const payment = await db.payment.create({
      data: {
        ...parsed.data,
        userId: session.user.id,
        status: "PENDING",
      },
      select: { id: true },
    });

    // 4. Invalidar cache das rotas afectadas
    revalidatePath("/payments");
    revalidatePath(`/dashboard`);

    return { data: payment };
  } catch (err) {
    logger.error({ err, userId: session.user.id }, "Payment creation failed");
    return { error: { general: ["Pagamento falhou. Tenta novamente."] } };
  }
}
```

### Action com redirect
```typescript
"use server";
import { redirect } from "next/navigation";

export async function createAndRedirect(formData: FormData) {
  const result = await createPayment(formData);
  if (result.error) return result; // devolve erros ao form
  redirect(`/payments/${result.data.id}?success=true`);
  // redirect() lança internamente — não precisa de return
}
```

### Usar Server Action com useActionState (React 19)
```typescript
"use client";
import { useActionState } from "react";
import { createPayment } from "../actions/create-payment";

const initialState = { error: undefined, data: undefined };

export function PaymentForm() {
  const [state, formAction, isPending] = useActionState(createPayment, initialState);

  return (
    <form action={formAction}>
      <input name="amount" type="number" />
      {state.error?.amount && (
        <p className="text-sm text-destructive">{state.error.amount[0]}</p>
      )}
      <button type="submit" disabled={isPending}>
        {isPending ? "A processar..." : "Pagar"}
      </button>
    </form>
  );
}
```

---

## Route Handlers — Quando e Como

### Usar Route Handler (não Server Action) quando:
- API consumida por clientes externos (mobile app, parceiros)
- Webhooks de serviços externos (Stripe, PayPal)
- Upload de ficheiros (multipart/form-data)
- SSE (Server-Sent Events) ou streaming de respostas
- Autenticação OAuth callback

### Padrão de Route Handler
```typescript
// app/api/payments/route.ts
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";
import { auth } from "@/lib/auth";

// GET — listar
export async function GET(request: NextRequest) {
  const session = await auth();
  if (!session?.user) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const { searchParams } = request.nextUrl;
  const page = Number(searchParams.get("page") ?? "1");

  const result = await getPayments({ page });
  return NextResponse.json(result);
}

// POST — criar
export async function POST(request: NextRequest) {
  const session = await auth();
  if (!session?.user) {
    return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
  }

  const body = await request.json();
  const parsed = CreatePaymentSchema.safeParse(body);

  if (!parsed.success) {
    return NextResponse.json(
      { error: parsed.error.flatten() },
      { status: 422 }
    );
  }

  const payment = await createPaymentInDb(parsed.data, session.user.id);
  return NextResponse.json(payment, { status: 201 });
}
```

### Webhook com verificação de assinatura
```typescript
// app/api/webhooks/stripe/route.ts
import { headers } from "next/headers";
import Stripe from "stripe";

export async function POST(request: NextRequest) {
  const body = await request.text();
  const signature = (await headers()).get("stripe-signature")!;

  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, signature, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch {
    return NextResponse.json({ error: "Invalid signature" }, { status: 400 });
  }

  switch (event.type) {
    case "payment_intent.succeeded":
      await handlePaymentSuccess(event.data.object);
      break;
    case "payment_intent.payment_failed":
      await handlePaymentFailure(event.data.object);
      break;
  }

  return NextResponse.json({ received: true });
}
```

---

## React Query — Dados Reactivos no Cliente

### Setup do Provider
```typescript
// src/components/providers/QueryProvider.tsx
"use client";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState } from "react";

export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,        // 1 minuto
        retry: (count, error) => count < 2 && !isAuthError(error),
      },
    },
  }));
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>;
}
```

### Hook de query
```typescript
// features/payments/hooks/usePayments.ts
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

export const paymentKeys = {
  all: ["payments"] as const,
  list: (filters?: PaymentFilters) => [...paymentKeys.all, "list", filters] as const,
  detail: (id: string) => [...paymentKeys.all, "detail", id] as const,
};

export function usePayments(filters?: PaymentFilters) {
  return useQuery({
    queryKey: paymentKeys.list(filters),
    queryFn: () => fetch(`/api/payments?${new URLSearchParams(filters as Record<string,string>)}`).then(r => r.json()),
  });
}

export function useCancelPayment() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (id: string) =>
      fetch(`/api/payments/${id}/cancel`, { method: "POST" }).then(r => r.json()),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: paymentKeys.all });
    },
  });
}
```

---

## Estratégia de Cache

```typescript
// Force static — página totalmente estática
export const dynamic = "force-static";

// Force dynamic — sempre re-render (dados de utilizador, etc.)
export const dynamic = "force-dynamic";

// Revalidação a cada N segundos (ISR)
export const revalidate = 3600; // 1 hora

// fetch() com cache explícito
const data = await fetch("https://api.external.com/data", {
  next: { revalidate: 300 },  // revalidar a cada 5 minutos
});

// fetch() sem cache (sempre fresco)
const data = await fetch("https://api.external.com/data", {
  cache: "no-store",
});

// Invalidar cache manualmente após mutação
revalidatePath("/payments");               // invalida a rota
revalidatePath("/payments", "layout");     // invalida o layout também
revalidateTag("payments");                 // invalida por tag (fetch com next.tags)
```
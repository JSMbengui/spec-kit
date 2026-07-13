# Estado, Formulários e Validação

## Zustand — Estado Global

### Quando usar estado global
- Dados do utilizador autenticado (sessão)
- Preferências de UI (tema, sidebar aberta/fechada)
- Estado partilhado entre rotas não relacionadas
- Shopping cart, notificações, filtros globais

### Padrão de store
```typescript
// src/store/ui.store.ts
import { create } from "zustand";
import { persist } from "zustand/middleware";

interface UIState {
  isSidebarOpen: boolean;
  theme: "light" | "dark" | "system";
  toggleSidebar: () => void;
  setTheme: (theme: UIState["theme"]) => void;
}

export const useUIStore = create<UIState>()(
  persist(
    (set) => ({
      isSidebarOpen: true,
      theme: "system",
      toggleSidebar: () => set((s) => ({ isSidebarOpen: !s.isSidebarOpen })),
      setTheme: (theme) => set({ theme }),
    }),
    {
      name: "ui-preferences",    // chave no localStorage
      partialize: (state) => ({ theme: state.theme }), // só persistir o tema
    }
  )
);

// Uso no componente
const { isSidebarOpen, toggleSidebar } = useUIStore();
```

### Store com lógica de domínio
```typescript
// features/cart/store/cart.store.ts
import { create } from "zustand";

interface CartItem { id: string; name: string; price: number; qty: number }

interface CartStore {
  items: CartItem[];
  total: number;                     // computado
  addItem: (item: Omit<CartItem, "qty">) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
}

export const useCartStore = create<CartStore>((set, get) => ({
  items: [],
  get total() {
    return get().items.reduce((sum, i) => sum + i.price * i.qty, 0);
  },
  addItem: (item) => set((state) => {
    const existing = state.items.find(i => i.id === item.id);
    if (existing) {
      return { items: state.items.map(i => i.id === item.id ? { ...i, qty: i.qty + 1 } : i) };
    }
    return { items: [...state.items, { ...item, qty: 1 }] };
  }),
  removeItem: (id) => set((s) => ({ items: s.items.filter(i => i.id !== id) })),
  clearCart: () => set({ items: [] }),
}));
```

---

## React Hook Form + Zod — Formulários Complexos

### Schema Zod partilhado
```typescript
// features/payments/schemas/payment.schema.ts
import { z } from "zod";

export const CreatePaymentSchema = z.object({
  amount: z
    .string()
    .min(1, "Montante é obrigatório")
    .transform(Number)
    .pipe(z.number().positive("Montante deve ser positivo")),
  recipientAccountNumber: z
    .string()
    .regex(/^\d{11}$/, "Número de conta inválido (11 dígitos)"),
  description: z.string().max(200, "Máximo 200 caracteres").optional(),
  scheduledDate: z
    .string()
    .optional()
    .transform(val => val ? new Date(val) : undefined),
});

export type CreatePaymentInput = z.infer<typeof CreatePaymentSchema>;
```

### Formulário completo com RHF + Zod + Server Action
```typescript
"use client";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { useTransition } from "react";
import { toast } from "sonner";
import { useRouter } from "next/navigation";
import { CreatePaymentSchema, type CreatePaymentInput } from "../schemas/payment.schema";
import { createPayment } from "../actions/create-payment";

export function PaymentForm() {
  const router = useRouter();
  const [isPending, startTransition] = useTransition();

  const form = useForm<CreatePaymentInput>({
    resolver: zodResolver(CreatePaymentSchema),
    defaultValues: {
      amount: "",
      recipientAccountNumber: "",
      description: "",
    },
  });

  const onSubmit = (data: CreatePaymentInput) => {
    startTransition(async () => {
      const formData = new FormData();
      Object.entries(data).forEach(([key, value]) => {
        if (value !== undefined) formData.set(key, String(value));
      });

      const result = await createPayment(formData);

      if (result.error) {
        // Mapear erros do servidor para os campos do formulário
        if (result.error.amount) {
          form.setError("amount", { message: result.error.amount[0] });
        }
        if (result.error.general) {
          toast.error(result.error.general[0]);
        }
        return;
      }

      toast.success("Pagamento criado com sucesso!");
      router.push(`/payments/${result.data.id}`);
    });
  };

  return (
    <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
      <div>
        <label htmlFor="amount">Montante (AOA)</label>
        <input
          id="amount"
          type="number"
          step="0.01"
          {...form.register("amount")}
          aria-describedby={form.formState.errors.amount ? "amount-error" : undefined}
        />
        {form.formState.errors.amount && (
          <p id="amount-error" role="alert" className="text-sm text-destructive">
            {form.formState.errors.amount.message}
          </p>
        )}
      </div>

      <div>
        <label htmlFor="recipientAccountNumber">Número de conta</label>
        <input id="recipientAccountNumber" {...form.register("recipientAccountNumber")} />
        {form.formState.errors.recipientAccountNumber && (
          <p role="alert" className="text-sm text-destructive">
            {form.formState.errors.recipientAccountNumber.message}
          </p>
        )}
      </div>

      <button type="submit" disabled={isPending}>
        {isPending ? "A processar..." : "Confirmar pagamento"}
      </button>
    </form>
  );
}
```

### Form com shadcn/ui Form components
```typescript
"use client";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import {
  Form, FormControl, FormDescription, FormField, FormItem, FormLabel, FormMessage,
} from "@/components/ui/form";
import { Input } from "@/components/ui/input";
import { Button } from "@/components/ui/button";

export function ProfileForm() {
  const form = useForm<ProfileInput>({
    resolver: zodResolver(ProfileSchema),
    defaultValues: { name: "", email: "" },
  });

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <FormField
          control={form.control}
          name="name"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Nome</FormLabel>
              <FormControl>
                <Input placeholder="João Silva" {...field} />
              </FormControl>
              <FormDescription>Nome que aparece no seu perfil.</FormDescription>
              <FormMessage /> {/* mostra erros automaticamente */}
            </FormItem>
          )}
        />
        <Button type="submit" disabled={form.formState.isSubmitting}>
          Guardar
        </Button>
      </form>
    </Form>
  );
}
```

---

## Validação de Variáveis de Ambiente

```typescript
// src/env.ts — validação no boot com @t3-oss/env-nextjs
import { createEnv } from "@t3-oss/env-nextjs";
import { z } from "zod";

export const env = createEnv({
  server: {
    DATABASE_URL: z.string().url("DATABASE_URL inválida"),
    NEXTAUTH_SECRET: z.string().min(32, "Secret muito curto (mín. 32 chars)"),
    NEXTAUTH_URL: z.string().url(),
    STRIPE_SECRET_KEY: z.string().startsWith("sk_"),
    STRIPE_WEBHOOK_SECRET: z.string().startsWith("whsec_"),
  },
  client: {
    NEXT_PUBLIC_APP_URL: z.string().url(),
    NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: z.string().startsWith("pk_"),
  },
  // Next.js exige esta config para variáveis de cliente
  experimental__runtimeEnv: {
    NEXT_PUBLIC_APP_URL: process.env.NEXT_PUBLIC_APP_URL,
    NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY: process.env.NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY,
  },
});

// Uso — em vez de process.env.DATABASE_URL directamente
import { env } from "@/env";
const db = new PrismaClient({ datasourceUrl: env.DATABASE_URL });
```

---

## Middleware — Auth e Redirects

```typescript
// middleware.ts (na raiz, não em src/)
import { auth } from "@/lib/auth";
import { NextResponse } from "next/server";

const PUBLIC_ROUTES = ["/", "/about", "/pricing", "/login", "/register"];
const API_ROUTES = /^\/api\//;

export default auth((request) => {
  const { nextUrl, auth: session } = request;
  const isPublic = PUBLIC_ROUTES.includes(nextUrl.pathname);
  const isApi = API_ROUTES.test(nextUrl.pathname);

  // API routes — auth verificada internamente
  if (isApi) return NextResponse.next();

  // Rota protegida sem sessão → redirect para login
  if (!isPublic && !session) {
    const loginUrl = new URL("/login", nextUrl);
    loginUrl.searchParams.set("callbackUrl", nextUrl.pathname);
    return NextResponse.redirect(loginUrl);
  }

  // Utilizador autenticado na página de login → redirect para dashboard
  if (session && nextUrl.pathname === "/login") {
    return NextResponse.redirect(new URL("/dashboard", nextUrl));
  }

  return NextResponse.next();
});

export const config = {
  // Aplicar middleware a todas as rotas excepto ficheiros estáticos
  matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|ico)$).*)"],
};
```
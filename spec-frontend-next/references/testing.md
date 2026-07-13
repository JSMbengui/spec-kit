# Testes — Estratégia, Ferramentas e Padrões

## Stack de Testes

| Tipo                              | Ferramenta               | Cobertura mínima     |
| --------------------------------- | ------------------------ | -------------------- |
| Unit (utils, hooks, schemas)      | Vitest                   | 80%                  |
| Component (Client Components)     | Testing Library + Vitest | 70%                  |
| Integration (Server Actions, API) | Vitest + MSW             | 70%                  |
| E2E (fluxos críticos)             | Playwright               | Fluxos críticos 100% |

---

## Configuração — Vitest

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./tests/setup.ts"],
    coverage: {
      provider: "v8",
      thresholds: { lines: 80, branches: 70, functions: 80 },
      exclude: [
        "src/components/ui/**",   // shadcn/ui — não testar gerado
        "src/app/**",             // routing files
        "**/*.config.*",
        "**/*.d.ts",
        "prisma/**",
      ],
    },
  },
  resolve: {
    alias: { "@": path.resolve(__dirname, "./src") },
  },
});

// tests/setup.ts
import "@testing-library/jest-dom";
import { cleanup } from "@testing-library/react";
import { afterEach, vi } from "vitest";

afterEach(() => cleanup());

// Mock next/navigation globalmente
vi.mock("next/navigation", () => ({
  useRouter: () => ({ push: vi.fn(), replace: vi.fn(), prefetch: vi.fn() }),
  usePathname: () => "/",
  useSearchParams: () => new URLSearchParams(),
  redirect: vi.fn(),
  notFound: vi.fn(),
}));

// Mock next/cache
vi.mock("next/cache", () => ({
  revalidatePath: vi.fn(),
  revalidateTag: vi.fn(),
}));
```

---

## Testes de Zod Schemas

```typescript
// features/payments/schemas/__tests__/payment.schema.test.ts
import { describe, it, expect } from "vitest";
import { CreatePaymentSchema } from "../payment.schema";

describe("CreatePaymentSchema", () => {
  const validInput = {
    amount: "1000",
    recipientAccountNumber: "12345678901",
    description: "Pagamento de renda",
  };

  it("parses valid input correctly", () => {
    const result = CreatePaymentSchema.safeParse(validInput);
    expect(result.success).toBe(true);
    if (result.success) {
      expect(result.data.amount).toBe(1000); // transformado para number
    }
  });

  it("rejects negative amount", () => {
    const result = CreatePaymentSchema.safeParse({ ...validInput, amount: "-50" });
    expect(result.success).toBe(false);
    if (!result.success) {
      expect(result.error.flatten().fieldErrors.amount).toBeDefined();
    }
  });

  it("rejects account number with wrong length", () => {
    const result = CreatePaymentSchema.safeParse({
      ...validInput,
      recipientAccountNumber: "1234",  // menos de 11 dígitos
    });
    expect(result.success).toBe(false);
  });
});
```

---

## Testes de Server Actions

```typescript
// features/payments/actions/__tests__/create-payment.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { createPayment } from "../create-payment";

// Mock das dependências
vi.mock("@/lib/auth", () => ({
  auth: vi.fn(),
}));
vi.mock("@/lib/db", () => ({
  db: { payment: { create: vi.fn() } },
}));

import { auth } from "@/lib/auth";
import { db } from "@/lib/db";

describe("createPayment", () => {
  beforeEach(() => vi.clearAllMocks());

  const makeFormData = (overrides = {}) => {
    const fd = new FormData();
    const data = {
      amount: "1000",
      recipientAccountNumber: "12345678901",
      ...overrides,
    };
    Object.entries(data).forEach(([k, v]) => fd.set(k, String(v)));
    return fd;
  };

  it("returns error when not authenticated", async () => {
    vi.mocked(auth).mockResolvedValue(null);
    const result = await createPayment(makeFormData());
    expect(result.error?.general).toBeDefined();
  });

  it("returns validation errors on invalid input", async () => {
    vi.mocked(auth).mockResolvedValue({ user: { id: "u1" } } as any);
    const result = await createPayment(makeFormData({ amount: "-100" }));
    expect(result.error?.amount).toBeDefined();
    expect(db.payment.create).not.toHaveBeenCalled();
  });

  it("creates payment and returns data on valid input", async () => {
    vi.mocked(auth).mockResolvedValue({ user: { id: "u1" } } as any);
    vi.mocked(db.payment.create).mockResolvedValue({ id: "pay-123" } as any);

    const result = await createPayment(makeFormData());

    expect(result.data?.id).toBe("pay-123");
    expect(db.payment.create).toHaveBeenCalledWith({
      data: expect.objectContaining({
        amount: 1000,
        userId: "u1",
        status: "PENDING",
      }),
      select: { id: true },
    });
  });

  it("returns generic error when DB throws", async () => {
    vi.mocked(auth).mockResolvedValue({ user: { id: "u1" } } as any);
    vi.mocked(db.payment.create).mockRejectedValue(new Error("DB error"));

    const result = await createPayment(makeFormData());
    expect(result.error?.general).toBeDefined();
  });
});
```

---

## Testes de Client Components

```typescript
// features/payments/components/__tests__/PaymentCard.test.tsx
import { render, screen, fireEvent } from "@testing-library/react";
import { describe, it, expect, vi } from "vitest";
import { PaymentCard } from "../PaymentCard";
import { createMockPayment } from "@/tests/fixtures/payment.fixture";

describe("PaymentCard", () => {
  const mockPayment = createMockPayment({ amount: 5000, status: "COMPLETED" });

  it("renders payment details correctly", () => {
    render(<PaymentCard payment={mockPayment} onSelect={vi.fn()} />);
    expect(screen.getByText("5.000,00 AOA")).toBeInTheDocument();
    expect(screen.getByText("Concluído")).toBeInTheDocument();
  });

  it("calls onSelect with payment id when clicked", async () => {
    const onSelect = vi.fn();
    render(<PaymentCard payment={mockPayment} onSelect={onSelect} />);
    fireEvent.click(screen.getByRole("button", { name: /ver pagamento/i }));
    expect(onSelect).toHaveBeenCalledWith(mockPayment.id);
  });

  it("shows selected state visually when isSelected=true", () => {
    const { container } = render(
      <PaymentCard payment={mockPayment} onSelect={vi.fn()} isSelected />
    );
    expect(container.firstChild).toHaveClass("ring-2"); // classe de selecção
  });
});
```

---

## Fixtures — Dados de Teste Reutilizáveis

```typescript
// tests/fixtures/payment.fixture.ts
import type { Payment } from "@/features/payments/types";

let idCounter = 0;

export function createMockPayment(overrides?: Partial<Payment>): Payment {
  return {
    id: `pay-${++idCounter}`,
    amount: 1000,
    status: "PENDING",
    recipientAccountNumber: "12345678901",
    description: null,
    userId: "user-1",
    createdAt: new Date("2024-01-15"),
    updatedAt: new Date("2024-01-15"),
    ...overrides,
  };
}

export function createMockUser(overrides = {}) {
  return {
    id: `user-${++idCounter}`,
    name: "João Silva",
    email: "joao@bpc.ao",
    ...overrides,
  };
}
```

---

## Playwright — Testes E2E

```typescript
// tests/e2e/payments.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Payment flow", () => {
  test.beforeEach(async ({ page }) => {
    // Login via API (mais rápido que UI)
    await page.request.post("/api/test/login", {
      data: { email: "test@bpc.ao", password: "Test1234!" },
    });
    await page.goto("/payments");
  });

  test("creates a payment successfully", async ({ page }) => {
    await page.getByRole("link", { name: "Novo pagamento" }).click();
    await page.getByLabel("Montante").fill("1000");
    await page.getByLabel("Número de conta").fill("12345678901");
    await page.getByRole("button", { name: "Confirmar pagamento" }).click();

    await expect(page.getByText("Pagamento criado com sucesso")).toBeVisible();
    await expect(page).toHaveURL(/\/payments\/pay-/);
  });

  test("shows validation errors on invalid input", async ({ page }) => {
    await page.goto("/payments/new");
    await page.getByRole("button", { name: "Confirmar pagamento" }).click();
    await expect(page.getByText("Montante é obrigatório")).toBeVisible();
  });
});

// playwright.config.ts
import { defineConfig } from "@playwright/test";
export default defineConfig({
  testDir: "./tests/e2e",
  use: { baseURL: "http://localhost:3000", screenshot: "only-on-failure" },
  webServer: { command: "npm run dev", port: 3000, reuseExistingServer: true },
});
```

---

## MSW — Mock de APIs Externas em Testes

```typescript
// tests/mocks/handlers.ts
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/api/payments", () => {
    return HttpResponse.json({
      payments: [createMockPayment(), createMockPayment()],
      total: 2,
    });
  }),
  http.post("/api/payments", async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: "pay-new", ...body }, { status: 201 });
  }),
];

// tests/setup.ts (adicionar ao setup existente)
import { setupServer } from "msw/node";
import { handlers } from "./mocks/handlers";

const server = setupServer(...handlers);
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```
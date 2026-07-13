# Performance, SEO e Optimização

## Core Web Vitals — Metas

| Métrica                         | Meta    | O que afecta                             |
| ------------------------------- | ------- | ---------------------------------------- |
| LCP (Largest Contentful Paint)  | < 2.5s  | Imagens, fonts, Server Components lentos |
| INP (Interaction to Next Paint) | < 200ms | JavaScript no cliente, handlers pesados  |
| CLS (Cumulative Layout Shift)   | < 0.1   | Imagens sem dimensões, fonts sem preload |
| TTFB (Time to First Byte)       | < 600ms | Queries lentas, sem cache                |

---

## next/image — Obrigatório para todas as imagens

```typescript
import Image from "next/image";

// ✅ Imagem com dimensões conhecidas
<Image
  src="/hero.jpg"
  alt="Dashboard de pagamentos BPC"
  width={1200}
  height={600}
  priority        // LCP — adicionar à imagem acima do fold
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,..."  // gerar com plaiceholder
/>

// ✅ Imagem de preenchimento (fill) — container define o tamanho
<div className="relative h-64 w-full">
  <Image
    src={user.avatarUrl}
    alt={`Avatar de ${user.name}`}
    fill
    className="object-cover rounded-full"
    sizes="(max-width: 768px) 64px, 128px"   // sizes SEMPRE com fill
  />
</div>

// ✅ Imagens remotas — configurar domínios em next.config.ts
// next.config.ts
const config: NextConfig = {
  images: {
    remotePatterns: [
      { protocol: "https", hostname: "cdn.bpc.ao", pathname: "/images/**" },
    ],
  },
};
```

---

## next/font — Zero Layout Shift

```typescript
// src/app/layout.tsx
import { Inter, JetBrains_Mono } from "next/font/google";

const inter = Inter({
  subsets: ["latin"],
  variable: "--font-inter",
  display: "swap",        // evita FOIT (Flash of Invisible Text)
});

const mono = JetBrains_Mono({
  subsets: ["latin"],
  variable: "--font-mono",
  display: "swap",
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="pt" className={`${inter.variable} ${mono.variable}`}>
      <body className="font-sans">{children}</body>
    </html>
  );
}
```

---

## Metadata e SEO

### Metadata estática
```typescript
// app/about/page.tsx
import type { Metadata } from "next";

export const metadata: Metadata = {
  title: "Sobre Nós | BPC Portal",
  description: "Conheça o Banco de Poupança e Crédito.",
  openGraph: {
    title: "Sobre Nós | BPC Portal",
    description: "...",
    images: [{ url: "/og-about.jpg", width: 1200, height: 630 }],
  },
};
```

### Metadata dinâmica
```typescript
// app/payments/[id]/page.tsx
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { id } = await params;
  const payment = await getPaymentById(id);
  if (!payment) return { title: "Pagamento não encontrado" };
  return {
    title: `Pagamento ${payment.reference} | BPC Portal`,
    description: `Detalhes do pagamento de ${formatCurrency(payment.amount)}`,
  };
}
```

### Metadata base (herança em todo o site)
```typescript
// app/layout.tsx
export const metadata: Metadata = {
  metadataBase: new URL("https://portal.bpc.ao"),
  title: { template: "%s | BPC Portal", default: "BPC Portal" },
  description: "Banco de Poupança e Crédito — Portal de Banca Digital",
  robots: { index: true, follow: true },
};
```

---

## Streaming e Suspense

```typescript
// ✅ Streaming — página carrega progressivamente
// app/(dashboard)/dashboard/page.tsx
import { Suspense } from "react";

export default function DashboardPage() {
  return (
    <main className="space-y-6">
      {/* Renderiza imediatamente */}
      <PageHeader title="Dashboard" />

      {/* Cada secção carrega independentemente */}
      <Suspense fallback={<StatsSkeleton />}>
        <DashboardStats />   {/* async Server Component */}
      </Suspense>

      <div className="grid grid-cols-2 gap-4">
        <Suspense fallback={<ChartSkeleton />}>
          <PaymentsChart />  {/* pode demorar mais */}
        </Suspense>
        <Suspense fallback={<ActivitySkeleton />}>
          <RecentActivity /> {/* independente do chart */}
        </Suspense>
      </div>
    </main>
  );
}

// Componente assíncrono (Server Component)
async function DashboardStats() {
  const stats = await getDashboardStats(); // pode demorar
  return <StatsGrid stats={stats} />;
}
```

---

## Bundle e JavaScript

### Dynamic imports — lazy loading
```typescript
import dynamic from "next/dynamic";

// Componente carregado só quando necessário (ex: modal, editor)
const RichTextEditor = dynamic(() => import("@/components/RichTextEditor"), {
  loading: () => <div className="h-64 animate-pulse bg-muted rounded" />,
  ssr: false,  // componentes que usam browser APIs
});

// Uso normal — Next.js gere o lazy loading
export function EditPage() {
  const [isEditing, setIsEditing] = useState(false);
  return (
    <>
      <button onClick={() => setIsEditing(true)}>Editar</button>
      {isEditing && <RichTextEditor />}  {/* carregado só aqui */}
    </>
  );
}
```

### Analisar bundle
```bash
# Instalar
npm install @next/bundle-analyzer

# next.config.ts
import bundleAnalyzer from "@next/bundle-analyzer";
const withBundleAnalyzer = bundleAnalyzer({ enabled: process.env.ANALYZE === "true" });
export default withBundleAnalyzer(nextConfig);

# Correr
ANALYZE=true npm run build
```

---

## Prisma — Performance de DB

```typescript
// ✅ Singleton do Prisma (evita múltiplas conexões em dev)
// src/lib/db/index.ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const db =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === "development" ? ["query", "error"] : ["error"],
  });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = db;

// ✅ Select explícito — nunca SELECT *
const user = await db.user.findUnique({
  where: { id },
  select: {
    id: true,
    name: true,
    email: true,
    // NÃO incluir: password, tokens, dados sensíveis
  },
});

// ✅ Paginação com cursor (tabelas grandes)
const payments = await db.payment.findMany({
  take: 20,
  skip: cursor ? 1 : 0,
  cursor: cursor ? { id: cursor } : undefined,
  orderBy: { createdAt: "desc" },
});
```
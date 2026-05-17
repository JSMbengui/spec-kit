Frontend – Next.js 14+ (App Router)

Tecnologias Recomendadas
Camada,Tecnologia
Framework,Next.js 14+ (App Router)
Linguagem,TypeScript (strict mode)
UI Library,Shadcn/ui + Tailwind CSS + Radix UI
State Management,Zustand
Formulários,React Hook Form + Zod
Tabelas,TanStack Table (React Table)
Gráficos,Recharts ou Tremor
Autenticação,custom JWT + HttpOnly cookies
Notificações,Sonner (toast) + React Hot Toast


Estrutura de Pastas Recomendada (App Router)
src/
├── app/                          # App Router
│   ├── (auth)/                   # Rotas de autenticação
│   ├── (dashboard)/              # Dashboard protegido
│   ├── contracts/
│   ├── invoices/
│   ├── entities/
│   └── api/                      # Route Handlers (se necessário)
│
├── components/                   # Componentes reutilizáveis
│   ├── ui/                       # Componentes shadcn
│   ├── layout/
│   ├── dashboard/                # Widgets do dashboard
│   ├── forms/
│   └── tables/
│
├── features/                     # Feature-based organization (recomendado)
│   ├── invoices/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── api/
│   │   └── types.ts
│   ├── contracts/
│   └── alerts/
│
├── lib/                          # Utilitários, axios instance, validators
├── hooks/                        # Custom hooks
├── stores/                       # Zustand stores
├── types/                        # Tipos globais
└── constants/


Estratégia de Testes no Frontend (Next.js)
Tipo de Teste,Ferramenta,Objetivo
Unit Tests,Jest + React Testing Library,Testar componentes isolados e hooks
Component Tests,Jest + RTL,Testar renderização e interações
Integration Tests,Jest + MSW (Mock Service Worker),Testar fluxos com API mockada
E2E Tests,Playwright ou Cypress,Testar fluxos completos do utilizador


Foco principal de testes no Frontend:
1. Fluxo completo de submissão de fatura
2. Validações de formulários (Zod)
3. Estados de loading, erro e sucesso
4. Responsividade
5. Permissões por perfil (RBAC no frontend)
6. Dashboard (KPIs, Funil, Alertas de Urgência)


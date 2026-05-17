Descrição Geral do Backend – SIGDC-UCP
O backend do SIGDC-UCP será desenvolvido em NestJS, seguindo os princípios da Clean Architecture (também conhecida como Arquitetura Hexagonal ou Ports & Adapters). Esta abordagem garante alta manutenibilidade, testabilidade, independência de frameworks e facilidade de evolução futura.

1. Estrutura de Pastas (Clean Architecture)
src/
├── common/                     # Utilitários, decorators, filters, guards, interceptors
├── config/                     # Configurações (env, typeorm, jwt, mailer, etc.)
├── core/                       # Camada mais interna (independente)
│   ├── domain/                 # Entidades de domínio puras + Value Objects
│   ├── usecases/               # Casos de uso (Application Services)
│   ├── ports/                  # Interfaces (Ports) - contratos com o mundo exterior
│   └── exceptions/             # Exceções de domínio
│
├── infrastructure/             # Implementações concretas
│   ├── persistence/            # TypeORM - Repositories, Entities, Migrations
│   ├── external/               # Integrações externas (email, storage, scheduler, etc.)
│   └── config/                 # Configurações específicas
│
├── modules/                    # Módulos organizados por domínio de negócio
│   ├── auth/
│   ├── users/
│   ├── entities/               # Entidades Contratantes (UC-01)
│   ├── contracts/              # Contratos + Adendas (UC-02, UC-03)
│   ├── invoices/               # Faturas + Workflow (UC-06 a UC-10)
│   ├── audit/                  # Trilha de Auditoria (UC-A05, UC-11)
│   ├── notifications/          # Notificações e Emails (UC-13)
│   ├── dashboard/              # KPIs e gráficos (UC-16 a UC-18)
│   └── alerts/                 # Alertas de estagnação e urgência (UC-12, UC-18)
│
├── shared/                     # Módulos reutilizáveis (guards, decorators, etc.)
└── main.ts


2. Camadas Principais (Clean Architecture)
  2.1 Domain Layer (Mais interna)
    2.1.1 Contém as regras de negócio puras.
    2.1.2 Não depende de frameworks nem de banco de dados.
    2.1.3 Inclui:
      2.1.3.1 Entidades de domínio (Contract, Invoice, Entity, Adenda, etc.)
      2.1.3.2 Value Objects (Nif, Iban, Money, ContractStatus, etc.)
      2.1.3.3 Domain Events (InvoiceRejected, InvoicePaid, ContractNearExpiry)
      2.1.3.4 Exceções de domínio (InsufficientBalanceException, InvalidIbanException)
  2.2  Application Layer (Use Cases)
    2.2.1 Contém os Casos de Uso (UCs) que implementaste.
    2.2.2 Exemplos:
      2.2.2.1 CreateContractUseCase
      2.2.2.2 SubmitInvoiceUseCase
      2.2.2.3 EmitTechnicalOpinionUseCase
      2.2.2.4 ApproveByProcurementUseCase
      2.2.2.5 RegisterPaymentUseCase
      2.2.2.6 CalculateRealBalanceUseCase
      2.2.2.7 GenerateUrgencyAlertsUseCase

Cada Use Case recebe inputs via DTOs e retorna outputs bem definidos.
  2.3 Infrastructure Layer
    2.3.1 Implementações concretas das interfaces definidas no core/ports.
    2.3.2 Inclui:
      2.3.2.1 TypeORM Entities (mapeamento das entidades de domínio)
      2.3.2.2 Repositories (implementação das interfaces ContractRepository, InvoiceRepository, etc.)
      2.3.2.3 MailerService (envio de emails)
      2.3.2.4 Scheduler (jobs para alertas de 48h, vigência, etc.)
      2.3.2.5 JWT Strategy, 2FA Service, etc.


3. Tecnologias e Bibliotecas Recomendadas
Camada / Função,Tecnologia / Biblioteca
Framework,NestJS
ORM,TypeORM
Banco de Dados,PostgreSQL (recomendado)
Autenticação,Passport + JWT + 2FA (TOTP + SMS)
Validação,class-validator + class-transformer
Emails,Nodemailer + Handlebars (templates)
Scheduler,@nestjs/schedule (cron jobs)
Documentação API,Swagger (OpenAPI)
Testes,Jest + Supertest
Logging,Winston ou Pino
Configuração,@nestjs/config + Joi


4. Princípios Importantes a Seguir
  4.1 Dependency Rule — Dependências só apontam para dentro (Domain → Application → Infrastructure).
  4.2 Repository Pattern — Abstrair o acesso a dados através de interfaces.
  4.3 Domain Events — Usar eventos para comunicação entre módulos (ex: InvoicePaid → atualiza saldo).
  4.4 Imutabilidade dos Logs — A trilha de auditoria deve ser append-only.
  4.5 Transações — Usar transações do TypeORM em operações críticas (ex: registo de pagamento + atualização de saldo).


5. Módulos Principais Recomendados
  5.1 auth → Autenticação + 2FA (UC-A01)
  5.2 users → Gestão de utilizadores e RBAC
  5.3 entities → Entidades Contratantes (UC-01)
  5.4 contracts → Contratos + Adendas + Saldo Real (UC-02, UC-03, UC-05)
  5.5 invoices → Workflow completo de faturas (UC-06 a UC-10)
  5.6 audit → Trilha de Auditoria imutável (UC-A05, UC-11)
  5.7 notifications → Emails e notificações (UC-13)
  5.8 alerts → Alertas de estagnação e urgência (UC-12, UC-18)
  5.9 dashboard → KPIs, Funil, Alertas de Urgência (UC-16 a UC-18)


6. Níveis de Testes:
Tipo de Teste,Ferramenta,Cobertura Esperada,Objetivo
Unit Tests,Jest,Alta,"Testar Use Cases, Domain Services e regras de negócio isoladamente"
Integration Tests,Jest + Supertest,Média,Testar controllers + repositories + use cases juntos
E2E Tests,Jest + Supertest,Baixa,Testar fluxos completos (ex: submeter fatura até pagamento)
Database Tests,TypeORM + Test Containers,Média,Testar migrações e queries complexas

Principais Regras de Testes:
  6.1 Testar primeiro os Use Cases (camada mais importante)
  6.2 Todos os Use Cases devem ter testes unitários
  6.3 Testes de domínio devem ser independentes de banco de dados
  6.4 Usar mocks para portas (ContractRepository, MailerService, etc.)
  6.5 Testar todas as regras críticas:
  6.6 Validação de NIF e IBAN
  6.7 Cálculo de saldo real
  6.8 Bloqueio quando saldo insuficiente
  6.9 Trava de todos os pareceres favoráveis
  6.10 Alertas de 48h e vigência




# Infrastructure Layer — Repositories, ORM, Adapters, Jobs

A Infrastructure implementa as interfaces definidas no Domain/Application.
Conhece TypeORM/Prisma, HTTP clients, ficheiros, Redis, etc.
Nunca é importada directamente pela Presentation — só via DI.

---

## ORM Entities (TypeORM) — Separadas das Domain Entities

```typescript
// modules/users/infrastructure/persistence/user.orm-entity.ts
import {
  Column,
  CreateDateColumn,
  Entity,
  PrimaryColumn,
  UpdateDateColumn,
} from 'typeorm';
import { UserRole } from '../../domain/entities/user.entity';

// Esta classe é apenas um mapeamento de tabela — zero lógica de negócio
@Entity('users')
export class UserOrmEntity {
  @PrimaryColumn('uuid')
  id: string;

  @Column({ unique: true, length: 255 })
  email: string;

  @Column({ length: 100 })
  name: string;

  @Column({ type: 'enum', enum: UserRole, default: UserRole.Customer })
  role: UserRole;

  @Column({ default: true })
  isActive: boolean;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

---

## Repository — TypeORM

```typescript
// modules/users/infrastructure/repositories/user.typeorm-repository.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { IUserRepository } from '../../domain/interfaces/user-repository.interface';
import { User } from '../../domain/entities/user.entity';
import { UserId } from '../../domain/value-objects/user-id.vo';
import { Email } from '../../domain/value-objects/email.vo';
import { UserOrmEntity } from '../persistence/user.orm-entity';
import { UserMapper } from '../../application/mappers/user.mapper';
import type {
  PaginationOptions,
  PaginatedResult,
} from '@/shared/domain/value-objects/pagination.vo';

@Injectable()
export class UserTypeOrmRepository implements IUserRepository {
  constructor(
    @InjectRepository(UserOrmEntity)
    private readonly repo: Repository<UserOrmEntity>,
  ) {}

  async findById(id: UserId): Promise<User | null> {
    const orm = await this.repo.findOne({ where: { id: id.value } });
    return orm ? UserMapper.toDomain(orm) : null;
  }

  async findByEmail(email: Email): Promise<User | null> {
    const orm = await this.repo.findOne({ where: { email: email.value } });
    return orm ? UserMapper.toDomain(orm) : null;
  }

  async findAll(options: PaginationOptions): Promise<PaginatedResult<User>> {
    const [orms, total] = await this.repo.findAndCount({
      where: { isActive: true },
      order: { [options.orderBy ?? 'createdAt']: options.order ?? 'DESC' },
      skip: (options.page - 1) * options.limit,
      take: options.limit,
    });

    return {
      data: orms.map(UserMapper.toDomain),
      total,
      page: options.page,
      totalPages: Math.ceil(total / options.limit),
    };
  }

  async save(user: User): Promise<void> {
    const orm = UserMapper.toOrm(user);
    await this.repo.save(orm);
  }

  async delete(id: UserId): Promise<void> {
    await this.repo.delete({ id: id.value });
  }

  async existsByEmail(email: Email): Promise<boolean> {
    return this.repo.exists({ where: { email: email.value } });
  }
}
```

---

## Repository — Prisma (alternativa)

```typescript
// modules/users/infrastructure/repositories/user.prisma-repository.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '@/shared/infrastructure/database/prisma.service';
import { IUserRepository } from '../../domain/interfaces/user-repository.interface';
import { User } from '../../domain/entities/user.entity';
import { UserId } from '../../domain/value-objects/user-id.vo';
import { Email } from '../../domain/value-objects/email.vo';
import { UserMapper } from '../../application/mappers/user.mapper';

@Injectable()
export class UserPrismaRepository implements IUserRepository {
  constructor(private readonly prisma: PrismaService) {}

  async findById(id: UserId): Promise<User | null> {
    const record = await this.prisma.user.findUnique({
      where: { id: id.value },
    });
    return record ? UserMapper.toDomain(record) : null;
  }

  async save(user: User): Promise<void> {
    const data = UserMapper.toOrm(user);
    await this.prisma.user.upsert({
      where: { id: data.id },
      create: data as any,
      update: data,
    });
  }

  async existsByEmail(email: Email): Promise<boolean> {
    const count = await this.prisma.user.count({
      where: { email: email.value },
    });
    return count > 0;
  }

  // ... outros métodos
}

// shared/infrastructure/database/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  async onModuleInit(): Promise<void> {
    await this.$connect();
  }
  async onModuleDestroy(): Promise<void> {
    await this.$disconnect();
  }
}
```

---

## Regra — Um Repositório por Agregado

Um repositório Prisma/TypeORM nunca deve persistir mais do que um agregado. Se um ficheiro
tem secções comentadas do tipo `// ─── Assembly ───`, `// ─── Documents ───`, isso é sinal de que
está a misturar agregados que deviam ter repositórios próprios.

```typescript
// ❌ Um único repositório a fazer persistência de 3 agregados
@Injectable()
export class PrismaAssemblyRepository implements IAssemblyRepository {
  async create(assembly: Assembly) { ... }
  async addDocument(document: AssemblyDocument) { ... }   // agregado diferente
  async saveResult(result: AssemblyResult) { ... }        // agregado diferente
}

// ✅ Um repositório por agregado, cada um com a sua interface
@Injectable()
export class PrismaAssemblyRepository implements IAssemblyRepository {
  async create(assembly: Assembly): Promise<Assembly> { ... }
  async findById(id: string): Promise<Assembly | null> { ... }
}

@Injectable()
export class PrismaAssemblyDocumentRepository implements IAssemblyDocumentRepository {
  async addDocument(document: AssemblyDocument): Promise<AssemblyDocument> { ... }
  async findActiveDocument(assemblyId: string): Promise<AssemblyDocument | null> { ... }
}

@Injectable()
export class PrismaAssemblyResultRepository implements IAssemblyResultRepository {
  async saveResult(result: AssemblyResult): Promise<AssemblyResult> { ... }
}
```

**Excepção:** só faz sentido manter tudo num repositório quando há consistência transacional
obrigatória entre os agregados (ex: criar `Assembly` + primeiro `AssemblyDocument` na mesma
transação Prisma com `$transaction`). Mesmo nesse caso, expõe um único método transacional
explícito (`createWithFirstDocument()`), não os CRUDs completos dos outros agregados.

---

## Regra — Mappers de Enum Centralizados

Conversão de enum domínio ↔ Prisma/TypeORM nunca deve ser feita inline com ternários encadeados
dentro de cada método do repositório.

```typescript
// ❌ Ternário encadeado repetido em create(), update() e em qualquer outro método
const status =
  assembly.status.toUpperCase() === 'FINISHED'
    ? AssemblyStatusPrisma.FINISHED
    : assembly.status.toUpperCase() === 'ONGOING'
      ? AssemblyStatusPrisma.ONGOING
      : AssemblyStatusPrisma.PLANNED;

// ✅ Mapper de enum num ficheiro dedicado, testável isoladamente
// modules/assembly/infrastructure/mappers/assembly-status.mapper.ts
import { AssemblyStatus as AssemblyStatusPrisma } from '@prisma/client';
import { Assembly } from '../../domain/entities/assembly.entity';

const TO_PRISMA: Record<Assembly['status'], AssemblyStatusPrisma> = {
  planned: AssemblyStatusPrisma.PLANNED,
  ongoing: AssemblyStatusPrisma.ONGOING,
  finished: AssemblyStatusPrisma.FINISHED,
};

export function toPrismaAssemblyStatus(
  status: Assembly['status'],
): AssemblyStatusPrisma {
  return TO_PRISMA[status] ?? AssemblyStatusPrisma.PLANNED;
}

export function toDomainAssemblyStatus(
  status: AssemblyStatusPrisma,
): Assembly['status'] {
  return status.toLowerCase() as Assembly['status'];
}
```

O repositório passa a só chamar a função:

```typescript
const row = await this.prisma.assembly.create({
  data: { ...rest, status: toPrismaAssemblyStatus(assembly.status) },
});
```

---

## Regra — Mappers Privados Saem do Repositório

Qualquer método `toDomain` / `toXxxDomain` dentro de um repositório que não dependa de
`this.prisma` (é uma função pura de transformação) deve sair para um ficheiro de mapper dedicado
em `infrastructure/.../mappers/`, da mesma forma que `UserMapper` já existe em `application/mappers/`
para Entity ↔ DTO. Isto torna o mapper testável sem mockar o Prisma e reduz o tamanho do
repositório a apenas chamadas de I/O.

```
modules/assembly/infrastructure/
  repositories/
    prisma-assembly.repository.ts          # só I/O — sem lógica de transformação
    prisma-assembly-document.repository.ts
    prisma-assembly-result.repository.ts
  mappers/
    assembly.mapper.ts                     # toDomain / toPrisma
    assembly-document.mapper.ts
    assembly-result.mapper.ts
    assembly-status.mapper.ts              # enum mapper isolado
```

---

## Regra — Tratamento de Erros do Prisma no Repositório

Operações que podem falhar por violação de unicidade ou registo inexistente
(`update`, `delete` por id) devem capturar `PrismaClientKnownRequestError` e traduzir para um
erro de domínio conhecido — nunca deixar a exceção genérica do Prisma subir até ao controller.

```typescript
import { Prisma } from '@prisma/client';
import { AssemblyNotFoundError } from '../../domain/errors/assembly-not-found.error';

async delete(id: string): Promise<void> {
  try {
    await this.prisma.assembly.delete({ where: { id } });
  } catch (err) {
    if (
      err instanceof Prisma.PrismaClientKnownRequestError &&
      err.code === 'P2025' // registo a eliminar/actualizar não existe
    ) {
      throw new AssemblyNotFoundError(id);
    }
    throw err;
  }
}
```

Códigos mais comuns a tratar: `P2025` (registo não encontrado em update/delete), `P2002`
(violação de unicidade → mapear para um erro `*AlreadyExistsError`), `P2003` (violação de
foreign key). Ver `references/error-handling.md` para o mapeamento equivalente no
`GlobalExceptionFilter`.

---

## Regra — Cliente HTTP de Integração Externa Separado dos Serviços de Domínio

Quando o parceiro externo expõe várias operações de negócio (ex: AGT Facturação Electrónica
expõe `solicitarSerie`, `registarFactura`, `listarSeries`, `obterEstado`, `consultarFactura`,
`listarFacturas`, `validarDocumento` — 7 operações), **nunca criar um único `@Injectable()`
com todos os métodos**. É a mesma violação de Single Responsibility da regra "um repositório
por agregado" acima — aqui são vários casos de uso externos acumulados num único serviço.

```typescript
// ❌ Deus-serviço: 7 operações de negócio + cliente HTTP genérico no mesmo ficheiro
@Injectable()
export class AgtFeService {
  async solicitarSerie(input) { ... }
  async registarFactura(input) { ... }
  async listarSeries(input) { ... }
  async obterEstado(input) { ... }
  async consultarFactura(input) { ... }
  async listarFacturas(input) { ... }
  async validarDocumento(input) { ... }
  private async post<T>(url, body) { ... } // transporte escondido aqui
}
```

```typescript
// ✅ Cliente HTTP genérico (transporte) + serviços agrupados por sub-domínio de negócio
// http/agt-http-client.ts — só sabe falar HTTP com a AGT, zero regra de negócio
@Injectable()
export class AgtHttpClient {
  async post<T>(path: string, body: unknown): Promise<T> { ... }
}

// services/agt-serie.service.ts
@Injectable()
export class AgtSerieService {
  constructor(private readonly http: AgtHttpClient) {}
  async solicitarSerie(input: SolicitarSerieInput): Promise<AgtSolicitarSerieResponse> { ... }
  async listarSeries(input: ListarSeriesInput): Promise<AgtListarSeriesResponse> { ... }
}

// services/agt-factura.service.ts, services/agt-estado.service.ts,
// services/agt-validacao.service.ts — mesmo padrão, cada um focado no seu sub-domínio
```

Critério de divisão: agrupar pelo **agregado/sub-domínio de negócio que a operação afecta**
(série, factura, estado, validação), nunca pelo facto de "todas falam com o mesmo parceiro" —
falar com o parceiro é responsabilidade exclusiva do `*HttpClient`.

**Configuração de rede (TLS, timeout) nunca hardcoded como "temporário":**

```typescript
// ❌ Comentário admite que é debug, mas fica hardcoded
httpsAgent: new https.Agent({
  minVersion: 'TLSv1.2',
  maxVersion: 'TLSv1.2', // força só 1.2 para testar
}),

// ✅ Configurável por ambiente, com default seguro
httpsAgent: new https.Agent({
  minVersion: this.config.get<string>('AGT_FE_TLS_MIN_VERSION') ?? 'TLSv1.2',
  // sem maxVersion fixo, a não ser que exista um ticket/incidente documentado
}),
```

Ver `references/external-integrations.md` para o detalhe completo (estrutura de pastas,
geração de envelope de submissão, mapeamento de erros de rede, checklist).

---

## Adapters — Serviços Externos (Ports & Adapters)

```typescript
// modules/payments/infrastructure/adapters/stripe-payment.adapter.ts
import { Injectable } from '@nestjs/common';
import Stripe from 'stripe';
import { ConfigService } from '@nestjs/config';
import {
  IPaymentGateway,
  ProcessPaymentRequest,
  ProcessPaymentResponse,
} from '../../domain/interfaces/payment-gateway.interface';

@Injectable()
export class StripePaymentAdapter implements IPaymentGateway {
  private readonly stripe: Stripe;

  constructor(private readonly config: ConfigService) {
    this.stripe = new Stripe(config.getOrThrow('STRIPE_SECRET_KEY'), {
      apiVersion: '2024-06-20',
    });
  }

  async processPayment(
    request: ProcessPaymentRequest,
  ): Promise<ProcessPaymentResponse> {
    const intent = await this.stripe.paymentIntents.create({
      amount: Math.round(request.amount * 100), // Stripe usa cêntimos
      currency: request.currency.toLowerCase(),
      metadata: { reference: request.reference, ...request.metadata },
      confirm: true,
      automatic_payment_methods: { enabled: true, allow_redirects: 'never' },
    });

    return {
      transactionId: intent.id,
      status: this.mapStripeStatus(intent.status),
      processedAt: new Date(intent.created * 1000),
    };
  }

  async refundPayment(transactionId: string, amount: number): Promise<void> {
    await this.stripe.refunds.create({
      payment_intent: transactionId,
      amount: Math.round(amount * 100),
    });
  }

  async getTransactionStatus(transactionId: string): Promise<string> {
    const intent = await this.stripe.paymentIntents.retrieve(transactionId);
    return this.mapStripeStatus(intent.status);
  }

  private mapStripeStatus(status: string): 'SUCCESS' | 'PENDING' | 'FAILED' {
    const map: Record<string, 'SUCCESS' | 'PENDING' | 'FAILED'> = {
      succeeded: 'SUCCESS',
      processing: 'PENDING',
      canceled: 'FAILED',
      requires_payment_method: 'FAILED',
    };
    return map[status] ?? 'PENDING';
  }
}
```

---

## Background Jobs — BullMQ

```typescript
// modules/sync/infrastructure/jobs/client-sync.processor.ts
import { Processor, WorkerHost, OnWorkerEvent } from '@nestjs/bullmq';
import { Logger } from '@nestjs/common';
import { Job } from 'bullmq';
import { SyncClientsUseCase } from '../../application/use-cases/sync-clients.use-case';

export interface SyncJobData {
  batchSize: number;
  offset: number;
  jobRunId: string;
}

@Processor('client-sync')
export class ClientSyncProcessor extends WorkerHost {
  private readonly logger = new Logger(ClientSyncProcessor.name);

  constructor(private readonly syncClientsUseCase: SyncClientsUseCase) {
    super();
  }

  async process(job: Job<SyncJobData>): Promise<void> {
    const { batchSize, offset, jobRunId } = job.data;
    this.logger.log(`Processing batch offset=${offset}, size=${batchSize}`);

    await job.updateProgress(0);

    const result = await this.syncClientsUseCase.execute({
      batchSize,
      offset,
      jobRunId,
    });

    await job.updateProgress(100);
    this.logger.log(`Batch completed: ${result.processed} records processed`);
  }

  @OnWorkerEvent('failed')
  onFailed(job: Job<SyncJobData>, error: Error): void {
    this.logger.error(
      { jobId: job.id, error: error.message, attempt: job.attemptsMade },
      'Sync job failed',
    );
  }
}

// Cron job para agendar
// modules/sync/infrastructure/jobs/client-sync.scheduler.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';

@Injectable()
export class ClientSyncScheduler {
  private readonly logger = new Logger(ClientSyncScheduler.name);

  constructor(@InjectQueue('client-sync') private readonly queue: Queue) {}

  @Cron(CronExpression.EVERY_DAY_AT_2AM, {
    name: 'daily-client-sync',
    timeZone: 'Africa/Luanda',
  })
  async scheduleDailySync(): Promise<void> {
    this.logger.log('Scheduling daily client sync');
    await this.queue.add(
      'sync-batch',
      { batchSize: 500, offset: 0, jobRunId: randomUUID() },
      {
        attempts: 3,
        backoff: { type: 'exponential', delay: 5000 },
        removeOnComplete: { count: 10 },
        removeOnFail: { count: 50 },
      },
    );
  }
}
```

---

## Database Config — TypeORM

```typescript
// config/database.config.ts
import { registerAs } from '@nestjs/config';
import { TypeOrmModuleOptions } from '@nestjs/typeorm';

export default registerAs(
  'database',
  (): TypeOrmModuleOptions => ({
    type: 'postgres',
    url: process.env.DATABASE_URL,
    autoLoadEntities: true, // carrega entities registadas nos módulos
    synchronize: false, // NUNCA true em produção
    logging:
      process.env.NODE_ENV === 'development' ? ['query', 'error'] : ['error'],
    extra: {
      max: 20, // pool size
      idleTimeoutMillis: 30_000,
      connectionTimeoutMillis: 2_000,
    },
    migrations: ['dist/shared/infrastructure/database/migrations/*.js'],
    migrationsRun: true, // corre migrations automaticamente no boot
  }),
);
```

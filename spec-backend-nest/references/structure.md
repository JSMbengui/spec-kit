# Estrutura de Pastas, Módulos e Dependency Injection

## Estrutura Completa de Projecto

```
my-api/
├── src/
│   ├── modules/                          # Feature modules — um por domínio
│   │   ├── users/
│   │   │   ├── domain/
│   │   │   │   ├── entities/
│   │   │   │   │   └── user.entity.ts
│   │   │   │   ├── value-objects/
│   │   │   │   │   ├── user-id.vo.ts
│   │   │   │   │   └── email.vo.ts
│   │   │   │   ├── events/
│   │   │   │   │   ├── user-created.event.ts
│   │   │   │   │   └── user-deactivated.event.ts
│   │   │   │   ├── errors/
│   │   │   │   │   ├── user-not-found.error.ts
│   │   │   │   │   └── user-already-exists.error.ts
│   │   │   │   └── interfaces/
│   │   │   │       └── user-repository.interface.ts
│   │   │   ├── application/
│   │   │   │   ├── use-cases/
│   │   │   │   │   ├── create-user/
│   │   │   │   │   │   ├── create-user.use-case.ts
│   │   │   │   │   │   └── create-user.use-case.spec.ts
│   │   │   │   │   ├── get-user/
│   │   │   │   │   │   └── get-user.use-case.ts
│   │   │   │   │   ├── update-user/
│   │   │   │   │   │   └── update-user.use-case.ts
│   │   │   │   │   └── deactivate-user/
│   │   │   │   │       └── deactivate-user.use-case.ts
│   │   │   │   ├── dtos/
│   │   │   │   │   ├── create-user.dto.ts
│   │   │   │   │   ├── update-user.dto.ts
│   │   │   │   │   └── user-response.dto.ts
│   │   │   │   └── mappers/
│   │   │   │       └── user.mapper.ts
│   │   │   ├── infrastructure/
│   │   │   │   ├── repositories/
│   │   │   │   │   └── user.typeorm-repository.ts
│   │   │   │   └── persistence/
│   │   │   │       ├── user.orm-entity.ts  (TypeORM entity com decorators)
│   │   │   │       └── user.schema.ts      (Prisma — se usar Prisma)
│   │   │   ├── presentation/
│   │   │   │   ├── controllers/
│   │   │   │   │   └── users.controller.ts
│   │   │   │   └── decorators/
│   │   │   │       └── current-user.decorator.ts
│   │   │   └── users.module.ts
│   │   │
│   │   ├── payments/
│   │   │   ├── domain/
│   │   │   │   ├── entities/
│   │   │   │   │   └── payment.entity.ts
│   │   │   │   ├── value-objects/
│   │   │   │   │   ├── money.vo.ts
│   │   │   │   │   └── payment-id.vo.ts
│   │   │   │   ├── errors/
│   │   │   │   │   ├── insufficient-funds.error.ts
│   │   │   │   │   └── payment-not-found.error.ts
│   │   │   │   └── interfaces/
│   │   │   │       ├── payment-repository.interface.ts
│   │   │   │       └── payment-gateway.interface.ts   # port para gateway externo
│   │   │   ├── application/
│   │   │   │   ├── use-cases/
│   │   │   │   │   ├── process-payment/
│   │   │   │   │   │   └── process-payment.use-case.ts
│   │   │   │   │   └── refund-payment/
│   │   │   │   │       └── refund-payment.use-case.ts
│   │   │   │   ├── dtos/
│   │   │   │   └── mappers/
│   │   │   ├── infrastructure/
│   │   │   │   ├── repositories/
│   │   │   │   ├── persistence/
│   │   │   │   └── adapters/
│   │   │   │       └── stripe-payment.adapter.ts     # implementa IPaymentGateway
│   │   │   ├── presentation/
│   │   │   │   └── controllers/
│   │   │   └── payments.module.ts
│   │   │
│   │   └── auth/
│   │       ├── domain/
│   │       ├── application/
│   │       │   └── use-cases/
│   │       │       ├── login/
│   │       │       └── refresh-token/
│   │       ├── infrastructure/
│   │       │   └── adapters/
│   │       │       └── bcrypt.adapter.ts
│   │       ├── presentation/
│   │       │   ├── controllers/
│   │       │   │   └── auth.controller.ts
│   │       │   └── strategies/
│   │       │       ├── jwt.strategy.ts
│   │       │       └── local.strategy.ts
│   │       └── auth.module.ts
│   │
│   ├── shared/                           # Código partilhado entre módulos
│   │   ├── domain/
│   │   │   ├── errors/
│   │   │   │   └── domain.error.ts       # Classe base de todos os erros de domínio
│   │   │   └── value-objects/
│   │   │       ├── pagination.vo.ts
│   │   │       └── unique-id.vo.ts
│   │   ├── application/
│   │   │   ├── use-case.interface.ts     # Interface base: execute(dto): Promise<R>
│   │   │   └── ports/
│   │   │       ├── event-bus.port.ts
│   │   │       └── logger.port.ts
│   │   ├── infrastructure/
│   │   │   ├── database/
│   │   │   │   ├── typeorm.config.ts
│   │   │   │   └── migrations/
│   │   │   ├── logger/
│   │   │   │   └── pino-logger.service.ts
│   │   │   └── events/
│   │   │       └── nestjs-event-bus.adapter.ts
│   │   └── presentation/
│   │       ├── filters/
│   │       │   └── global-exception.filter.ts
│   │       ├── interceptors/
│   │       │   ├── logging.interceptor.ts
│   │       │   └── transform-response.interceptor.ts
│   │       ├── guards/
│   │       │   ├── jwt-auth.guard.ts
│   │       │   └── roles.guard.ts
│   │       ├── pipes/
│   │       │   └── validation.pipe.ts
│   │       └── decorators/
│   │           ├── public.decorator.ts
│   │           └── roles.decorator.ts
│   │
│   ├── config/
│   │   ├── app.config.ts
│   │   ├── database.config.ts
│   │   └── env.validation.ts             # Joi/Zod schema de env vars
│   │
│   ├── app.module.ts
│   └── main.ts
│
├── test/
│   ├── e2e/
│   │   └── users.e2e-spec.ts
│   └── fixtures/
│       ├── user.fixture.ts
│       └── payment.fixture.ts
│
├── prisma/                               # Se usar Prisma
│   ├── schema.prisma
│   └── migrations/
│
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
│
├── .env.example
├── nest-cli.json
├── tsconfig.json
└── package.json
```

---

## Naming Conventions

```
Ficheiros de entities:        kebab-case.entity.ts         user.entity.ts
Ficheiros de value objects:   kebab-case.vo.ts             email.vo.ts
Ficheiros de use cases:       kebab-case.use-case.ts       create-user.use-case.ts
Ficheiros de DTOs:            kebab-case.dto.ts            create-user.dto.ts
Ficheiros de controllers:     kebab-case.controller.ts     users.controller.ts
Ficheiros de repositories:    kebab-case.repository.ts     user.typeorm-repository.ts
Ficheiros de ORM entities:    kebab-case.orm-entity.ts     user.orm-entity.ts
Ficheiros de erros:           kebab-case.error.ts          user-not-found.error.ts
Ficheiros de interfaces:      kebab-case.interface.ts      user-repository.interface.ts
Ficheiros de módulos:         kebab-case.module.ts         users.module.ts
Classes:                      PascalCase                   CreateUserUseCase
Interfaces:                   I + PascalCase               IUserRepository
DI Tokens:                    SCREAMING_SNAKE_CASE symbol  USER_REPOSITORY
```

---

## Module Wiring — Padrão de DI

```typescript
// modules/users/users.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

// DI Tokens — sempre symbols para evitar colisões
export const USER_REPOSITORY = Symbol('IUserRepository');
export const USER_EVENT_BUS = Symbol('IEventBus');

@Module({
  imports: [
    TypeOrmModule.forFeature([UserOrmEntity]),
    SharedModule, // logger, event bus, etc.
  ],
  controllers: [UsersController],
  providers: [
    // Use Cases — um provider por use case
    CreateUserUseCase,
    GetUserUseCase,
    UpdateUserUseCase,
    DeactivateUserUseCase,

    // Repository — injected via interface token
    {
      provide: USER_REPOSITORY,
      useClass: UserTypeOrmRepository,
    },
  ],
  exports: [
    // Exportar apenas o que outros módulos precisam
    GetUserUseCase,
  ],
})
export class UsersModule {}
```

---

## Injecção com Tokens de Interface

```typescript
// Definir o token no módulo ou num ficheiro de tokens
export const USER_REPOSITORY = Symbol('IUserRepository');

// Use Case usa @Inject() com o token
@Injectable()
export class CreateUserUseCase {
  constructor(
    @Inject(USER_REPOSITORY)
    private readonly userRepository: IUserRepository,
  ) {}
}

// Repository implementa a interface
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
}
```

---

## Shared Module

```typescript
// shared/shared.module.ts
@Module({
  imports: [LoggerModule, EventEmitterModule.forRoot()],
  providers: [
    PinoLoggerService,
    { provide: EVENT_BUS, useClass: NestJsEventBusAdapter },
  ],
  exports: [
    PinoLoggerService,
    { provide: EVENT_BUS, useClass: NestJsEventBusAdapter },
  ],
})
export class SharedModule {}
```

---

## main.ts — Bootstrap Completo

```typescript
// src/main.ts
import { NestFactory, Reflector } from '@nestjs/core';
import {
  ValidationPipe,
  ClassSerializerInterceptor,
  VersioningType,
} from '@nestjs/common';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import helmet from 'helmet';
import compression from 'compression';
import { AppModule } from './app.module';
import { GlobalExceptionFilter } from './shared/presentation/filters/global-exception.filter';
import { LoggingInterceptor } from './shared/presentation/interceptors/logging.interceptor';
import { TransformResponseInterceptor } from './shared/presentation/interceptors/transform-response.interceptor';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    bufferLogs: true, // aguarda logger customizado
  });

  // Segurança
  app.use(helmet());
  app.use(compression());
  app.enableCors({
    origin: process.env.ALLOWED_ORIGINS?.split(',') ?? [
      'http://localhost:3000',
    ],
    credentials: true,
  });

  // Prefixo e versionamento global
  app.setGlobalPrefix('api');
  app.enableVersioning({ type: VersioningType.URI, defaultVersion: '1' });

  // Pipes globais — validação e transformação de DTOs
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true, // remove campos não decorados
      forbidNonWhitelisted: true, // erro se campos extra
      transform: true, // transforma para o tipo do DTO
      transformOptions: { enableImplicitConversion: true },
    }),
  );

  // Filtros e interceptors globais
  const reflector = app.get(Reflector);
  app.useGlobalFilters(new GlobalExceptionFilter());
  app.useGlobalInterceptors(
    new LoggingInterceptor(),
    new ClassSerializerInterceptor(reflector),
    new TransformResponseInterceptor(),
  );

  // Swagger — apenas em não-produção
  if (process.env.NODE_ENV !== 'production') {
    const config = new DocumentBuilder()
      .setTitle('BPC API')
      .setDescription('Banking API — Clean Architecture')
      .setVersion('1.0')
      .addBearerAuth()
      .build();
    const document = SwaggerModule.createDocument(app, config);
    SwaggerModule.setup('api/docs', app, document, {
      swaggerOptions: { persistAuthorization: true },
    });
  }

  // Graceful shutdown
  app.enableShutdownHooks();

  const port = process.env.PORT ?? 3000;
  await app.listen(port);
  console.log(`Application running on: http://localhost:${port}/api`);
}

bootstrap();
```

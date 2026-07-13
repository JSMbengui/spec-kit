---
name: nestjs-clean-architecture
description: >
  Guia de padrões de organização e arquitectura para projectos NestJS com Clean Architecture.
  Usar SEMPRE que o utilizador pedir para: organizar, estruturar, refactorizar ou rever um projecto
  NestJS; criar módulos, controllers, services, repositories, DTOs, entities, guards, interceptors,
  filters ou pipes em NestJS; aplicar Clean Architecture, SOLID, DDD, CQRS ou boas práticas de
  grandes empresas em NestJS/Node.js; decidir onde colocar código (domain vs application vs
  infrastructure vs presentation); rever ou melhorar código NestJS existente; configurar ESLint,
  Prettier, Jest, Swagger, Docker ou CI/CD em projectos NestJS; ou sempre que o utilizador mencionar
  "nestjs", "nest.js", "clean architecture backend", "domain layer", "use case", "repository pattern",
  "dependency injection node", "backend typescript" ou perguntar sobre estrutura de APIs REST/GraphQL
  em Node.js com TypeScript.
---

# NestJS Clean Architecture Skill

Este skill define os padrões exactos a seguir ao organizar, criar ou rever qualquer projecto NestJS
com Clean Architecture. Aplica estes padrões de forma consistente e proactiva — não esperes que o
utilizador os peça explicitamente.

## Como usar este skill

1. Lê este SKILL.md completo antes de qualquer resposta sobre estrutura NestJS
2. Para detalhes de uma área específica, lê o ficheiro de referência correspondente:
   - Estrutura de pastas e módulos → `references/structure.md`
   - Domain Layer (entities, value objects, events) → `references/domain.md`
   - Application Layer (use cases, DTOs, interfaces) → `references/application.md`
   - Infrastructure Layer (repositories, ORM, externos) → `references/infrastructure.md`
   - Presentation Layer (controllers, guards, pipes) → `references/presentation.md`
   - Error handling, exceptions e filtros → `references/error-handling.md`
   - Testes (unit, integration, e2e) → `references/testing.md`
   - Tooling, CI/CD e configuração → `references/tooling.md`
   - Integrações com APIs externas (AGT, BNA, gateways) → `references/external-integrations.md`
3. Aplica sempre **todos** os padrões relevantes, mesmo quando o utilizador só pede "cria um service"

---

## As Quatro Camadas — Regra Absoluta de Dependências

```
┌─────────────────────────────────────────────────────────────┐
│                    PRESENTATION                              │
│   Controllers · Guards · Interceptors · Pipes · Filters      │
│   (HTTP in/out — conhece Application, nunca Infrastructure)  │
├─────────────────────────────────────────────────────────────┤
│                    APPLICATION                               │
│        Use Cases · DTOs · Interfaces de Repositório          │
│   (Orquestra — conhece Domain, nunca Infrastructure)         │
├─────────────────────────────────────────────────────────────┤
│                      DOMAIN                                  │
│   Entities · Value Objects · Domain Events · Domain Errors   │
│   (Núcleo puro — ZERO dependências externas)                 │
├─────────────────────────────────────────────────────────────┤
│                  INFRASTRUCTURE                              │
│  Repositories (TypeORM/Prisma) · APIs externas · Jobs · DB   │
│   (Implementa interfaces — conhece tudo, não é conhecido)    │
└─────────────────────────────────────────────────────────────┘
         Setas de dependência: sempre ↑ (para dentro/cima)
```

**Regra de ouro:** O Domain não importa nada. O Application importa só Domain.
A Infrastructure implementa interfaces do Application. A Presentation usa Application.

---

## Princípios Fundamentais (NUNCA violar)

### 1. Domain é sagrado — zero dependências externas

```typescript
// ❌ NUNCA no Domain
import { Column, Entity } from 'typeorm'; // ORM
import { IsEmail } from 'class-validator'; // validação HTTP
import { Injectable } from '@nestjs/common'; // framework
import { Repository } from 'typeorm'; // DB

// ✅ Domain apenas usa TypeScript puro
export class User {
  constructor(
    public readonly id: UserId,
    public readonly email: Email,
    private _name: string,
  ) {}
  changeName(name: string): void {
    /* lógica de negócio */
  }
}
```

### 2. Use Cases como unidade de comportamento de negócio

Cada caso de uso é uma classe com um único método `execute()`. Um use case = uma operação de negócio.

### 3. Interfaces no Application, implementações na Infrastructure

```typescript
// Application define o contrato
export interface IUserRepository {
  findById(id: UserId): Promise<User | null>;
  save(user: User): Promise<void>;
}

// Infrastructure implementa
@Injectable()
export class UserTypeOrmRepository implements IUserRepository { ... }
```

### 4. DTOs na fronteira — Entities nunca saem da Application

Controllers recebem DTOs → Use Cases retornam DTOs de resposta → nunca Entities.

### 5. Erros tipados, nunca strings

```typescript
// Domain define erros de negócio
export class UserNotFoundError extends DomainError {
  constructor(id: string) {
    super(`User ${id} not found`, 'USER_NOT_FOUND');
  }
}
// Infrastructure/Presentation mapeia para HTTP
```

### 6. Um repositório por agregado

Repositórios Prisma/TypeORM não devem misturar persistência de múltiplos agregados num único
ficheiro (ex: `Assembly`, `AssemblyDocument` e `AssemblyResult` em três classes separadas, não
numa só). Excepção: consistência transacional obrigatória entre agregados, e mesmo nesse caso
através de um método transacional explícito, não dos CRUDs completos misturados.

### 7. Enums mapeados num ficheiro dedicado

Conversão de enum domínio ↔ Prisma/TypeORM nunca em ternário encadeado inline dentro do
repositório — sempre numa função `toPrismaXxx()` / `toDomainXxx()` isolada e testável.

### 8. Erros do Prisma traduzidos no repositório

Métodos que podem lançar `Prisma.PrismaClientKnownRequestError` (`update`, `delete` por id,
criação com campo único) devem capturar e traduzir para o `DomainError` correspondente, nunca
deixar a exceção genérica do Prisma chegar ao controller.

### 9. Controllers nunca usam `@Res()` com try/catch manual

O controller retorna directamente o resultado do Use Case (`return this.useCase.execute(dto)`).
Erros lançados pelo Use Case sobem para o `GlobalExceptionFilter`, que já trata `DomainError`,
`HttpException` e `Prisma.PrismaClientKnownRequestError` de forma centralizada (ver
`references/error-handling.md`). Injectar `@Res() res` e construir a resposta/erro à mão
desliga o filtro global e o `ClassSerializerInterceptor`, e obriga a reimplementar o mapeamento
de erros em cada método.

### 10. Integração com APIs externas: cliente HTTP separado dos serviços de domínio

Um parceiro externo (AGT, BNA, gateway de pagamento) que expõe várias operações de negócio
nunca é servido por um único `@Injectable()` deus-serviço. Separa-se num `*HttpClient`
(transporte: HTTP, TLS, timeout) e em serviços agrupados por sub-domínio de negócio (ex:
`AgtSerieService`, `AgtFacturaService`). Erros de rede mapeiam-se por `error.code`, nunca por
`error.message.includes(...)`. Ver `references/external-integrations.md`.

---

## Decisões Rápidas (cheat-sheet)

| Situação                                    | Solução                             |
| ------------------------------------------- | ----------------------------------- |
| Regra de negócio                            | Método na Entity ou Domain Service  |
| Operação que envolve múltiplos repositórios | Use Case                            |
| Validação de input HTTP                     | DTO com class-validator             |
| Acesso a base de dados                      | Repository na Infrastructure        |
| Chamada a API externa                       | Port/Adapter na Infrastructure      |
| Autenticação                                | Guard na Presentation               |
| Logging de requests                         | Interceptor na Presentation         |
| Transformar resposta                        | Interceptor ou mapper no Use Case   |
| Rate limiting                               | Guard + Throttler                   |
| Background job                              | Job na Infrastructure, usa Use Case |
| Evento de domínio                           | DomainEvent + EventHandler          |
| Config e env vars                           | ConfigModule + Joi/Zod validation   |
| Conversão de enum domínio ↔ Prisma/TypeORM   | Mapper dedicado, nunca ternário inline no repositório |
| Persistência de múltiplos agregados         | Um repositório por agregado, nunca um repositório "gordo" |
| Erro de unicidade/registo não encontrado no Prisma | try/catch no repositório → DomainError tipado |
| Tratamento de erro/resposta HTTP no controller | Retorno directo + GlobalExceptionFilter, nunca `@Res()` + try/catch |
| Várias operações de um parceiro externo (AGT, BNA, gateway) | `*HttpClient` (transporte) + serviços por sub-domínio, nunca um deus-serviço |

---

## Estrutura de Pastas — Resumo

```
src/
  modules/                    # Feature modules — um por domínio de negócio
    users/
      domain/                 # Camada de domínio
        entities/             # User.entity.ts (TypeScript puro)
        value-objects/        # Email.vo.ts, UserId.vo.ts
        events/               # UserCreatedEvent.ts
        errors/               # UserNotFoundError.ts
        interfaces/           # IUserRepository.ts (contrato)
      application/            # Camada de aplicação
        use-cases/            # CreateUserUseCase.ts, GetUserUseCase.ts
        dtos/                 # CreateUserDto.ts, UserResponseDto.ts
        mappers/              # UserMapper.ts (entity → dto)
      infrastructure/         # Camada de infraestrutura
        repositories/         # UserTypeOrmRepository.ts
        persistence/          # User.orm-entity.ts, UserSchema.ts
      presentation/           # Camada de apresentação
        controllers/          # UsersController.ts
        decorators/           # @CurrentUser()
      users.module.ts         # Wiring de DI

  shared/                     # Partilhado entre módulos
    domain/
      errors/                 # DomainError base
      value-objects/          # Pagination, Money, etc.
    application/
      use-case.base.ts        # UseCase interface base
    infrastructure/
      database/               # TypeORM config, migrations
      logger/                 # Logger service
    presentation/
      filters/                # GlobalExceptionFilter
      interceptors/           # LoggingInterceptor, TransformInterceptor
      guards/                 # JwtAuthGuard, RolesGuard
      pipes/                  # ValidationPipe config
      decorators/             # @Public(), @Roles()

  config/                     # ConfigModule, env validation
  app.module.ts
  main.ts
```

> Para estrutura completa com exemplos → lê `references/structure.md`

---

## Padrão de Cada Tipo de Ficheiro

### Entity (Domain)

```typescript
// modules/users/domain/entities/user.entity.ts
export class User {
  private constructor(
    public readonly id: UserId,
    public readonly email: Email,
    private _name: string,
    private _isActive: boolean,
    public readonly createdAt: Date,
  ) {}

  static create(props: CreateUserProps): User {
    // Validação de negócio — não de HTTP
    if (!props.name || props.name.trim().length < 2) {
      throw new InvalidUserDataError('Name must have at least 2 characters');
    }
    return new User(
      UserId.generate(),
      Email.create(props.email),
      props.name.trim(),
      true,
      new Date(),
    );
  }

  get name() {
    return this._name;
  }
  get isActive() {
    return this._isActive;
  }

  deactivate(): void {
    if (!this._isActive) throw new UserAlreadyInactiveError(this.id.value);
    this._isActive = false;
  }
}
```

### Use Case (Application)

```typescript
// modules/users/application/use-cases/create-user.use-case.ts
@Injectable()
export class CreateUserUseCase {
  constructor(
    @Inject(IUserRepository) private readonly userRepository: IUserRepository,
    @Inject(IEventBus) private readonly eventBus: IEventBus,
  ) {}

  async execute(dto: CreateUserDto): Promise<UserResponseDto> {
    const existing = await this.userRepository.findByEmail(dto.email);
    if (existing) throw new UserAlreadyExistsError(dto.email);

    const user = User.create(dto);
    await this.userRepository.save(user);
    await this.eventBus.publish(new UserCreatedEvent(user.id.value));

    return UserMapper.toResponseDto(user);
  }
}
```

### Controller (Presentation)

```typescript
// modules/users/presentation/controllers/users.controller.ts
@ApiTags('users')
@Controller('users')
@UseGuards(JwtAuthGuard)
export class UsersController {
  constructor(
    private readonly createUser: CreateUserUseCase,
    private readonly getUser: GetUserUseCase,
  ) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @ApiOperation({ summary: 'Create a new user' })
  @ApiResponse({ status: 201, type: UserResponseDto })
  @ApiResponse({ status: 409, description: 'User already exists' })
  create(@Body() dto: CreateUserDto): Promise<UserResponseDto> {
    return this.createUser.execute(dto);
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get user by ID' })
  findOne(@Param('id', ParseUUIDPipe) id: string): Promise<UserResponseDto> {
    return this.getUser.execute(id);
  }
}
```

> Para padrões completos de cada camada → lê os ficheiros em `references/`

---

## O que NUNCA fazer em NestJS com Clean Architecture

```typescript
// ❌ Lógica de negócio no Controller
@Post()
async create(@Body() dto: CreateUserDto) {
  if (dto.age < 18) throw new BadRequestException("Too young"); // ERRADO — vai para Domain/UseCase
  return this.userService.create(dto);
}

// ❌ Acesso directo a DB no Service/UseCase
@Injectable()
export class UserService {
  constructor(@InjectRepository(UserEntity) private repo: Repository<UserEntity>) {} // ERRADO
}

// ❌ Entity de ORM usada no Domain
@Entity("users")                           // decorator TypeORM no Domain — ERRADO
export class User {
  @PrimaryGeneratedColumn("uuid") id: string;
}

// ❌ Retornar Entity directamente pelo Controller
findOne(): Promise<User> {                 // User é entity de domínio — ERRADO
  return this.getUser.execute(id);
}

// ❌ Throw de string genérica
throw new Error("User not found");        // usar UserNotFoundError — ERRADO

// ❌ new Service() dentro de outro Service
@Injectable()
export class OrderService {
  private userService = new UserService(); // ERRADO — usar DI
}

// ❌ any implícito
const data: any = await this.repo.find(); // ERRADO — tipar explicitamente

// ❌ console.log em produção
console.log("user created", user);        // usar Logger do NestJS — ERRADO

// ❌ Conversão de enum inline repetida em vários métodos do repositório
const status = dto.status.toUpperCase() === "FINISHED" ? StatusPrisma.FINISHED : ...; // ERRADO — extrair para mapper

// ❌ Repositório a persistir múltiplos agregados num único ficheiro
class PrismaAssemblyRepository {
  async create(assembly: Assembly) {}
  async addDocument(doc: AssemblyDocument) {}  // agregado diferente — ERRADO, repositório próprio
}

// ❌ Deixar erro do Prisma subir sem tradução
async delete(id: string) {
  await this.prisma.user.delete({ where: { id } }); // ERRADO — sem try/catch para P2025
}

// ❌ @Res() + try/catch manual no controller, reimplementando o GlobalExceptionFilter
@Get()
async list(@Res() res): Promise<void> {
  try {
    const result = await this.useCase.execute();
    return res.status(200).json({ success: true, data: result }); // ERRADO
  } catch (error) {
    return res.status(500).json({ success: false, message: error.message }); // ERRADO
  }
}

// ❌ Deus-serviço acumulando todas as operações de um parceiro externo
@Injectable()
export class AgtFeService {
  async solicitarSerie(input) {}
  async registarFactura(input) {}
  async obterEstado(input) {}      // 7 operações diferentes no mesmo ficheiro — ERRADO
  private async post<T>(url, body) {} // transporte HTTP escondido aqui — ERRADO
}

// ❌ Detectar erro de rede por string matching na mensagem
if (error.message?.includes('socket disconnected')) { ... } // ERRADO — usar error.code
```

---

## Referências disponíveis

| Ficheiro                       | Conteúdo                                                               |
| ------------------------------ | ---------------------------------------------------------------------- |
| `references/structure.md`      | Estrutura completa, naming, módulo wiring, DI tokens                   |
| `references/domain.md`         | Entities, Value Objects, Domain Events, Domain Errors, Domain Services |
| `references/application.md`    | Use Cases, DTOs, Mappers, interfaces de repositório e ports            |
| `references/infrastructure.md` | Repositories TypeORM/Prisma, ORM entities, adapters externos, jobs     |
| `references/presentation.md`   | Controllers, Guards, Interceptors, Pipes, Decorators, Swagger          |
| `references/error-handling.md` | DomainError hierarchy, GlobalExceptionFilter, RFC 7807, retry          |
| `references/testing.md`        | Unit (Jest), Integration (TestingModule), E2E (Supertest), fixtures    |
| `references/tooling.md`        | ESLint, Prettier, tsconfig, Docker, CI/CD GitHub Actions               |
| `references/external-integrations.md` | Integrações com APIs externas (AGT, BNA, gateways): estrutura, envelope de submissão, erros de rede |

# Application Layer — Use Cases, DTOs e Mappers

A camada de Application orquestra o Domain. Conhece o Domain mas não a Infrastructure
(depende de interfaces, não de implementações concretas).

---

## Use Case — Padrão Base

```typescript
// shared/application/use-case.interface.ts
export interface UseCase<TInput, TOutput> {
  execute(input: TInput): Promise<TOutput>;
}
```

---

## Use Cases — Exemplos Completos

### Create (comando com efeito lateral)

```typescript
// modules/users/application/use-cases/create-user/create-user.use-case.ts
import { Injectable, Inject } from '@nestjs/common';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { User } from '../../domain/entities/user.entity';
import { IUserRepository } from '../../domain/interfaces/user-repository.interface';
import { UserAlreadyExistsError } from '../../domain/errors/user-already-exists.error';
import { CreateUserDto } from '../../application/dtos/create-user.dto';
import { UserResponseDto } from '../../application/dtos/user-response.dto';
import { UserMapper } from '../../application/mappers/user.mapper';
import { USER_REPOSITORY } from '../../users.module';

@Injectable()
export class CreateUserUseCase {
  constructor(
    @Inject(USER_REPOSITORY)
    private readonly userRepository: IUserRepository,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async execute(dto: CreateUserDto): Promise<UserResponseDto> {
    // 1. Regra de negócio: email único
    const exists = await this.userRepository.existsByEmail(
      Email.create(dto.email),
    );
    if (exists) throw new UserAlreadyExistsError(dto.email);

    // 2. Criar entity — regras de negócio aplicadas no domain
    const user = User.create({
      name: dto.name,
      email: dto.email,
      role: dto.role,
    });

    // 3. Persistir
    await this.userRepository.save(user);

    // 4. Publicar domain events
    for (const event of user.domainEvents) {
      await this.eventEmitter.emitAsync(event.eventName, event);
    }
    user.clearDomainEvents();

    // 5. Retornar DTO (nunca a entity)
    return UserMapper.toResponseDto(user);
  }
}
```

### Get / Query (sem efeito lateral)

```typescript
// modules/users/application/use-cases/get-user/get-user.use-case.ts
@Injectable()
export class GetUserUseCase {
  constructor(
    @Inject(USER_REPOSITORY)
    private readonly userRepository: IUserRepository,
  ) {}

  async execute(id: string): Promise<UserResponseDto> {
    const user = await this.userRepository.findById(UserId.from(id));
    if (!user) throw new UserNotFoundError(id);
    return UserMapper.toResponseDto(user);
  }
}
```

### List / Paginated Query

```typescript
// modules/users/application/use-cases/list-users/list-users.use-case.ts
@Injectable()
export class ListUsersUseCase {
  constructor(
    @Inject(USER_REPOSITORY)
    private readonly userRepository: IUserRepository,
  ) {}

  async execute(
    query: ListUsersQueryDto,
  ): Promise<PaginatedResponseDto<UserResponseDto>> {
    const result = await this.userRepository.findAll({
      page: query.page ?? 1,
      limit: Math.min(query.limit ?? 20, 100), // max 100 por página
      orderBy: query.orderBy ?? 'createdAt',
      order: query.order ?? 'DESC',
    });

    return {
      data: result.data.map(UserMapper.toResponseDto),
      total: result.total,
      page: result.page,
      totalPages: result.totalPages,
    };
  }
}
```

### Update (parcial — RFC 7396)

```typescript
// modules/users/application/use-cases/update-user/update-user.use-case.ts
@Injectable()
export class UpdateUserUseCase {
  constructor(
    @Inject(USER_REPOSITORY)
    private readonly userRepository: IUserRepository,
  ) {}

  async execute(id: string, dto: UpdateUserDto): Promise<UserResponseDto> {
    const user = await this.userRepository.findById(UserId.from(id));
    if (!user) throw new UserNotFoundError(id);

    // Aplica apenas os campos presentes no DTO (PATCH semântico)
    if (dto.name !== undefined) user.changeName(dto.name);
    if (dto.role !== undefined) user.assignRole(dto.role);

    await this.userRepository.save(user);
    return UserMapper.toResponseDto(user);
  }
}
```

### Complex Use Case (multi-repositório, orquestração)

```typescript
// modules/payments/application/use-cases/process-payment/process-payment.use-case.ts
@Injectable()
export class ProcessPaymentUseCase {
  constructor(
    @Inject(PAYMENT_REPOSITORY)
    private readonly paymentRepo: IPaymentRepository,
    @Inject(ACCOUNT_REPOSITORY)
    private readonly accountRepo: IAccountRepository,
    @Inject(PAYMENT_GATEWAY) private readonly gateway: IPaymentGateway,
    private readonly paymentLimitService: PaymentLimitService,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async execute(dto: ProcessPaymentDto): Promise<PaymentResponseDto> {
    // 1. Carregar agregados necessários
    const [senderAccount, recipientAccount] = await Promise.all([
      this.accountRepo.findByUserId(UserId.from(dto.senderUserId)),
      this.accountRepo.findByNumber(dto.recipientAccountNumber),
    ]);

    if (!senderAccount) throw new AccountNotFoundError(dto.senderUserId);
    if (!recipientAccount)
      throw new AccountNotFoundError(dto.recipientAccountNumber);

    const amount = Money.of(dto.amount, dto.currency);

    // 2. Validar regras de domínio
    const dailyTotal = await this.paymentRepo.getDailyTotalByAccount(
      senderAccount.id,
    );
    this.paymentLimitService.validatePaymentAmount(amount, dailyTotal);
    senderAccount.validateSufficientFunds(amount);

    // 3. Criar entity de pagamento
    const payment = Payment.create({
      senderAccountId: senderAccount.id,
      recipientAccountId: recipientAccount.id,
      amount,
      description: dto.description,
    });

    // 4. Processar no gateway externo (port — não conhece Stripe/PayPal directamente)
    const gatewayResult = await this.gateway.processPayment({
      reference: payment.id.value,
      amount: amount.amount,
      currency: amount.currency,
    });

    payment.markAsProcessed(gatewayResult.transactionId);

    // 5. Persistir
    await this.paymentRepo.save(payment);

    // 6. Publicar eventos
    for (const event of payment.domainEvents) {
      await this.eventEmitter.emitAsync(event.eventName, event);
    }

    return PaymentMapper.toResponseDto(payment);
  }
}
```

---

## DTOs — Padrões com class-validator

### Request DTO

```typescript
// modules/users/application/dtos/create-user.dto.ts
import {
  IsEmail,
  IsEnum,
  IsNotEmpty,
  IsOptional,
  IsString,
  MaxLength,
  MinLength,
} from 'class-validator';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { Transform } from 'class-transformer';
import { UserRole } from '../../domain/entities/user.entity';

export class CreateUserDto {
  @ApiProperty({ example: 'João Silva', minLength: 2, maxLength: 100 })
  @IsString()
  @IsNotEmpty()
  @MinLength(2)
  @MaxLength(100)
  @Transform(({ value }) => value?.trim())
  name: string;

  @ApiProperty({ example: 'joao@bpc.ao' })
  @IsEmail({}, { message: 'Invalid email format' })
  @Transform(({ value }) => value?.toLowerCase().trim())
  email: string;

  @ApiPropertyOptional({ enum: UserRole, default: UserRole.Customer })
  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole;
}
```

### Response DTO — com @Exclude e @Expose

```typescript
// modules/users/application/dtos/user-response.dto.ts
import { Exclude, Expose } from 'class-transformer';
import { ApiProperty } from '@nestjs/swagger';
import { UserRole } from '../../domain/entities/user.entity';

@Exclude() // por defeito, nada é exposto
export class UserResponseDto {
  @Expose()
  @ApiProperty({ example: '550e8400-e29b-41d4-a716-446655440000' })
  id: string;

  @Expose()
  @ApiProperty({ example: 'João Silva' })
  name: string;

  @Expose()
  @ApiProperty({ example: 'joao@bpc.ao' })
  email: string;

  @Expose()
  @ApiProperty({ enum: UserRole })
  role: UserRole;

  @Expose()
  @ApiProperty()
  isActive: boolean;

  @Expose()
  @ApiProperty()
  createdAt: Date;

  // updatedAt NÃO exposto por escolha — não aparece na resposta
}
```

### Update DTO — PartialType

```typescript
// modules/users/application/dtos/update-user.dto.ts
import { PartialType, OmitType } from '@nestjs/swagger';
import { CreateUserDto } from './create-user.dto';

// PartialType torna todos os campos opcionais E mantém decorators de validação
export class UpdateUserDto extends PartialType(
  OmitType(CreateUserDto, ['email'] as const), // email não pode ser alterado
) {}
```

### Pagination DTO partilhado

```typescript
// shared/application/dtos/pagination-query.dto.ts
import { IsInt, IsOptional, IsString, Max, Min } from 'class-validator';
import { ApiPropertyOptional } from '@nestjs/swagger';
import { Type } from 'class-transformer';

export class PaginationQueryDto {
  @ApiPropertyOptional({ default: 1, minimum: 1 })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @ApiPropertyOptional({ default: 20, minimum: 1, maximum: 100 })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 20;

  @ApiPropertyOptional()
  @IsOptional()
  @IsString()
  orderBy?: string;

  @ApiPropertyOptional({ enum: ['ASC', 'DESC'], default: 'DESC' })
  @IsOptional()
  order?: 'ASC' | 'DESC' = 'DESC';
}

// shared/application/dtos/paginated-response.dto.ts
export class PaginatedResponseDto<T> {
  data: T[];
  total: number;
  page: number;
  totalPages: number;
}
```

---

## Mappers — Entity ↔ DTO ↔ ORM

```typescript
// modules/users/application/mappers/user.mapper.ts
import { User } from '../../domain/entities/user.entity';
import { UserResponseDto } from '../dtos/user-response.dto';
import { UserOrmEntity } from '../../infrastructure/persistence/user.orm-entity';

export class UserMapper {
  // Entity de domínio → DTO de resposta
  static toResponseDto(user: User): UserResponseDto {
    const dto = new UserResponseDto();
    dto.id = user.id.value;
    dto.name = user.name;
    dto.email = user.email.value;
    dto.role = user.role;
    dto.isActive = user.isActive;
    dto.createdAt = user.createdAt;
    return dto;
  }

  // ORM entity → Domain entity (reconstituição)
  static toDomain(orm: UserOrmEntity): User {
    return User.reconstitute({
      id: orm.id,
      email: orm.email,
      name: orm.name,
      role: orm.role,
      isActive: orm.isActive,
      createdAt: orm.createdAt,
      updatedAt: orm.updatedAt,
    });
  }

  // Domain entity → ORM entity (para persistência)
  static toOrm(user: User): Partial<UserOrmEntity> {
    return {
      id: user.id.value,
      email: user.email.value,
      name: user.name,
      role: user.role,
      isActive: user.isActive,
      createdAt: user.createdAt,
      updatedAt: user.updatedAt,
    };
  }
}
```

---

## Ports — Interfaces para Serviços Externos

```typescript
// modules/payments/domain/interfaces/payment-gateway.interface.ts
// Port definido no Application/Domain — interface agnóstica de implementação

export interface ProcessPaymentRequest {
  reference: string;
  amount: number;
  currency: string;
  metadata?: Record<string, string>;
}

export interface ProcessPaymentResponse {
  transactionId: string;
  status: 'SUCCESS' | 'PENDING' | 'FAILED';
  processedAt: Date;
}

export interface IPaymentGateway {
  processPayment(
    request: ProcessPaymentRequest,
  ): Promise<ProcessPaymentResponse>;
  refundPayment(transactionId: string, amount: number): Promise<void>;
  getTransactionStatus(transactionId: string): Promise<string>;
}

// O token para DI
export const PAYMENT_GATEWAY = Symbol('IPaymentGateway');
```

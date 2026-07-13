# Domain Layer — Entities, Value Objects, Events, Errors

O Domain é o coração da aplicação. É TypeScript puro — zero imports de NestJS, TypeORM, Prisma,
class-validator ou qualquer framework. Deve ser possível copiar a pasta `domain/` para qualquer
projecto Node.js e continuar a funcionar.

---

## Entities de Domínio

### Padrão de Entity com factory method

```typescript
// modules/users/domain/entities/user.entity.ts

import { UserId } from '../value-objects/user-id.vo';
import { Email } from '../value-objects/email.vo';
import { UserCreatedEvent } from '../events/user-created.event';
import { UserDeactivatedEvent } from '../events/user-deactivated.event';
import { InvalidUserDataError } from '../errors/invalid-user-data.error';
import { UserAlreadyInactiveError } from '../errors/user-already-inactive.error';

export interface CreateUserProps {
  name: string;
  email: string;
  role?: UserRole;
}

export enum UserRole {
  Admin = 'ADMIN',
  Customer = 'CUSTOMER',
  Agent = 'AGENT',
}

export class User {
  private readonly _domainEvents: unknown[] = [];

  // Constructor privado — força uso do factory method
  private constructor(
    public readonly id: UserId,
    public readonly email: Email,
    private _name: string,
    private _role: UserRole,
    private _isActive: boolean,
    public readonly createdAt: Date,
    private _updatedAt: Date,
  ) {}

  // ── Factory method: criação de novo utilizador ──────────────────────────────
  static create(props: CreateUserProps): User {
    // Regras de negócio — não de HTTP
    if (!props.name || props.name.trim().length < 2) {
      throw new InvalidUserDataError('name', 'Must have at least 2 characters');
    }

    const user = new User(
      UserId.generate(),
      Email.create(props.email),
      props.name.trim(),
      props.role ?? UserRole.Customer,
      true,
      new Date(),
      new Date(),
    );

    user._domainEvents.push(
      new UserCreatedEvent(user.id.value, user.email.value),
    );
    return user;
  }

  // ── Factory method: reconstituição a partir da persistência ─────────────────
  static reconstitute(props: {
    id: string;
    email: string;
    name: string;
    role: UserRole;
    isActive: boolean;
    createdAt: Date;
    updatedAt: Date;
  }): User {
    return new User(
      UserId.from(props.id),
      Email.from(props.email),
      props.name,
      props.role,
      props.isActive,
      props.createdAt,
      props.updatedAt,
    );
  }

  // ── Getters ─────────────────────────────────────────────────────────────────
  get name(): string {
    return this._name;
  }
  get role(): UserRole {
    return this._role;
  }
  get isActive(): boolean {
    return this._isActive;
  }
  get updatedAt(): Date {
    return this._updatedAt;
  }
  get domainEvents(): unknown[] {
    return [...this._domainEvents];
  }

  // ── Comportamentos de negócio ────────────────────────────────────────────────
  changeName(name: string): void {
    if (!name || name.trim().length < 2) {
      throw new InvalidUserDataError('name', 'Must have at least 2 characters');
    }
    this._name = name.trim();
    this._updatedAt = new Date();
  }

  deactivate(): void {
    if (!this._isActive) {
      throw new UserAlreadyInactiveError(this.id.value);
    }
    this._isActive = false;
    this._updatedAt = new Date();
    this._domainEvents.push(new UserDeactivatedEvent(this.id.value));
  }

  assignRole(role: UserRole): void {
    this._role = role;
    this._updatedAt = new Date();
  }

  clearDomainEvents(): void {
    this._domainEvents.length = 0;
  }
}
```

---

## Value Objects

### Padrão de Value Object

```typescript
// modules/users/domain/value-objects/email.vo.ts

import { InvalidUserDataError } from '../errors/invalid-user-data.error';

export class Email {
  private static readonly REGEX = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

  private constructor(private readonly _value: string) {}

  // Criação com validação (novo email)
  static create(email: string): Email {
    if (!email || !this.REGEX.test(email.toLowerCase())) {
      throw new InvalidUserDataError('email', 'Invalid email format');
    }
    return new Email(email.toLowerCase().trim());
  }

  // Reconstituição sem validação (já foi validado ao guardar)
  static from(email: string): Email {
    return new Email(email);
  }

  get value(): string {
    return this._value;
  }

  equals(other: Email): boolean {
    return this._value === other._value;
  }

  toString(): string {
    return this._value;
  }
}
```

```typescript
// modules/users/domain/value-objects/user-id.vo.ts
import { randomUUID } from 'crypto';

export class UserId {
  private constructor(private readonly _value: string) {}

  static generate(): UserId {
    return new UserId(randomUUID());
  }

  static from(id: string): UserId {
    if (!id) throw new Error('UserId cannot be empty');
    return new UserId(id);
  }

  get value(): string {
    return this._value;
  }

  equals(other: UserId): boolean {
    return this._value === other._value;
  }

  toString(): string {
    return this._value;
  }
}
```

```typescript
// shared/domain/value-objects/money.vo.ts
export class Money {
  private constructor(
    private readonly _amount: number, // em cêntimos / valor inteiro
    private readonly _currency: string,
  ) {}

  static of(amount: number, currency: string): Money {
    if (amount < 0) throw new Error('Amount cannot be negative');
    if (!currency || currency.length !== 3)
      throw new Error('Invalid currency code');
    return new Money(Math.round(amount * 100) / 100, currency.toUpperCase());
  }

  get amount(): number {
    return this._amount;
  }
  get currency(): string {
    return this._currency;
  }

  add(other: Money): Money {
    this.assertSameCurrency(other);
    return Money.of(this._amount + other._amount, this._currency);
  }

  subtract(other: Money): Money {
    this.assertSameCurrency(other);
    if (other._amount > this._amount) {
      throw new Error('Cannot subtract: insufficient funds');
    }
    return Money.of(this._amount - other._amount, this._currency);
  }

  isGreaterThan(other: Money): boolean {
    this.assertSameCurrency(other);
    return this._amount > other._amount;
  }

  equals(other: Money): boolean {
    return this._amount === other._amount && this._currency === other._currency;
  }

  private assertSameCurrency(other: Money): void {
    if (this._currency !== other._currency) {
      throw new Error(
        `Currency mismatch: ${this._currency} vs ${other._currency}`,
      );
    }
  }

  toString(): string {
    return `${this._amount.toFixed(2)} ${this._currency}`;
  }
}
```

---

## Domain Events

```typescript
// shared/domain/events/domain-event.base.ts
export abstract class DomainEvent {
  public readonly occurredAt: Date;
  public readonly eventId: string;

  constructor(public readonly aggregateId: string) {
    this.occurredAt = new Date();
    this.eventId = randomUUID();
  }

  abstract get eventName(): string;
}

// modules/users/domain/events/user-created.event.ts
import { DomainEvent } from '@/shared/domain/events/domain-event.base';

export class UserCreatedEvent extends DomainEvent {
  constructor(
    userId: string,
    public readonly email: string,
  ) {
    super(userId);
  }

  get eventName(): string {
    return 'user.created';
  }
}
```

### Event Handler (Infrastructure)

```typescript
// modules/users/infrastructure/event-handlers/send-welcome-email.handler.ts
import { OnEvent } from '@nestjs/event-emitter';
import { Injectable } from '@nestjs/common';
import { UserCreatedEvent } from '../../domain/events/user-created.event';

@Injectable()
export class SendWelcomeEmailHandler {
  constructor(private readonly emailService: EmailService) {}

  @OnEvent('user.created', { async: true })
  async handle(event: UserCreatedEvent): Promise<void> {
    await this.emailService.sendWelcome(event.email);
  }
}
```

---

## Domain Errors

```typescript
// shared/domain/errors/domain.error.ts
export abstract class DomainError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusHint: number = 400, // HTTP status sugerido
  ) {
    super(message);
    this.name = this.constructor.name;
    Object.setPrototypeOf(this, new.target.prototype); // fix instanceof em TypeScript
  }
}

// modules/users/domain/errors/user-not-found.error.ts
export class UserNotFoundError extends DomainError {
  constructor(identifier: string) {
    super(`User "${identifier}" not found`, 'USER_NOT_FOUND', 404);
  }
}

// modules/users/domain/errors/user-already-exists.error.ts
export class UserAlreadyExistsError extends DomainError {
  constructor(email: string) {
    super(
      `User with email "${email}" already exists`,
      'USER_ALREADY_EXISTS',
      409,
    );
  }
}

// modules/users/domain/errors/invalid-user-data.error.ts
export class InvalidUserDataError extends DomainError {
  constructor(field: string, reason: string) {
    super(`Invalid value for "${field}": ${reason}`, 'INVALID_USER_DATA', 422);
  }
}

// modules/payments/domain/errors/insufficient-funds.error.ts
export class InsufficientFundsError extends DomainError {
  constructor(available: number, required: number, currency: string) {
    super(
      `Insufficient funds: available ${available} ${currency}, required ${required} ${currency}`,
      'INSUFFICIENT_FUNDS',
      422,
    );
  }
}
```

---

## Domain Services

Domain Services existem para lógica de negócio que não pertence a uma única Entity.

```typescript
// modules/payments/domain/services/payment-limit.service.ts
// (TypeScript puro — sem @Injectable, sem NestJS)

export class PaymentLimitService {
  private static readonly DAILY_LIMIT = Money.of(500_000, 'AOA');
  private static readonly SINGLE_LIMIT = Money.of(100_000, 'AOA');

  validatePaymentAmount(amount: Money, dailyTotal: Money): void {
    if (amount.isGreaterThan(PaymentLimitService.SINGLE_LIMIT)) {
      throw new ExceedsSinglePaymentLimitError(
        amount.toString(),
        PaymentLimitService.SINGLE_LIMIT.toString(),
      );
    }

    const newDailyTotal = dailyTotal.add(amount);
    if (newDailyTotal.isGreaterThan(PaymentLimitService.DAILY_LIMIT)) {
      throw new ExceedsDailyLimitError(newDailyTotal.toString());
    }
  }
}
```

---

## Repository Interface (Domain/Application)

```typescript
// modules/users/domain/interfaces/user-repository.interface.ts

export interface IUserRepository {
  findById(id: UserId): Promise<User | null>;
  findByEmail(email: Email): Promise<User | null>;
  findAll(pagination: PaginationOptions): Promise<PaginatedResult<User>>;
  save(user: User): Promise<void>; // insert ou update (upsert)
  delete(id: UserId): Promise<void>;
  existsByEmail(email: Email): Promise<boolean>;
}

// shared/domain/value-objects/pagination.vo.ts
export interface PaginationOptions {
  page: number;
  limit: number;
  orderBy?: string;
  order?: 'ASC' | 'DESC';
}

export interface PaginatedResult<T> {
  data: T[];
  total: number;
  page: number;
  totalPages: number;
}
```

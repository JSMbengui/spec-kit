# Testes — Unit, Integration e E2E

## Stack de Testes

| Tipo        | Ferramenta           | Foco                       | Cobertura mínima |
| ----------- | -------------------- | -------------------------- | ---------------- |
| Unit        | Jest                 | Use Cases, Domain, Mappers | 80%              |
| Integration | Jest + TestingModule | Módulos com DB de teste    | 70%              |
| E2E         | Jest + Supertest     | HTTP endpoints completos   | Fluxos críticos  |

---

## Configuração Jest

```typescript
// jest.config.ts
import type { Config } from 'jest';

const config: Config = {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  transform: { '^.+\\.(t|j)s$': 'ts-jest' },
  collectCoverageFrom: [
    '**/*.(t|j)s',
    '!**/*.module.ts',
    '!**/*.orm-entity.ts',
    '!**/main.ts',
    '!**/*.dto.ts', // DTOs validados pelos testes de integração
    '!**/migrations/**',
  ],
  coverageThresholds: {
    global: { statements: 80, branches: 70, functions: 80, lines: 80 },
  },
  coverageDirectory: '../coverage',
  testEnvironment: 'node',
  moduleNameMapper: { '@/(.*)': '<rootDir>/$1' },
};

// jest-e2e.config.ts
const e2eConfig: Config = {
  ...config,
  rootDir: '.',
  testRegex: '.e2e-spec.ts$',
  testEnvironment: 'node',
};
```

---

## Testes Unitários — Use Cases

```typescript
// modules/users/application/use-cases/create-user/create-user.use-case.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { EventEmitter2 } from '@nestjs/event-emitter';
import { CreateUserUseCase } from './create-user.use-case';
import { IUserRepository } from '../../domain/interfaces/user-repository.interface';
import { UserAlreadyExistsError } from '../../domain/errors/user-already-exists.error';
import { USER_REPOSITORY } from '../../users.module';
import { createMockUser } from '../../../../../test/fixtures/user.fixture';

describe('CreateUserUseCase', () => {
  let useCase: CreateUserUseCase;
  let userRepository: jest.Mocked<IUserRepository>;
  let eventEmitter: jest.Mocked<EventEmitter2>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        CreateUserUseCase,
        {
          provide: USER_REPOSITORY,
          useValue: {
            findById: jest.fn(),
            findByEmail: jest.fn(),
            existsByEmail: jest.fn(),
            save: jest.fn(),
            findAll: jest.fn(),
            delete: jest.fn(),
          } satisfies jest.Mocked<IUserRepository>,
        },
        {
          provide: EventEmitter2,
          useValue: { emitAsync: jest.fn().mockResolvedValue([]) },
        },
      ],
    }).compile();

    useCase = module.get(CreateUserUseCase);
    userRepository = module.get(USER_REPOSITORY);
    eventEmitter = module.get(EventEmitter2);
  });

  afterEach(() => jest.clearAllMocks());

  describe('execute', () => {
    const validDto = { name: 'João Silva', email: 'joao@bpc.ao' };

    it('creates user successfully when email is unique', async () => {
      userRepository.existsByEmail.mockResolvedValue(false);
      userRepository.save.mockResolvedValue(undefined);

      const result = await useCase.execute(validDto);

      expect(result.name).toBe('João Silva');
      expect(result.email).toBe('joao@bpc.ao');
      expect(result.id).toBeDefined();
      expect(userRepository.save).toHaveBeenCalledTimes(1);
      expect(eventEmitter.emitAsync).toHaveBeenCalledWith(
        'user.created',
        expect.objectContaining({ email: 'joao@bpc.ao' }),
      );
    });

    it('throws UserAlreadyExistsError when email is taken', async () => {
      userRepository.existsByEmail.mockResolvedValue(true);

      await expect(useCase.execute(validDto)).rejects.toThrow(
        UserAlreadyExistsError,
      );

      expect(userRepository.save).not.toHaveBeenCalled();
    });

    it('trims whitespace from name', async () => {
      userRepository.existsByEmail.mockResolvedValue(false);
      userRepository.save.mockResolvedValue(undefined);

      const result = await useCase.execute({ ...validDto, name: '  João  ' });
      expect(result.name).toBe('João');
    });
  });
});
```

---

## Testes Unitários — Domain Entities

```typescript
// modules/users/domain/entities/user.entity.spec.ts
import { User } from './user.entity';
import { InvalidUserDataError } from '../errors/invalid-user-data.error';
import { UserAlreadyInactiveError } from '../errors/user-already-inactive.error';

describe('User Entity', () => {
  const validProps = { name: 'Maria Santos', email: 'maria@bpc.ao' };

  describe('create()', () => {
    it('creates a user with correct initial state', () => {
      const user = User.create(validProps);
      expect(user.name).toBe('Maria Santos');
      expect(user.email.value).toBe('maria@bpc.ao');
      expect(user.isActive).toBe(true);
      expect(user.id).toBeDefined();
      expect(user.createdAt).toBeInstanceOf(Date);
    });

    it('emits UserCreatedEvent on creation', () => {
      const user = User.create(validProps);
      expect(user.domainEvents).toHaveLength(1);
      expect(user.domainEvents[0]).toMatchObject({
        email: 'maria@bpc.ao',
        eventName: 'user.created',
      });
    });

    it('throws InvalidUserDataError for name shorter than 2 chars', () => {
      expect(() => User.create({ ...validProps, name: 'A' })).toThrow(
        InvalidUserDataError,
      );
    });

    it('throws InvalidUserDataError for invalid email', () => {
      expect(() =>
        User.create({ ...validProps, email: 'not-an-email' }),
      ).toThrow(InvalidUserDataError);
    });
  });

  describe('deactivate()', () => {
    it('deactivates an active user', () => {
      const user = User.create(validProps);
      user.clearDomainEvents();
      user.deactivate();
      expect(user.isActive).toBe(false);
      expect(user.domainEvents[0]).toMatchObject({
        eventName: 'user.deactivated',
      });
    });

    it('throws UserAlreadyInactiveError if already inactive', () => {
      const user = User.create(validProps);
      user.deactivate();
      expect(() => user.deactivate()).toThrow(UserAlreadyInactiveError);
    });
  });

  describe('changeName()', () => {
    it('updates name and updatedAt', () => {
      const user = User.create(validProps);
      const before = user.updatedAt;
      user.changeName('Maria Silva');
      expect(user.name).toBe('Maria Silva');
      expect(user.updatedAt.getTime()).toBeGreaterThanOrEqual(before.getTime());
    });
  });
});
```

---

## Testes de Value Objects

```typescript
// modules/users/domain/value-objects/email.vo.spec.ts
import { Email } from './email.vo';
import { InvalidUserDataError } from '../errors/invalid-user-data.error';

describe('Email Value Object', () => {
  it('creates valid email in lowercase', () => {
    const email = Email.create('JOAO@BPC.AO');
    expect(email.value).toBe('joao@bpc.ao');
  });

  it.each([['not-email'], ['@bpc.ao'], ['joao@'], [''], ['  ']])(
    'throws for invalid email: %s',
    (invalid) => {
      expect(() => Email.create(invalid)).toThrow(InvalidUserDataError);
    },
  );

  it('two emails with same value are equal', () => {
    expect(Email.create('a@b.com').equals(Email.create('A@B.COM'))).toBe(true);
  });
});
```

---

## Testes de Integração — TestingModule com DB

```typescript
// modules/users/infrastructure/repositories/user.typeorm-repository.spec.ts
import { Test } from '@nestjs/testing';
import { TypeOrmModule, getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { UserTypeOrmRepository } from './user.typeorm-repository';
import { UserOrmEntity } from '../persistence/user.orm-entity';
import { User } from '../../domain/entities/user.entity';
import { UserId } from '../../domain/value-objects/user-id.vo';
import { Email } from '../../domain/value-objects/email.vo';

describe('UserTypeOrmRepository (integration)', () => {
  let repository: UserTypeOrmRepository;
  let ormRepo: Repository<UserOrmEntity>;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [
        TypeOrmModule.forRoot({
          type: 'sqlite', // SQLite em memória para testes
          database: ':memory:',
          entities: [UserOrmEntity],
          synchronize: true,
        }),
        TypeOrmModule.forFeature([UserOrmEntity]),
      ],
      providers: [UserTypeOrmRepository],
    }).compile();

    repository = module.get(UserTypeOrmRepository);
    ormRepo = module.get(getRepositoryToken(UserOrmEntity));
  });

  beforeEach(async () => {
    await ormRepo.clear(); // limpar entre testes
  });

  it('saves and retrieves a user by id', async () => {
    const user = User.create({ name: 'Test User', email: 'test@bpc.ao' });
    await repository.save(user);

    const found = await repository.findById(user.id);
    expect(found).not.toBeNull();
    expect(found!.name).toBe('Test User');
    expect(found!.email.value).toBe('test@bpc.ao');
  });

  it('returns null for non-existent user', async () => {
    const result = await repository.findById(UserId.generate());
    expect(result).toBeNull();
  });

  it('existsByEmail returns true when email exists', async () => {
    const user = User.create({ name: 'Test', email: 'exists@bpc.ao' });
    await repository.save(user);
    expect(await repository.existsByEmail(Email.create('exists@bpc.ao'))).toBe(
      true,
    );
    expect(await repository.existsByEmail(Email.create('no@bpc.ao'))).toBe(
      false,
    );
  });

  it('paginates results correctly', async () => {
    for (let i = 0; i < 5; i++) {
      await repository.save(
        User.create({ name: `User ${i}`, email: `user${i}@bpc.ao` }),
      );
    }
    const page1 = await repository.findAll({ page: 1, limit: 3 });
    const page2 = await repository.findAll({ page: 2, limit: 3 });

    expect(page1.data).toHaveLength(3);
    expect(page1.total).toBe(5);
    expect(page1.totalPages).toBe(2);
    expect(page2.data).toHaveLength(2);
  });
});
```

---

## Testes E2E — Supertest

```typescript
// test/e2e/users.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../../src/app.module';
import { GlobalExceptionFilter } from '../../src/shared/presentation/filters/global-exception.filter';

describe('Users API (e2e)', () => {
  let app: INestApplication;
  let authToken: string;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();

    // Mesma configuração do main.ts
    app.setGlobalPrefix('api');
    app.useGlobalPipes(
      new ValidationPipe({ whitelist: true, transform: true }),
    );
    app.useGlobalFilters(new GlobalExceptionFilter());

    await app.init();

    // Login para obter token
    const loginRes = await request(app.getHttpServer())
      .post('/api/v1/auth/login')
      .send({ email: 'admin@bpc.ao', password: 'Admin1234!' });
    authToken = loginRes.body.data.accessToken;
  });

  afterAll(async () => await app.close());

  describe('POST /api/v1/users', () => {
    it('creates user and returns 201', async () => {
      const res = await request(app.getHttpServer())
        .post('/api/v1/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'New User', email: `user_${Date.now()}@bpc.ao` })
        .expect(201);

      expect(res.body.data.id).toBeDefined();
      expect(res.body.data.email).toContain('@bpc.ao');
      expect(res.body.timestamp).toBeDefined();
    });

    it('returns 422 on invalid email', async () => {
      const res = await request(app.getHttpServer())
        .post('/api/v1/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'Test', email: 'not-an-email' })
        .expect(422);

      expect(res.body.type).toContain('unprocessable-entity');
      expect(res.body.errors).toBeDefined();
    });

    it('returns 409 on duplicate email', async () => {
      const email = `dup_${Date.now()}@bpc.ao`;
      await request(app.getHttpServer())
        .post('/api/v1/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'First', email })
        .expect(201);

      const res = await request(app.getHttpServer())
        .post('/api/v1/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'Second', email })
        .expect(409);

      expect(res.body.type).toContain('user-already-exists');
    });

    it('returns 401 without token', async () => {
      await request(app.getHttpServer())
        .post('/api/v1/users')
        .send({ name: 'Test', email: 'test@bpc.ao' })
        .expect(401);
    });
  });

  describe('GET /api/v1/users/:id', () => {
    it('returns user by id', async () => {
      // Criar primeiro
      const created = await request(app.getHttpServer())
        .post('/api/v1/users')
        .set('Authorization', `Bearer ${authToken}`)
        .send({ name: 'Fetch Test', email: `fetch_${Date.now()}@bpc.ao` });

      const res = await request(app.getHttpServer())
        .get(`/api/v1/users/${created.body.data.id}`)
        .set('Authorization', `Bearer ${authToken}`)
        .expect(200);

      expect(res.body.data.name).toBe('Fetch Test');
    });

    it('returns 404 for non-existent id', async () => {
      const res = await request(app.getHttpServer())
        .get('/api/v1/users/00000000-0000-0000-0000-000000000000')
        .set('Authorization', `Bearer ${authToken}`)
        .expect(404);

      expect(res.body.type).toContain('user-not-found');
    });
  });
});
```

---

## Fixtures — Dados de Teste

```typescript
// test/fixtures/user.fixture.ts
import { User } from '../../src/modules/users/domain/entities/user.entity';
import { UserOrmEntity } from '../../src/modules/users/infrastructure/persistence/user.orm-entity';
import { UserRole } from '../../src/modules/users/domain/entities/user.entity';

let counter = 0;

export function createMockUser(
  overrides?: Partial<{
    name: string;
    email: string;
    role: UserRole;
  }>,
): User {
  return User.create({
    name: overrides?.name ?? `User ${++counter}`,
    email: overrides?.email ?? `user${counter}@bpc.ao`,
    role: overrides?.role,
  });
}

export function createMockUserOrmEntity(
  overrides?: Partial<UserOrmEntity>,
): UserOrmEntity {
  const entity = new UserOrmEntity();
  entity.id = overrides?.id ?? randomUUID();
  entity.name = overrides?.name ?? `User ${++counter}`;
  entity.email = overrides?.email ?? `user${counter}@bpc.ao`;
  entity.role = overrides?.role ?? UserRole.Customer;
  entity.isActive = overrides?.isActive ?? true;
  entity.createdAt = overrides?.createdAt ?? new Date();
  entity.updatedAt = overrides?.updatedAt ?? new Date();
  return entity;
}
```

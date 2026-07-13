# Presentation Layer — Controllers, Guards, Interceptors, Pipes, Swagger

A Presentation é a fronteira HTTP. Recebe requests, valida via DTOs,
delega aos Use Cases e formata respostas. Zero lógica de negócio aqui.

---

## Controller — Padrão Completo

```typescript
// modules/users/presentation/controllers/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Patch,
  Delete,
  Body,
  Param,
  Query,
  HttpCode,
  HttpStatus,
  UseGuards,
  ParseUUIDPipe,
} from '@nestjs/common';
import {
  ApiTags,
  ApiOperation,
  ApiResponse,
  ApiBearerAuth,
  ApiParam,
  ApiQuery,
} from '@nestjs/swagger';
import { JwtAuthGuard } from '@/shared/presentation/guards/jwt-auth.guard';
import { RolesGuard } from '@/shared/presentation/guards/roles.guard';
import { Roles } from '@/shared/presentation/decorators/roles.decorator';
import { CurrentUser } from '../decorators/current-user.decorator';
import { CreateUserUseCase } from '../../application/use-cases/create-user/create-user.use-case';
import { GetUserUseCase } from '../../application/use-cases/get-user/get-user.use-case';
import { ListUsersUseCase } from '../../application/use-cases/list-users/list-users.use-case';
import { UpdateUserUseCase } from '../../application/use-cases/update-user/update-user.use-case';
import { DeactivateUserUseCase } from '../../application/use-cases/deactivate-user/deactivate-user.use-case';
import { CreateUserDto } from '../../application/dtos/create-user.dto';
import { UpdateUserDto } from '../../application/dtos/update-user.dto';
import { UserResponseDto } from '../../application/dtos/user-response.dto';
import { PaginationQueryDto } from '@/shared/application/dtos/pagination-query.dto';
import { PaginatedResponseDto } from '@/shared/application/dtos/paginated-response.dto';
import { UserRole } from '../../domain/entities/user.entity';

@ApiTags('users')
@ApiBearerAuth()
@Controller({ path: 'users', version: '1' })
@UseGuards(JwtAuthGuard, RolesGuard)
export class UsersController {
  constructor(
    private readonly createUserUseCase: CreateUserUseCase,
    private readonly getUserUseCase: GetUserUseCase,
    private readonly listUsersUseCase: ListUsersUseCase,
    private readonly updateUserUseCase: UpdateUserUseCase,
    private readonly deactivateUserUseCase: DeactivateUserUseCase,
  ) {}

  // ── POST /v1/users ─────────────────────────────────────────────────────────
  @Post()
  @HttpCode(HttpStatus.CREATED)
  @Roles(UserRole.Admin)
  @ApiOperation({ summary: 'Create a new user' })
  @ApiResponse({
    status: 201,
    type: UserResponseDto,
    description: 'User created',
  })
  @ApiResponse({
    status: 409,
    description: 'User with this email already exists',
  })
  @ApiResponse({ status: 422, description: 'Validation error' })
  create(@Body() dto: CreateUserDto): Promise<UserResponseDto> {
    return this.createUserUseCase.execute(dto);
  }

  // ── GET /v1/users ──────────────────────────────────────────────────────────
  @Get()
  @Roles(UserRole.Admin, UserRole.Agent)
  @ApiOperation({ summary: 'List all users (paginated)' })
  @ApiResponse({ status: 200, description: 'Paginated list of users' })
  findAll(
    @Query() query: PaginationQueryDto,
  ): Promise<PaginatedResponseDto<UserResponseDto>> {
    return this.listUsersUseCase.execute(query);
  }

  // ── GET /v1/users/me ───────────────────────────────────────────────────────
  @Get('me')
  @ApiOperation({ summary: 'Get current authenticated user' })
  @ApiResponse({ status: 200, type: UserResponseDto })
  getMe(@CurrentUser() userId: string): Promise<UserResponseDto> {
    return this.getUserUseCase.execute(userId);
  }

  // ── GET /v1/users/:id ──────────────────────────────────────────────────────
  @Get(':id')
  @Roles(UserRole.Admin, UserRole.Agent)
  @ApiOperation({ summary: 'Get user by ID' })
  @ApiParam({ name: 'id', type: 'string', format: 'uuid' })
  @ApiResponse({ status: 200, type: UserResponseDto })
  @ApiResponse({ status: 404, description: 'User not found' })
  findOne(@Param('id', ParseUUIDPipe) id: string): Promise<UserResponseDto> {
    return this.getUserUseCase.execute(id);
  }

  // ── PATCH /v1/users/:id ────────────────────────────────────────────────────
  @Patch(':id')
  @Roles(UserRole.Admin)
  @ApiOperation({ summary: 'Partially update a user (RFC 7396)' })
  @ApiResponse({ status: 200, type: UserResponseDto })
  @ApiResponse({ status: 404, description: 'User not found' })
  update(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() dto: UpdateUserDto,
  ): Promise<UserResponseDto> {
    return this.updateUserUseCase.execute(id, dto);
  }

  // ── DELETE /v1/users/:id ───────────────────────────────────────────────────
  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  @Roles(UserRole.Admin)
  @ApiOperation({ summary: 'Deactivate a user (soft delete)' })
  @ApiResponse({ status: 204, description: 'User deactivated' })
  @ApiResponse({ status: 404, description: 'User not found' })
  deactivate(@Param('id', ParseUUIDPipe) id: string): Promise<void> {
    return this.deactivateUserUseCase.execute(id);
  }
}
```

---

## Regra — Nunca usar `@Res()` com try/catch manual

O controller **nunca** injecta `@Res() res` para construir a resposta HTTP à mão, e **nunca**
envolve a chamada ao Use Case num `try/catch` para mapear erros manualmente. Isto existe
precisamente para ser resolvido pelo `GlobalExceptionFilter` (ver `references/error-handling.md`)
e, quando aplicável, pelo `TransformResponseInterceptor` (ver acima nesta página).

**Por que isto é grave, não apenas estilo:**
- `@Res()` desliga o modo de resposta automática do NestJS — a partir do momento em que injectas
  `@Res()`, és **tu** que tens de chamar `res.status().json()`, e o `GlobalExceptionFilter`
  deixa de apanhar excepções lançadas dentro de um `try` que já as capturou primeiro.
- `ClassSerializerInterceptor` (campos `@Exclude()`/`@Expose()` nos DTOs) deixa de correr sobre
  a resposta, porque já não há um valor de retorno para o Nest serializar — a serialização,
  se existir, tem de ser feita manualmente.
- O `try/catch` reimplementa, método a método, exactamente o que o filtro global já faz de forma
  centralizada — normalmente pior, com lógica frágil de extrair `error?.response?.message ??
  error?.message ?? error` e mapeamento de status ad-hoc.
- Qualquer regra nova de formatação de erro (novo campo no RFC 7807, novo código Prisma a tratar)
  tem de ser replicada em cada controller, em vez de um único ponto em
  `references/error-handling.md`.

```typescript
// ❌ NUNCA fazer — @Res() + try/catch manual reimplementa o GlobalExceptionFilter
@Get()
async list(
  @Res() res,
  @Query('companyId') companyId: string,
): Promise<void> {
  const timestamp = new Date().toISOString();
  try {
    const result = await this.listUseCase.execute(companyId);
    return res.status(HttpStatus.OK).json({
      success: true,
      statusCode: HttpStatus.OK,
      data: result,
      timestamp,
    });
  } catch (error) {
    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = 'Erro ao listar';
    if (error.status && error.message) {
      status = error.status;
      message = error.message;
    }
    return res.status(status).json({ success: false, statusCode: status, message, timestamp });
  }
}

// ✅ SEMPRE fazer — retorno directo, deixa o filtro e os interceptors tratarem do resto
@Get()
@ApiOperation({ summary: 'List document series' })
list(
  @Query() query: ListDocumentSeriesQueryDto,
): Promise<PaginatedResponseDto<DocumentSeriesResponseDto>> {
  return this.listUseCase.execute(query); // erros lançados pelo Use Case sobem para o filtro global
}
```

**Validação de pré-condição (ex: `companyId` obrigatório) nunca é um `res.status(400).json()`
manual — é uma excepção tipada lançada pelo Use Case ou por um DTO validado:**

```typescript
// ❌ Validação manual com resposta construída à mão no controller
if (!companyId) {
  return res.status(HttpStatus.BAD_REQUEST).json({
    success: false,
    message: 'companyId é obrigatório',
  });
}

// ✅ DomainError tipado, lançado no Use Case, traduzido pelo GlobalExceptionFilter
// modules/document-series/domain/errors/company-id-required.error.ts
export class CompanyIdRequiredError extends DomainError {
  constructor() {
    super('companyId é obrigatório', 'COMPANY_ID_REQUIRED', 400);
  }
}
```

**Única excepção legítima ao uso de `@Res()`:** quando o NestJS não suporta nativamente o tipo
de resposta necessário — por exemplo, stream de ficheiros binários, redirects manuais com
headers muito específicos, ou Server-Sent Events fora do `@Sse()` decorator nativo. Mesmo nesses
casos, usa `@Res({ passthrough: true })` em vez de `@Res()` puro sempre que possível, para
manter os interceptors globais activos e só tomar controlo explícito do necessário.

---

## Guards

### JWT Auth Guard

```typescript
// shared/presentation/guards/jwt-auth.guard.ts
import {
  Injectable,
  ExecutionContext,
  UnauthorizedException,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { AuthGuard } from '@nestjs/passport';
import { IS_PUBLIC_KEY } from '../decorators/public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext) {
    // Verificar se a rota está marcada como @Public()
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) return true;
    return super.canActivate(context);
  }

  handleRequest(err: any, user: any) {
    if (err || !user) {
      throw new UnauthorizedException('Invalid or expired token');
    }
    return user;
  }
}
```

### Roles Guard

```typescript
// shared/presentation/guards/roles.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  ForbiddenException,
} from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../decorators/roles.decorator';
import { UserRole } from '@/modules/users/domain/entities/user.entity';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<UserRole[]>(
      ROLES_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredRoles || requiredRoles.length === 0) return true;

    const { user } = context.switchToHttp().getRequest();
    if (!requiredRoles.includes(user?.role)) {
      throw new ForbiddenException('Insufficient permissions');
    }
    return true;
  }
}
```

### Throttler Guard (Rate Limiting)

```typescript
// app.module.ts — configurar globalmente
import { ThrottlerModule, ThrottlerGuard } from "@nestjs/throttler";
import { APP_GUARD } from "@nestjs/core";

@Module({
  imports: [
    ThrottlerModule.forRoot([
      { name: "short",  ttl: 1_000,  limit: 10  },  // 10 req/s
      { name: "medium", ttl: 10_000, limit: 50  },  // 50 req/10s
      { name: "long",   ttl: 60_000, limit: 200 },  // 200 req/min
    ]),
  ],
  providers: [
    { provide: APP_GUARD, useClass: ThrottlerGuard },  // global
  ],
})
export class AppModule {}

// Sobrepor limite por rota específica
@Throttle({ short: { limit: 3, ttl: 60_000 } })  // 3 req/min para login
@Post("auth/login")
login(@Body() dto: LoginDto) { ... }

// Desactivar throttle numa rota
@SkipThrottle()
@Get("health")
health() { return { status: "ok" }; }
```

---

## Interceptors

### Logging Interceptor

```typescript
// shared/presentation/interceptors/logging.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Logger,
} from '@nestjs/common';
import { Observable, tap } from 'rxjs';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger('HTTP');

  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const req = context.switchToHttp().getRequest();
    const { method, url, body, user } = req;
    const start = Date.now();

    return next.handle().pipe(
      tap({
        next: () => {
          const ms = Date.now() - start;
          this.logger.log(
            `${method} ${url} ${Date.now() - start}ms userId=${user?.id ?? 'anonymous'}`,
          );
        },
        error: (err) => {
          const ms = Date.now() - start;
          this.logger.error(`${method} ${url} ${ms}ms ERROR: ${err.message}`);
        },
      }),
    );
  }
}
```

### Transform Response Interceptor — envelope padronizado

```typescript
// shared/presentation/interceptors/transform-response.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable, map } from 'rxjs';

export interface ApiResponse<T> {
  data: T;
  timestamp: string;
  path: string;
}

@Injectable()
export class TransformResponseInterceptor<T> implements NestInterceptor<
  T,
  ApiResponse<T>
> {
  intercept(
    context: ExecutionContext,
    next: CallHandler<T>,
  ): Observable<ApiResponse<T>> {
    const req = context.switchToHttp().getRequest();
    return next.handle().pipe(
      map((data) => ({
        data,
        timestamp: new Date().toISOString(),
        path: req.url,
      })),
    );
  }
}
```

---

## Decorators Customizados

```typescript
// shared/presentation/decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common';
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

// shared/presentation/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';
import { UserRole } from '@/modules/users/domain/entities/user.entity';
export const ROLES_KEY = 'roles';
export const Roles = (...roles: UserRole[]) => SetMetadata(ROLES_KEY, roles);

// modules/users/presentation/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: keyof JwtPayload | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user as JwtPayload;
    return data ? user?.[data] : user?.sub; // por defeito retorna o userId
  },
);

// Uso: @CurrentUser() userId: string
// Uso: @CurrentUser("role") role: UserRole
```

---

## JWT Strategy

```typescript
// modules/auth/presentation/strategies/jwt.strategy.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';

export interface JwtPayload {
  sub: string; // userId
  email: string;
  role: string;
  iat: number;
  exp: number;
}

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: config.getOrThrow('JWT_SECRET'),
    });
  }

  async validate(payload: JwtPayload): Promise<JwtPayload> {
    if (!payload.sub) throw new UnauthorizedException();
    return payload; // este objecto fica em request.user
  }
}
```

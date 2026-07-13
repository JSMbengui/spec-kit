# Error Handling — Exceptions, Filtros e RFC 7807

## Hierarquia de Erros

```
Error (nativo JS)
└── DomainError (base — shared/domain/errors/)
    ├── UserNotFoundError            → 404
    ├── UserAlreadyExistsError       → 409
    ├── InvalidUserDataError         → 422
    ├── UserAlreadyInactiveError     → 409
    ├── InsufficientFundsError       → 422
    ├── ExceedsDailyLimitError       → 422
    └── PaymentNotFoundError         → 404
```

---

## Global Exception Filter — RFC 7807

> **Onde traduzir erros do Prisma:** o lugar preferido é dentro do próprio repositório
> (try/catch em `update`/`delete`, ver `references/infrastructure.md`), lançando o
> `DomainError` correspondente (ex: `AssemblyNotFoundError`). O bloco Prisma neste filtro é
> apenas uma rede de segurança para erros que não foram traduzidos na origem.

```typescript
// shared/presentation/filters/global-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Response, Request } from 'express';
import { DomainError } from '@/shared/domain/errors/domain.error';
import { QueryFailedError } from 'typeorm';
import { Prisma } from '@prisma/client';

interface ProblemDetail {
  type: string;
  title: string;
  status: number;
  detail: string;
  instance: string;
  traceId?: string;
  errors?: Record<string, string[]>;
}

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(GlobalExceptionFilter.name);
  private readonly baseUrl =
    process.env.API_DOCS_URL ?? 'https://api.example.com/errors';

  catch(exception: unknown, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const problem = this.buildProblemDetail(exception, request);

    // Logar apenas erros inesperados (5xx)
    if (problem.status >= 500) {
      this.logger.error(
        { exception, path: request.url, method: request.method },
        'Unexpected error',
      );
    }

    response.status(problem.status).json(problem);
  }

  private buildProblemDetail(
    exception: unknown,
    request: Request,
  ): ProblemDetail {
    const instance = request.url;
    const traceId = request.headers['x-trace-id'] as string | undefined;

    // 1. Domain Errors — mapeados para HTTP
    if (exception instanceof DomainError) {
      return {
        type: `${this.baseUrl}/${this.toSlug(exception.code)}`,
        title: this.toTitle(exception.code),
        status: exception.statusHint,
        detail: exception.message,
        instance,
        traceId,
      };
    }

    // 2. NestJS HttpException (BadRequestException, NotFoundException, etc.)
    if (exception instanceof HttpException) {
      const status = exception.getStatus();
      const res = exception.getResponse();
      const isValidationError = status === 422 || status === 400;

      return {
        type: `${this.baseUrl}/${this.httpStatusToSlug(status)}`,
        title: exception.message,
        status,
        detail:
          typeof res === 'object' && 'message' in res
            ? Array.isArray((res as any).message)
              ? 'Validation failed'
              : String((res as any).message)
            : exception.message,
        instance,
        traceId,
        ...(isValidationError && typeof res === 'object' && 'message' in res
          ? { errors: this.formatValidationErrors((res as any).message) }
          : {}),
      };
    }

    // 3. TypeORM — constraint violations
    if (exception instanceof QueryFailedError) {
      const pgError = (exception as any).code;
      if (pgError === '23505') {
        // unique violation
        return {
          type: `${this.baseUrl}/conflict`,
          title: 'Conflict',
          status: 409,
          detail: 'A record with this data already exists',
          instance,
          traceId,
        };
      }
    }

    // 3b. Prisma — erros conhecidos do client
    // Nota: idealmente estes erros já foram capturados e traduzidos para um DomainError
    // dentro do próprio repositório (ver references/infrastructure.md). Este bloco é uma
    // rede de segurança para o caso de algum escapar sem tradução.
    if (exception instanceof Prisma.PrismaClientKnownRequestError) {
      const prismaCodeMap: Record<string, { status: number; title: string }> = {
        P2025: { status: 404, title: 'Not Found' }, // registo não encontrado (update/delete)
        P2002: { status: 409, title: 'Conflict' }, // violação de unicidade
        P2003: { status: 409, title: 'Conflict' }, // violação de foreign key
      };
      const mapped = prismaCodeMap[exception.code];
      if (mapped) {
        return {
          type: `${this.baseUrl}/${this.httpStatusToSlug(mapped.status)}`,
          title: mapped.title,
          status: mapped.status,
          detail:
            exception.code === 'P2002'
              ? 'A record with this data already exists'
              : 'The requested record was not found',
          instance,
          traceId,
        };
      }
    }

    // 4. Erro inesperado — nunca expor detalhes internos
    return {
      type: `${this.baseUrl}/internal-server-error`,
      title: 'Internal Server Error',
      status: HttpStatus.INTERNAL_SERVER_ERROR,
      detail:
        'An unexpected error occurred. Please try again or contact support.',
      instance,
      traceId,
    };
  }

  private formatValidationErrors(
    messages: string | string[],
  ): Record<string, string[]> {
    if (typeof messages === 'string') return { general: [messages] };
    // class-validator devolve "field must be..." — tenta agrupar por campo
    const errors: Record<string, string[]> = {};
    for (const msg of messages) {
      const field = msg.split(' ')[0] ?? 'general';
      errors[field] = [...(errors[field] ?? []), msg];
    }
    return errors;
  }

  private toSlug(code: string): string {
    return code.toLowerCase().replace(/_/g, '-');
  }

  private toTitle(code: string): string {
    return code.replace(/_/g, ' ').replace(/\b\w/g, (c) => c.toUpperCase());
  }

  private httpStatusToSlug(status: number): string {
    const map: Record<number, string> = {
      400: 'bad-request',
      401: 'unauthorized',
      403: 'forbidden',
      404: 'not-found',
      409: 'conflict',
      422: 'unprocessable-entity',
      429: 'too-many-requests',
      500: 'internal-server-error',
    };
    return map[status] ?? 'error';
  }
}
```

### Exemplo de resposta RFC 7807

```json
// 404 — UserNotFoundError
{
  "type": "https://api.bpc.ao/errors/user-not-found",
  "title": "User Not Found",
  "status": 404,
  "detail": "User \"550e8400-e29b-41d4-a716-446655440000\" not found",
  "instance": "/api/v1/users/550e8400-e29b-41d4-a716-446655440000",
  "traceId": "abc123"
}

// 422 — Validação de DTO
{
  "type": "https://api.bpc.ao/errors/unprocessable-entity",
  "title": "Validation Failed",
  "status": 422,
  "detail": "Validation failed",
  "instance": "/api/v1/users",
  "errors": {
    "email": ["email must be an email"],
    "name": ["name must be longer than or equal to 2 characters"]
  }
}

// 422 — InsufficientFundsError
{
  "type": "https://api.bpc.ao/errors/insufficient-funds",
  "title": "Insufficient Funds",
  "status": 422,
  "detail": "Insufficient funds: available 5000 AOA, required 10000 AOA",
  "instance": "/api/v1/payments"
}
```

---

## Erros de Rede em Integrações Externas

### `catch` nunca tipado com um shape customizado

TypeScript só aceita `any` ou `unknown` na variável de um `catch` — um tipo customizado
nessa posição é inválido e só "funciona" com `strict` desligado.

```typescript
// ❌ Inválido — engana o leitor sobre o tipo real
catch (error: { code?: string }) {
  if (error.code === 'ECONNREFUSED') { ... }
}

// ✅ unknown + type guard
catch (error: unknown) {
  if (isAxiosNetworkError(error) && error.code === 'ECONNREFUSED') { ... }
}

function isAxiosNetworkError(
  error: unknown,
): error is { code?: string; message?: string } {
  return typeof error === 'object' && error !== null && 'message' in error;
}
```

### Detectar falha de rede por `error.code`, nunca por `error.message.includes(...)`

Pattern-matching em texto livre é frágil: uma mudança de mensagem do Node/Axios/OpenSSL entre
versões ou sistemas operativos quebra a detecção silenciosamente, sem erro de compilação nem
teste a falhar.

```typescript
// ❌ Frágil
if (
  error.message?.includes('socket disconnected before secure TLS connection') ||
  error.message?.includes('certificate')
) { ... }

// ✅ Mapear pelo código de erro, que é estável
const NETWORK_ERROR_CODES = new Set([
  'ECONNREFUSED', 'ENOTFOUND', 'ECONNRESET', 'ETIMEDOUT', 'EPROTO',
]);

if (NETWORK_ERROR_CODES.has(error.code)) {
  throw new AgtNetworkError(error.code, error.message);
}
```

`error.message` continua útil — mas só como **conteúdo do log**, nunca como **condição de uma
branch de negócio**.

### Erro do parceiro externo propagado como erro tipado, não `BadRequestException` genérica

Um erro de comunicação com a AGT (timeout, TLS, rejeição com `statusCode` + corpo) não é o
mesmo tipo de erro que uma validação de input do nosso domínio. Misturar os dois sob
`BadRequestException` esconde do caller a diferença entre "o utilizador mandou dados
inválidos" e "o parceiro está em baixo" — que normalmente exigem tratamento diferente (retry,
alerta, fallback).

```typescript
// shared/domain/errors/agt-network.error.ts
export class AgtNetworkError extends DomainError {
  constructor(
    public readonly networkCode: string,
    message: string,
  ) {
    super(message, 'AGT_NETWORK_ERROR', 503);
  }
}

export class AgtRejectionError extends DomainError {
  constructor(statusCode: number, responseBody: unknown) {
    super(`AGT rejeitou com status ${statusCode}`, 'AGT_REJECTION', 502);
  }
}
```

O `GlobalExceptionFilter` acima já trata qualquer `DomainError` de forma centralizada — por
isso estes erros devem estender `DomainError`, não `Error` puro, para caírem no mesmo
mapeamento sem precisar de um bloco novo no filtro. Ver `references/external-integrations.md`
para o checklist completo de integrações externas.

---

## Retry com Exponential Backoff

```typescript
// shared/infrastructure/utils/retry.util.ts

export interface RetryOptions {
  maxAttempts?: number;
  baseDelayMs?: number;
  maxDelayMs?: number;
  isRetryable?: (error: unknown) => boolean;
}

export async function withRetry<T>(
  fn: () => Promise<T>,
  options: RetryOptions = {},
): Promise<T> {
  const {
    maxAttempts = 3,
    baseDelayMs = 200,
    maxDelayMs = 10_000,
    isRetryable = isTransientError,
  } = options;

  let lastError: unknown;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err;

      if (attempt === maxAttempts || !isRetryable(err)) {
        throw err;
      }

      // Exponential backoff com jitter
      const exponential = Math.min(
        baseDelayMs * Math.pow(2, attempt - 1),
        maxDelayMs,
      );
      const jitter = Math.random() * exponential * 0.1;
      const delay = exponential + jitter;

      await sleep(delay);
    }
  }

  throw lastError;
}

function isTransientError(err: unknown): boolean {
  if (err instanceof DomainError) return false; // erros de negócio não retentam
  if (err instanceof HttpException) {
    const status = err.getStatus();
    return status >= 500 || status === 429; // server errors e rate limits
  }
  // Erros de rede — mapear por err.code (estável), nunca por err.message.includes(...)
  // (ver "Erros de Rede em Integrações Externas" acima)
  const code = (err as { code?: string })?.code;
  return ['ECONNRESET', 'ETIMEDOUT', 'ECONNREFUSED'].includes(code ?? '');
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));
```

---

## Logging Estruturado com Pino

```typescript
// shared/infrastructure/logger/pino-logger.service.ts
import { Injectable, LoggerService } from '@nestjs/common';
import pino, { Logger } from 'pino';

@Injectable()
export class PinoLoggerService implements LoggerService {
  private readonly logger: Logger;

  constructor() {
    this.logger = pino({
      level: process.env.LOG_LEVEL ?? 'info',
      ...(process.env.NODE_ENV === 'development'
        ? { transport: { target: 'pino-pretty', options: { colorize: true } } }
        : {}),
      serializers: {
        req: pino.stdSerializers.req,
        res: pino.stdSerializers.res,
        err: pino.stdSerializers.err,
      },
      redact: {
        paths: [
          'req.headers.authorization',
          '*.password',
          '*.token',
          '*.secret',
        ],
        censor: '[REDACTED]',
      },
    });
  }

  log(message: string, context?: string, meta?: object): void {
    this.logger.info({ context, ...meta }, message);
  }

  error(message: string, trace?: string, context?: string): void {
    this.logger.error({ context, trace }, message);
  }

  warn(message: string, context?: string): void {
    this.logger.warn({ context }, message);
  }

  debug(message: string, context?: string): void {
    this.logger.debug({ context }, message);
  }
}
```

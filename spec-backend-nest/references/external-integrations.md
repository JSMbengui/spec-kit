# Integrações Externas (APIs de terceiros / governo)

> Regras para qualquer serviço que comunica com uma API externa que a nossa equipa não controla
> (ex.: AGT Facturação Electrónica, BNA, bancos, gateways de pagamento).
> Casos reais de referência: `AgtFeService` (integração com a AGT).

## 1. Um serviço de integração nunca deve acumular todas as operações do parceiro externo

Se a API externa expõe N operações de negócio distintas (ex.: AGT expõe `solicitarSerie`,
`registarFactura`, `listarSeries`, `obterEstado`, `consultarFactura`, `listarFacturas`,
`validarDocumento` — 7 operações), **não criamos um único `@Injectable()` com 7 métodos
públicos**. Isto é a mesma violação de Single Responsibility já identificada em
`PrismaAssemblyRepository` (3 agregados num ficheiro) — aqui são 7 casos de uso externos
num único serviço.

### ❌ Errado

```typescript
@Injectable()
export class AgtFeService {
  async solicitarSerie(input) { ... }
  async registarFactura(input) { ... }
  async listarSeries(input) { ... }
  async obterEstado(input) { ... }
  async consultarFactura(input) { ... }
  async listarFacturas(input) { ... }
  async validarDocumento(input) { ... }
  private async post<T>(url, body) { ... } // cliente HTTP genérico escondido aqui
}
```

### ✅ Correto

Separar em: **um cliente HTTP genérico** (camada de transporte, reutilizável) +
**serviços agrupados por sub-domínio de negócio** (cada um focado, testável isoladamente).

```typescript
// http/agt-http-client.ts — só sabe falar HTTP com a AGT, nada de regras de negócio
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

// services/agt-factura.service.ts
@Injectable()
export class AgtFacturaService {
  constructor(private readonly http: AgtHttpClient) {}
  async registarFactura(input: RegistarFacturaInput): Promise<AgtRegistarFacturaResponse> { ... }
  async consultarFactura(input: ConsultarFacturaInput): Promise<AgtConsultarFacturaResponse> { ... }
  async listarFacturas(input: ListarFacturasInput): Promise<AgtListarFacturasResponse> { ... }
}

// services/agt-estado.service.ts
@Injectable()
export class AgtEstadoService {
  constructor(private readonly http: AgtHttpClient) {}
  async obterEstado(input: ConsultarEstadoFacturaInput): Promise<AgtObterEstadoResponse> { ... }
}

// services/agt-validacao.service.ts
@Injectable()
export class AgtValidacaoService {
  constructor(private readonly http: AgtHttpClient) {}
  async validarDocumento(input: ValidarDocumentoInput): Promise<AgtValidarDocumentoResponse> { ... }
}
```

Critério de divisão: agrupar pelo **agregado/sub-domínio de negócio que a operação afeta**
(série, factura, estado, validação), nunca pelo facto de "todas falam com o mesmo parceiro".
Falar com o mesmo parceiro é responsabilidade do `AgtHttpClient`, não dos serviços de domínio.

## 2. Estrutura de pastas para um módulo de integração externa

```
agt-fe/
├── agt-fe.module.ts
├── http/
│   └── agt-http-client.ts          # POST/GET genérico, TLS, timeout, logging técnico
├── services/
│   ├── agt-serie.service.ts
│   ├── agt-factura.service.ts
│   ├── agt-estado.service.ts
│   └── agt-validacao.service.ts
├── mappers/
│   └── agt-document-type.mapper.ts
├── helpers/
│   └── agt-submission-envelope.ts  # ver regra 3
├── errors/
│   └── agt-network.error.ts        # ver error-handling.md
└── dtos/
    └── agt-*.dto.ts
```

## 3. Geração de "envelope de submissão" (UUID + timestamp) deve ser um único helper, nunca decidida método a método

No `AgtFeService` original, a geração de `submissionUUID`/`submissionTimeStamp` é feita de
**três formas diferentes** ao longo dos 7 métodos:

- `solicitarSerie`: gera um novo UUID, ignora o que vier em `input`.
- `registarFactura`: usa o UUID que vier em `input` (não gera).
- `obterEstado`, `consultarFactura`, `listarFacturas`: geram sempre um novo UUID — e o
  `submissionUUID` recebido em `input`, se existir, é **descartado silenciosamente**.
- `validarDocumento`: usa o UUID recebido em `input`.

Isto é uma regra de negócio (formato/origem do envelope de submissão exigido pela AGT)
implementada de forma inconsistente, com um bug real: chamar `obterEstado` passando um
`submissionUUID` específico não tem efeito nenhum, e nada avisa o caller disso.

### ✅ Correto

```typescript
// helpers/agt-submission-envelope.ts
export interface AgtSubmissionEnvelope {
  submissionUUID: string;
  submissionTimeStamp: string;
}

/**
 * Gera o envelope de submissão da AGT.
 * Se já existir um submissionUUID válido fornecido pelo caller, é reaproveitado
 * (idempotência); caso contrário, gera um novo.
 */
export function buildAgtSubmissionEnvelope(
  existingUUID?: string,
): AgtSubmissionEnvelope {
  return {
    submissionUUID: existingUUID ?? crypto.randomUUID().replace(/-/g, ''),
    submissionTimeStamp: new Date().toISOString().replace(/\.\d{3}Z$/, 'Z'),
  };
}
```

Cada serviço chama sempre o mesmo helper. Se a regra de negócio exigir comportamentos
diferentes por operação (ex.: `registarFactura` *nunca* gera, só aceita), isso deve ser
explícito no nome do parâmetro/assinatura — não uma decisão silenciosa enterrada dentro
do corpo do método.

## 4. Nunca deixar código comentado como "implementação pendente"

```typescript
// ❌ Errado — código morto, não faz nada, mas fica lá como ruído
if (input.deductibleVATPercentage !== undefined) {
  /* body.deductibleVATPercentage = input.deductibleVATPercentage; */
}
```

```typescript
// ✅ Correto — ou implementa, ou remove e documenta com TODO rastreável
if (input.deductibleVATPercentage !== undefined) {
  body.deductibleVATPercentage = input.deductibleVATPercentage;
}
// TODO(JIRA-1234): confirmar com AGT se nonDeductibleAmount é obrigatório
// quando deductibleVATPercentage < 100. Pendente resposta do suporte AGT.
```

## 5. Mapeamento de tipos/enums do parceiro externo: só documentar exceções reais

```typescript
// ❌ Errado — 5 entradas idênticas escondem a única exceção real
export const AGT_DOCUMENT_TYPE_MAP: Record<string, string> = {
  FT: 'FT',
  FR: 'FR',
  NC: 'NC',
  ND: 'ND',
  FG: 'FG',
  PROFORMA: 'PP',
};
```

```typescript
// ✅ Correto — o mapa só documenta onde o nosso código difere do código AGT;
// o resto é validado, não "mapeado"
const AGT_DOCUMENT_TYPE_EXCEPTIONS: Record<string, string> = {
  PROFORMA: 'PP',
};

const AGT_VALID_DOCUMENT_TYPES = ['FT', 'FR', 'NC', 'ND', 'FG', 'PP'] as const;

export function toAgtDocumentType(documentType: string): string {
  return AGT_DOCUMENT_TYPE_EXCEPTIONS[documentType] ?? documentType;
}
```

## 6. Nunca hardcodar configuração de TLS "para testar"

```typescript
// ❌ Errado — comentário admite que é debug, mas está hardcoded
httpsAgent: new https.Agent({
  minVersion: 'TLSv1.2',
  maxVersion: 'TLSv1.2', // força só 1.2 para testar
}),
```

```typescript
// ✅ Correto — configurável por ambiente, com default seguro em produção
httpsAgent: new https.Agent({
  minVersion: this.config.get<string>('AGT_FE_TLS_MIN_VERSION') ?? 'TLSv1.2',
  // sem maxVersion: deixa o Node negociar a versão mais recente suportada,
  // a não ser que exista uma razão documentada (ticket/incidente) para fixar.
}),
```

Se for mesmo necessário fixar `maxVersion` por incompatibilidade conhecida do parceiro,
isso é uma decisão de produção e deve vir com um comentário a referenciar o incidente/ticket
que a justificou — nunca "para testar".

## Checklist rápido para qualquer novo serviço de integração externa

- [ ] Operações de negócio separadas por sub-domínio, não um único deus-serviço
- [ ] Transporte HTTP (`*HttpClient`) isolado da lógica de domínio
- [ ] Geração de envelope/correlação (UUID, timestamp, request-id) num único helper
- [ ] Erros de rede mapeados por `error.code`, nunca por `error.message.includes(...)`
- [ ] Nenhuma configuração de TLS/timeout "temporária" sem flag de ambiente
- [ ] Nenhum código comentado — implementar ou remover com `TODO` rastreável
- [ ] Resposta de erro do parceiro propagada como erro tipado que estende `DomainError`
      (`AgtNetworkError`, `AgtRejectionError`), nunca `BadRequestException` genérica.
      Ver `references/error-handling.md`.

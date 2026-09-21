# Padrões de Instrumentação — CondoHome

Guia de boas práticas para instrumentação de telemetria (métricas, logs, traces) na stack CondoHome.

> ⚠️ **Obrigatório seguir estes padrões em toda instrumentação nova.**

## Princípios Gerais

1. **Baixa cardinalidade:** labels de métricas devem ter poucos valores únicos (< 100)
2. **Safe logs only:** sem secrets, PII sensível, ou dados de alto risco
3. **Soft-fail:** falha de telemetria **nunca** deve bloquear requests de produção
4. **Correlation first:** todo log/métrica deve ser correlacionável via `correlation_id` / `trace_id`
5. **Lean footprint:** instrumentação não deve impactar performance ou memória significativamente

## Métricas (Micrometer OTLP)

### Naming conventions

Seguir [OpenTelemetry semantic conventions](https://opentelemetry.io/docs/specs/semconv/) quando aplicável.

**Formato:** `{namespace}.{component}.{metric}_{unit}`

Exemplos:
- ✅ `charges_import_total` (counter)
- ✅ `charges_import_duration_seconds` (timer/histogram)
- ✅ `charges_import_parse_errors_total` (counter)
- ✅ `http_server_requests_seconds` (Micrometer default)
- ❌ `import_charge` (sem tipo/unidade)
- ❌ `charges_import_org_123_total` (alta cardinalidade)

### Tipos de métricas

| Tipo | Uso | Exemplo |
|------|-----|---------|
| **Counter** | Valores crescentes (soma) | `charges_import_total{result="success"}` |
| **Gauge** | Valores instantâneos | `db_connections_active` |
| **Histogram/Timer** | Distribuição de latências/tamanhos | `charges_import_duration_seconds` |
| **Summary** | Quantis pré-calculados (evitar em favor de histogram) | - |

### Labels obrigatórios

Toda métrica de request HTTP deve ter:
- `method`: GET, POST, PUT, DELETE
- `uri`: **template** (ex.: `/api/v1/charges/{id}` — **não** `/api/v1/charges/abc-123`)
- `status`: código HTTP (200, 400, 500, etc.)
- `outcome`: SUCCESS, CLIENT_ERROR, SERVER_ERROR

Métricas custom devem ter:
- Labels de baixa cardinalidade (status, tipo, resultado enum)
- **Nunca:** `organization_id`, `user_id`, `charge_id` individuais (usar logs para drill-down)

### Labels PROIBIDOS em métricas de alto volume

❌ **NÃO usar** em counters/timers de requests frequentes:
- `organization_id` (pode ter centenas/milhares de tenants)
- `user_id` / `client_id` (exceto em métricas específicas de autenticação agregadas)
- `charge_id`, `import_id`, `condominium_id` (UUIDs únicos)
- `correlation_id` / `trace_id` (únicos por request)
- Query params, request bodies, tokens

**Alternativa:** usar **logs estruturados** com esses IDs + correlation; agregar métricas por rota/status; criar recording rules para top-N orgs se necessário.

### Métricas custom — import uCondo (exemplo)

```java
// Counter — resultado de imports
Counter.builder("charges_import_total")
    .tag("source", "ucondo")
    .tag("result", result.name()) // enum: SUCCESS, FAILURE, PARTIAL
    .register(meterRegistry)
    .increment();

// Timer — duração de import
Timer.builder("charges_import_duration_seconds")
    .tag("source", "ucondo")
    .register(meterRegistry)
    .record(() -> {
        // lógica de import
    });

// Counter — erros de parse
Counter.builder("charges_import_parse_errors_total")
    .tag("error_type", errorType) // enum curto: INVALID_DATE, MISSING_FIELD, etc.
    .register(meterRegistry)
    .increment();
```

**NÃO:**
```java
// ❌ Alta cardinalidade
Counter.builder("charges_import_total")
    .tag("organization_id", organizationId) // ❌ pode ter milhares de valores
    .register(meterRegistry)
    .increment();
```

## Logs Estruturados

### Formato

Preferir **JSON** estruturado (ou logfmt se JSON inviável).

**Campos obrigatórios em todo log de request:**
```json
{
  "timestamp": "2024-01-15T14:30:22.123Z",
  "level": "INFO",
  "logger": "com.condohome.charges.ImportService",
  "message": "Import uCondo completed",
  "correlation_id": "550e8400-e29b-41d4-a716-446655440000",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "organization_id": "org-123",
  "condominium_id": "condo-456",
  "import_id": "imp-789",
  "route": "/api/v1/charges/imports/ucondo",
  "method": "POST",
  "status": 201,
  "duration_ms": 1234,
  "outcome": "SUCCESS"
}
```

### Campos permitidos

| Campo | Notas |
|-------|--------|
| `correlation_id` | UUID; aceito via `X-Correlation-Id` header ou gerado |
| `trace_id`, `span_id` | W3C Trace Context (`traceparent`) |
| `organization_id` | Tenant SaaS |
| `condominium_id` | Sub-entidade |
| `client_id` | ID público OAuth/API (não secret) |
| `import_id`, `charge_id` | UUIDs de recursos |
| `error_code` | Código de erro estável (enum) |
| `route`, `method`, `status` | HTTP metadata |
| `duration_ms` | Latência request |
| `outcome` | SUCCESS, CLIENT_ERROR, SERVER_ERROR |

### Campos PROIBIDOS

❌ **NUNCA logar:**
- `client_secret`
- JWT/Bearer tokens completos (max: últimos 4 chars se necessário debug)
- `password`, `api_key`, `refresh_token`
- CPF/CNPJ em texto claro (hash ou últimos 4 dígitos se inevitável)
- Nomes/emails de moradores em logs de alto volume (só em logs de erro específico, sem repetição)
- Bodies de import xlsx (PII sensível)
- SQL queries com valores sensíveis
- Senhas de DB, tokens Resend/WhatsApp, credentials de serviços externos

### Níveis de log

| Nível | Uso |
|-------|-----|
| **TRACE** | Debug detalhado (desabilitado em prod) |
| **DEBUG** | Informação de debug (desabilitado em prod) |
| **INFO** | Fluxo normal da aplicação (imports, requests bem-sucedidos) |
| **WARN** | Situações anômalas não-críticas (retry, fallback, validação falhada) |
| **ERROR** | Erros que exigem atenção (5xx, exceptions, falhas de integração) |

**Produção:** INFO em diante (DEBUG/TRACE desabilitados).

### Exemplo — import bem-sucedido

```java
log.info("Import uCondo completed",
    kv("correlation_id", correlationId),
    kv("organization_id", organizationId),
    kv("import_id", importId),
    kv("rows_total", rowsTotal),
    kv("rows_matched", rowsMatched),
    kv("rows_unmatched", rowsUnmatched),
    kv("duration_ms", durationMs),
    kv("outcome", "SUCCESS")
);
```

### Exemplo — erro de parse

```java
log.error("Import uCondo parse error",
    kv("correlation_id", correlationId),
    kv("organization_id", organizationId),
    kv("import_id", importId),
    kv("error_code", "INVALID_DATE"),
    kv("row_number", rowNumber),
    kv("outcome", "FAILURE")
    // ❌ NÃO incluir conteúdo da linha (pode ter PII)
);
```

## Correlation (W3C Trace Context)

### Headers

Aceitar/propagar em **toda request/response HTTP:**

1. **`traceparent`** (W3C Trace Context):
   - Formato: `00-<trace-id>-<span-id>-<flags>`
   - Exemplo: `00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`
   - Se ausente: gerar novo `trace_id` (128-bit hex) e `span_id` (64-bit hex)

2. **`X-Correlation-Id`** (fallback/complementar):
   - UUID v4
   - Se ausente: gerar
   - **Sempre** ecoar na response

### Implementação na API

1. **Filter/Interceptor:**
   ```java
   String correlationId = request.getHeader("X-Correlation-Id");
   if (correlationId == null) {
       correlationId = UUID.randomUUID().toString();
   }
   
   String traceparent = request.getHeader("traceparent");
   String traceId = extractTraceId(traceparent); // parse W3C
   
   MDC.put("correlation_id", correlationId);
   MDC.put("trace_id", traceId);
   
   response.setHeader("X-Correlation-Id", correlationId);
   ```

2. **Outgoing calls (n8n, Resend, etc.):**
   ```java
   httpClient.send(request
       .header("X-Correlation-Id", MDC.get("correlation_id"))
       .header("traceparent", buildTraceparent()) // propagar W3C
   );
   ```

3. **MDC cleanup:**
   ```java
   finally {
       MDC.clear();
   }
   ```

## Actuator / Health

### Endpoints Actuator

Expor **apenas rede interna:**
- `/api/v1/actuator/health` — health checks (Docker probes)
- `/api/v1/actuator/prometheus` — métricas Prometheus (scrape interno)
- `/api/v1/actuator/info` — informação da aplicação (versão, build)

**Traefik DEVE BLOQUEAR** `/api/v1/actuator/**` de acesso público (middleware deny ou sem router).

### Health checks

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus,info
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

**Docker healthcheck:**
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/api/v1/actuator/health/liveness"]
  interval: 30s
  timeout: 5s
  retries: 3
```

**NÃO** incluir secrets ou dados sensíveis em health details (ex.: connection strings, tokens).

## Alertas (fase 2)

Quando alertas Prometheus forem ativados:

### Naming conventions

`{severity}_{component}_{condition}`

Exemplos:
- `critical_api_down`
- `warning_import_failure_rate_high`
- `critical_disk_space_low`

### Severidades

| Severidade | Uso | Ação |
|------------|-----|------|
| **critical** | Indisponibilidade, data loss, security breach | Page on-call imediato |
| **warning** | Degradação, SLO próximo do limite | Ticket, investigar em 24h |
| **info** | Mudanças planejadas, deploys | Log, sem ação |

## Dashboards Grafana

### Naming conventions

`{Produto} — {Funcionalidade}`

Exemplos:
- ✅ `CondoHome — API RED + Import`
- ✅ `CondoHome — Infra (VPS)`
- ❌ `Dashboard 1` (vago)

### Estrutura recomendada

1. **Overview (RED):**
   - Rate (requests/s)
   - Errors (5xx rate, error rate)
   - Duration (p50, p95, p99 latency)

2. **Functional metrics:**
   - Import: success rate, duration, parse errors
   - Auth: 401/403 rate, RLS denials
   - Automation: execution success rate

3. **Infra:**
   - JVM: heap, GC pause, threads
   - DB: connections, slow queries
   - Disco: usage VPS

4. **Logs (panel Loki):**
   - Filtros rápidos: `{job="docker"} | json | correlation_id="..."`
   - Links para trace (fase 2)

## Checklist de Review

Antes de merge de instrumentação nova:

- [ ] Métricas: labels de baixa cardinalidade (< 100 valores)
- [ ] Métricas: sem `organization_id`, `user_id`, `charge_id` em counters de alto volume
- [ ] Logs: JSON estruturado com `correlation_id`, `trace_id`
- [ ] Logs: sem secrets, JWT completos, CPF/CNPJ claro, PII sensível
- [ ] Correlation: aceitar/gerar/propagar `X-Correlation-Id` e `traceparent`
- [ ] Health: detalhes não expõem secrets ou dados sensíveis
- [ ] Actuator: **não** exposto publicamente (verificar Traefik)
- [ ] Soft-fail: falha de telemetria não bloqueia requests (try-catch, circuit breaker)
- [ ] Testes: smoke test de métricas/logs em ambiente de staging

## Governança

- **Obrigatório** seguir estes padrões em toda instrumentação nova
- Mudanças de padrões: revisar com Arquiteto + OK do Rodrigo
- Violações de safe logs (PII vazada): **incident P1** — remediar imediatamente

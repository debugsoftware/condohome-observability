# Checklist de observabilidade — 5 fluxos críticos (CondoHome piloto)

> Stack atual: Prometheus + Grafana + Loki + Alloy + OTel Collector (métricas + logs).  
> Traces/Tempo = **fase 2**. Sem deploy/merge prd sem ok do Rodrigo.  
> Labels: **baixa cardinalidade**. Não colocar `organization_id` em todo counter se o nº de tenants for alto — preferir logs + amostragem ou métricas agregadas por rota/status.

**Legenda de gaps**
- **[API PR]** — exige mudanças Micrometer/logging na `condohome-api`
- **[N8N]** — exige propagação de correlation no workflow / Developer n8n
- **[OPS]** — configuração Traefik/Docker/VPS

---

## 1. Import uCondo — `POST .../charges/imports/ucondo`

### O que logar (ok)
| Campo | Notas |
|-------|--------|
| `organization_id`, `condominium_id` | OK em log estruturado |
| `import_id`, `client_id` (público) | OK |
| `trace_id` / `correlation_id` | W3C `traceparent` e/ou `X-Correlation-Id` |
| `route`, `method`, `status`, `duration_ms` | Sempre |
| `error_code`, contagens parse/match | Sem CPF/CNPJ claro; sem body xlsx |

### Nunca logar
- Body do arquivo xlsx / linhas brutas com PII
- CPF/CNPJ em texto claro (usar hash ou só últimos dígitos se inevitável)
- Nomes/emails de moradores em alto volume

### Métricas (piloto)
| Métrica | Tipo | Labels sugeridos (baixa card.) |
|---------|------|--------------------------------|
| `charges_import_total` | counter | `status` (success/partial/fail), `source=ucondo` |
| `charges_import_duration_seconds` | histogram/timer | `source` |
| `charges_import_parse_errors_total` | counter | `error_code` (enum curto) |
| (opcional) `charges_import_rows_total` | counter | `result` (matched/unmatched/skipped) — **[API PR]** |

### Alertas fase 1
1. **Import fail spike**: `rate(charges_import_total{status="fail"}[15m])` > limiar ou `> 0` sustentado 10m
2. **Parse errors**: `increase(charges_import_parse_errors_total[30m]) > N`
3. **Duração alta**: p95 `charges_import_duration_seconds` > 60s (ajustar ao piloto)
4. **Timeouts 5xx** na rota de import via RED HTTP

### SLO draft
- Disponibilidade da rota import: **99%** success HTTP (não 5xx) em janela 7d
- p95 duração import < **45s** (ajustar após baseline piloto)
- Taxa parse_error / total_rows < **2%** (quando métrica de rows existir)

### Gaps
- **[API PR]** Instrumentar counters/timers Micrometer com nomes acima; emitir OTLP
- **[API PR]** Log estruturado com `import_id` + `error_code` sem PII
- **[OPS]** Confirmar que Prometheus scrape da API é só rede Docker

---

## 2. Cobranças — `GET .../charges`

### O que logar
- `organization_id`, `condominium_id` (em log, não em todo counter)
- `route`, `method`, `status`, `duration_ms`, `trace_id`/`correlation_id`
- `error_code` em 4xx/5xx
- Paginação: `page_size` (não listar IDs de charges em massa)

### Métricas
| Métrica | Labels |
|---------|--------|
| `http_server_requests_*` (Micrometer) | `method`, `uri` (template!), `status`, `outcome` |
| Evitar | `organization_id` como label em high-volume counters |

Para latência/erros **por tenant** no piloto: usar **logs** + LogQL agregando `organization_id`, ou métrica amostrada / recording rule com top-N — não label em todo request se scale for alto. **[API PR]** decidir política.

### Alertas fase 1
1. **p95 latência** GET charges > 1s por 10m
2. **Taxa 5xx** > 1% em 15m
3. **Taxa 4xx** anômala (ex. sudden spike 400/404) — investigar contrato/cliente
4. (fase 1 soft) 403 elevados — correlacionar com fluxo Auth/RLS

### SLO draft
- Disponibilidade (não-5xx): **99.5%** / 30d
- p95 latência: **< 800ms** (baseline a calibrar no piloto)

### Gaps
- **[API PR]** URI templates no Micrometer (evitar path params como label)
- **[API PR]** Correlation header em todas as respostas de erro

---

## 3. Auth tech + RLS — 401/403 e vazamento cross-tenant

### O que logar
- `client_id` público, `organization_id` alvo da request
- `status` 401/403, `error_code` (ex. `UNAUTHENTICATED`, `FORBIDDEN`, `TENANT_MISMATCH`)
- `route`, `method`, `correlation_id`
- **Nunca**: JWT/Bearer completo, `client_secret`, refresh tokens

### Métricas
| Métrica | Notas |
|---------|--------|
| `http_server_requests_*` com status 401/403 | Por `uri` template |
| `security_auth_failures_total` **[API PR]** | `reason` enum curto |
| `security_rls_denials_total` **[API PR]** | `reason=cross_tenant` etc. |

### Alertas fase 1 (críticos)
1. **Cross-tenant leak attempt**: qualquer log/`error_code=TENANT_MISMATCH` ou denial RLS com evidência de outro tenant → alerta **P1** imediato
2. **Spike 401**: possível ataque / cliente mal configurado
3. **Spike 403** em rotas sensíveis após deploy

### SLO draft
- Zero incidentes confirmados de leitura cross-tenant
- Tempo para detectar tentativa: **< 5 min** (alerta + dashboard)

### Gaps
- **[API PR]** Códigos de erro estáveis e métrica dedicada RLS
- **[API PR]** Garantir que respostas 403 não vazam existência de recursos de outro tenant
- **[OPS]** Actuator nunca público (evita info leak + superfície de ataque)

---

## 4. automation-config + executions (n8n)

### O que logar
- `organization_id`, `condominium_id`, `execution_id` / `workflow_id`
- `correlation_id` **propagado** da API → n8n → callbacks
- Resultado: `success`/`fail`, `error_code`, `duration_ms`
- Idempotência: `idempotency_key` ou hash de trigger (sem payload sensível)

### Métricas
| Métrica | Fonte |
|---------|--------|
| `automation_execution_total` | **[API PR]** e/ou **[N8N]** |
| `automation_execution_duration_seconds` | idem |
| Success rate | `success / total` |

### Alertas fase 1
1. Success rate n8n < **95%** em 1h
2. Retriggers idempotentes duplicados acima do esperado (mesmo `idempotency_key` com side-effect)
3. Fila/timeout de executions (se exposto)

### SLO draft
- Success rate automations: **≥ 98%** / 7d (excluindo falhas de configuração do condomínio)
- Idempotência: **0** side-effects duplicados em retrigger

### Gaps
- **[N8N]** Propagar `X-Correlation-Id` / `traceparent` em todos os nós HTTP
- **[N8N]** Não logar tokens Resend/WhatsApp nem bodies com PII
- **[API PR]** Métricas de automation-config/executions se a API orquestra

---

## 5. Infra — Flyway, health, JVM, Postgres (VPS)

### O que logar / observar
- Flyway migrate: sucesso/falha no startup (log + exit code do container)
- Health: `GET /api/v1/actuator/health` **somente rede interna**
- JVM (Micrometer): heap, GC pause, threads — via OTLP ou scrape Actuator interno
- Postgres: conexões, slow queries (sem logar SQL com PII), disk VPS

### Métricas
| Alvo | Como |
|------|------|
| `up{job="condohome-api"}` | Prometheus scrape |
| Health details | Blackbox interno ou script — **não** expor Traefik público |
| JVM Micrometer | `jvm_memory_*`, `jvm_gc_*` |
| Disco Loki/Prometheus | volume Docker + alerta disk VPS **[OPS]** |

### Alertas fase 1
1. API scrape `up == 0` por 2m
2. Health DOWN (se probe interno existir)
3. Flyway fail no boot (restart loop) — alerta em logs Alloy
4. Disco VPS > **80%** (Loki+Prom TSDB)
5. Heap / GC anômalo (após baseline)

### SLO draft
- Uptime API piloto: **99%** / 7d (janela mantida)
- RTO detect health down: **< 3 min**

### Gaps
- **[OPS]** Traefik: bloquear `/api/v1/actuator/**` no entrypoint público
- **[OPS]** Rede: API + obs na mesma Docker network (ou attach)
- **[API PR]** Micrometer OTLP registry apontando para `otel-collector:4318`
- Capacidade VPS **não verificada neste artefato** — calibrar mem_limit do compose no host real

---

## Matriz rápida — quem faz o quê

| Item | API PR (Micrometer/logs) | N8N Dev | OPS (Traefik/Docker) |
|------|--------------------------|---------|----------------------|
| RED HTTP + import metrics | ✅ | | |
| Correlation headers | ✅ | ✅ propagar | |
| Redação PII em logs | ✅ + Alloy | ✅ | |
| Actuator interno only | | | ✅ |
| Alertas Grafana/Prometheus | | | ✅ (fase 1 rules) |
| Tempo/traces | fase 2 | fase 2 | fase 2 |

---

## Próximos passos sugeridos (piloto)

1. Subir stack em VPS de homologação (`docker compose up -d`)
2. Conectar API à rede `condohome-obs` e apontar OTLP
3. Validar dashboard RED + painéis de import (mesmo com métricas zeradas)
4. Abrir PRs API para nomes de métricas deste checklist
5. **Não** promover a produção sem ok explícito do Rodrigo

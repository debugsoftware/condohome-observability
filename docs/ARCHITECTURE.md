# Arquitetura de Observabilidade — CondoHome Piloto

Stack lean para observabilidade do piloto CondoHome (SaaS multi-tenant) em **Hostinger VPS com Docker + Traefik** e edge Cloudflare.

> ⚠️ **Sem deploy em produção sem ok do Rodrigo.**

## Visão Geral

```
┌─────────────────────────────────────────────────────────────────┐
│                         Internet                                 │
│                            ↓                                     │
│                  Cloudflare Edge / WAF                           │
│                            ↓                                     │
│                    Traefik (VPS)                                 │
│          (rotas públicas; BLOQUEIA /actuator/*)                  │
└──────────────────────────────┬──────────────────────────────────┘
                               ↓
┌──────────────────────────────────────────────────────────────────┐
│  CondoHome API (Spring Boot Modulith)                            │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Micrometer OTLP Registry → otel-collector:4318 (HTTP)    │ │
│  │  + Actuator /health, /prometheus (só rede interna)        │ │
│  │  + MDC: correlation_id, trace_id, organization_id         │ │
│  └────────────────────────────────────────────────────────────┘ │
│         ↓ OTLP/HTTP                      ↓ Scrape (opcional)    │
└─────────┼──────────────────────────────────┼────────────────────┘
          ↓                                  ↓
┌─────────────────────────────────────────────────────────────────┐
│  Docker Network: condohome-obs (bridge)                          │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  OTel Collector                                          │   │
│  │  • Receiver: OTLP gRPC (4317), OTLP HTTP (4318)         │   │
│  │  • Processor: batch, memory_limiter, attributes, resource│  │
│  │  • Exporters:                                            │   │
│  │    - Prometheus (exporter :8889 — scrape endpoint)      │   │
│  │    - Loki (via OTLP HTTP push)                          │   │
│  └────────┬──────────────────────────────┬─────────────────┘   │
│           │ scrape :8889                 │ push OTLP/HTTP      │
│           ↓                              ↓                      │
│  ┌─────────────────┐            ┌──────────────────┐           │
│  │  Prometheus     │            │  Loki             │           │
│  │  • TSDB 15d     │            │  • Filesystem     │           │
│  │  • 2GB cap      │            │  • Retenção 14d   │           │
│  │  • Scrape       │            │  • TSDB v13       │           │
│  │    - otel:8889  │            └──────────────────┘           │
│  │    - api:8080   │                    ↑                       │
│  │      /actuator  │                    │ push logs             │
│  └─────────────────┘                    │                       │
│           ↑                     ┌──────────────────┐            │
│           │                     │  Alloy            │            │
│           │                     │  • Docker logs    │            │
│           │                     │  • PII redaction  │            │
│           │                     │  • Correlation    │            │
│           │                     └──────────────────┘            │
│           │                              ↑                       │
│           │                              │ read containers       │
│           │                              │ /var/lib/docker/...   │
│           │                                                      │
│           └──────────────┬───────────────────────────────────┐  │
│                          ↓                                    ↓  │
│                   ┌─────────────────────────────────────────────┤
│                   │  Grafana (UI)                               │
│                   │  • Datasources: Prometheus, Loki            │
│                   │  • Dashboard: CondoHome API RED + Import    │
│                   │  • Correlation links: Log ↔ Metric          │
│                   │  • Port :3000 (host) — proteger com auth!   │
│                   └─────────────────────────────────────────────┘
└───────────────────────────────────────────────────────────────────┘
```

## Componentes

### 1. CondoHome API

**Função:** Aplicação SaaS multi-tenant (Spring Boot Modulith).

**Telemetria:**
- **Métricas:** Micrometer nativo com registry OTLP → `otel-collector:4318/v1/metrics`
- **Logs:** Estruturados (JSON preferível) com MDC: `correlation_id`, `trace_id`, `organization_id`, `error_code`, etc.
- **Health:** `/api/v1/actuator/health` — **só rede interna** (probes Docker)
- **Prometheus endpoint:** `/api/v1/actuator/prometheus` — scrape interno opcional

**Segurança:**
- Traefik **DEVE BLOQUEAR** acesso público a `/api/v1/actuator/**`
- Soft-fail se OTel Collector indisponível (métricas não bloqueiam requests)
- Safe logs: sem secrets, JWT completos, CPF/CNPJ claro, PII sensível

**Labels baixa cardinalidade:**
- ✅ `method`, `uri` (template), `status`, `outcome`, `error_code`
- ⚠️ **NÃO** `organization_id` em counters de alto volume — preferir logs/agregações

### 2. OTel Collector

**Função:** Gateway de telemetria — recebe OTLP, processa e exporta.

**Receivers:**
- OTLP gRPC `:4317`
- OTLP HTTP `:4318` (usado pela API)

**Processors:**
- `batch`: agrupa exportações (512 items / 5s)
- `memory_limiter`: protege de OOM (200 MiB / spike 50 MiB)
- `attributes/metrics_sanitize`: remove labels sensíveis (`http.request.body`, `db.statement`, `client_secret`, `authorization`)
- `resource`: adiciona `service.namespace=condohome`

**Exporters:**
- **Prometheus:** expõe endpoint de scrape em `:8889` (Prometheus pull)
- **Loki:** push OTLP/HTTP para `loki:3100/otlp`
- **Sem Tempo** nesta fase (traces = fase 2)

**Limites:** 256 MiB RAM, 0.5 CPU

### 3. Prometheus

**Função:** Time-series database para métricas.

**Scrape targets:**
1. `otel-collector:8889` — métricas exportadas pelo Collector (fonte principal)
2. `otel-collector:8888` — self-metrics do Collector
3. `condohome-api:8080/api/v1/actuator/prometheus` — scrape direto da API (opcional, se OTLP falhar)
4. `localhost:9090` — self-metrics do Prometheus

**Config:**
- Intervalo: 30s (ajustável)
- Retenção: **15 dias**
- Cap: **2 GB** TSDB
- External labels: `project=condohome`, `env=pilot`

**Limites:** 512 MiB RAM, 0.5 CPU

### 4. Loki

**Função:** Log aggregation e storage.

**Ingestão:**
- Alloy → push logs de containers Docker
- OTel Collector → logs via OTLP/HTTP (se API enviar logs OTLP — opcional na fase 1)

**Storage:**
- Filesystem: `/loki` (volume Docker)
- Schema: TSDB v13
- Retenção: **14 dias** (336h)
- Compaction: 10 min intervals

**Correlation:**
- Datasource Grafana: derived fields para `trace_id` e `correlation_id`
- Permite queries por correlation id para debug end-to-end

**Limites:**
- 384 MiB RAM, 0.4 CPU
- Ingestion: 4 MB/s, burst 8 MB/s
- Max entries per query: 5000

**Disco:** ~1–5 GB estimado no piloto (monitorar VPS)

### 5. Alloy (Grafana Agent v2)

**Função:** Coleta logs de containers Docker e push para Loki.

**Discovery:**
- `discovery.docker` — monitora `/var/run/docker.sock`
- Label containers por nome e stream (stdout/stderr)

**Processing:**
- Drop health check noise: `GET /actuator/health`, `/healthz`, `/ready`
- **PII redaction:**
  - Bearer tokens → `***REDACTED***`
  - `client_secret` → `***REDACTED***`
  - `password` → `***REDACTED***`
  - CPF: `###.###.###-##` → `***CPF***`
  - CNPJ: `##.###.###/####-##` → `***CNPJ***`
  - Emails (opcional, comentado por padrão)

**Push:** `loki:3100/loki/api/v1/push`

**Limites:** 192 MiB RAM, 0.3 CPU

### 6. Grafana

**Função:** UI de observabilidade — dashboards, explore, alertas (fase 2).

**Datasources:**
- **Prometheus** (default): `http://prometheus:9090`
- **Loki**: `http://loki:3100`
- **Sem Tempo** nesta fase

**Provisioning:**
- Datasources: `grafana/provisioning/datasources/datasources.yml`
- Dashboards: `grafana/provisioning/dashboards/dashboards.yml`
- Dashboard inicial: `CondoHome — API RED + Import` (`grafana/dashboards/condohome-api-red.json`)

**Exposição:**
- Port `:3000` (host) — **PROTEGER com Traefik Basic Auth / OAuth / VPN**
- Admin user/senha: `.env` (`GRAFANA_ADMIN_USER`, `GRAFANA_ADMIN_PASSWORD`)
- **Nunca** expor sem autenticação na internet

**Limites:** 256 MiB RAM, 0.3 CPU

## Fluxo de Dados

### Métricas

```
API (Micrometer OTLP)
  ↓ OTLP/HTTP :4318
OTel Collector
  ↓ Process (batch, sanitize, resource)
  ↓ Export to Prometheus exporter :8889
Prometheus (scrape :8889)
  ↓ Store TSDB (15d / 2GB)
Grafana (query Prometheus datasource)
  ↓ Dashboard / Explore
User
```

### Logs

```
Containers Docker (API, n8n, etc.)
  ↓ /var/lib/docker/containers/
Alloy (discovery.docker)
  ↓ Process (drop health, PII redaction)
  ↓ Push to Loki :3100
Loki (filesystem TSDB)
  ↓ Store (14d)
Grafana (query Loki datasource)
  ↓ Explore / Logs panel
User
```

### Correlation (end-to-end)

```
HTTP Request → API
  ↓ Accept/Generate X-Correlation-Id ou traceparent
  ↓ MDC: correlation_id, trace_id
API Log → JSON estruturado {correlation_id, trace_id, ...}
  ↓ Alloy → Loki
API Métrica → labels {method, uri, status} + traces {correlation_id}
  ↓ OTLP → Collector → Prometheus
Grafana:
  - Dashboard: métricas RED por rota
  - Explore Loki: filtrar logs por correlation_id
  - Derived field: link Log → (fase 2: Trace)
```

## Rede Docker

**Nome:** `condohome-obs`  
**Driver:** bridge  
**Membros:**
- otel-collector
- prometheus
- loki
- alloy
- grafana
- **condohome-api** (conectar via `docker network connect condohome-obs <api-container>`)
- n8n (opcional, se rodar no mesmo VPS)

**Portas internas:**
- Todos os serviços comunicam via hostnames Docker: `otel-collector`, `prometheus`, `loki`, etc.
- **Nenhuma** porta de obs (exceto Grafana :3000) deve ser exposta no firewall do VPS

**Portas externas (host):**
- Grafana `:3000` — **única porta exposta**, proteger com Traefik + auth

## Segurança

### 1. Actuator NUNCA público

- `/api/v1/actuator/**` **DEVE** estar bloqueado no Traefik (middleware deny ou sem router público)
- Scrape Prometheus: **só rede Docker interna** (`condohome-api:8080`)
- Health probes: **só rede Docker** (Docker healthcheck ou Traefik interno)

### 2. Grafana protegido

- Senha forte no `.env` (`GRAFANA_ADMIN_PASSWORD`)
- Preferencialmente: Traefik Basic Auth / OAuth / VPN / IP allowlist
- **Nunca** expor Grafana sem autenticação na internet

### 3. Safe logs (PII redaction)

**Na API (responsabilidade dev):**
- ✅ `organization_id`, `condominium_id`, `import_id`, `charge_id`, `error_code`, `route`, `method`, `status`, `duration_ms`
- ❌ `client_secret`, JWT completos, CPF/CNPJ claro, nomes/emails moradores em alto volume, bodies xlsx, senhas DB, tokens Resend/WhatsApp

**No Alloy (camada de defesa):**
- Redação automática de Bearer tokens, `client_secret`, `password`, CPF, CNPJ
- Regex patterns em `alloy/config.alloy` — ajustar conforme formato real dos logs

### 4. Firewall VPS

Bloquear portas (iptables / ufw):
- 4317, 4318 (OTel Collector)
- 8888, 8889 (Collector self/exporter)
- 9090 (Prometheus)
- 3100 (Loki)
- 12345 (Alloy)

Permitir:
- 3000 (Grafana) — **apenas se protegido** (preferir Traefik interno + VPN)
- 80, 443 (Traefik → API pública)

## Escalabilidade e Limites

### Piloto (2–4 GB VPS)

**RAM total alocada:** ~1.5 GB (limites somados no compose)
- API: ~512 MB–1 GB (fora do compose obs)
- Obs stack: ~1.5 GB
- **Total:** ~2.5 GB mínimo recomendado (com margem para SO + buffers)

**CPU:** ~2.3 cores alocados (limites somados) — ajustar conforme VPS

**Disco:**
- Prometheus TSDB: ~500 MB–1 GB (15d / 2GB cap)
- Loki: ~1–5 GB (14d, depende do volume de logs)
- **Monitorar:** alertar se disco VPS > 80%

### Limitações do piloto

- **Sem alta disponibilidade:** single-node, sem replicação
- **Sem backups automáticos:** dados Prometheus/Loki em volumes locais
- **Sem Tempo:** traces = fase 2
- **Sem Cloudflare Logpush:** edge logs não ingeridos nesta fase
- **Sem alertas ativos:** Prometheus rules vazias (fase 2)

**Capacidade real do VPS não foi validada por estes artefatos.**

## Fase 2 (Traces)

Quando aprovado:

1. **Adicionar Tempo:**
   - Container `grafana/tempo:latest` no compose
   - Volume para traces (retenção ajustável)
   - Datasource Tempo no Grafana

2. **Pipeline traces no OTel Collector:**
   ```yaml
   traces:
     receivers: [otlp]
     processors: [memory_limiter, batch]
     exporters: [otlp/tempo]
   ```

3. **API: Micrometer Tracing:**
   - Dependência: `micrometer-tracing-bridge-brave` ou `micrometer-tracing-bridge-otel`
   - Registry: enviar spans via OTLP ao Collector
   - Correlation: `trace_id` / `span_id` automático

4. **Grafana:**
   - Links Log → Trace (via `trace_id` derived field)
   - Service map / trace timeline

**Não adicionar Tempo até fase 2 aprovada.**

## Governança

- Stack aprovada H4.1: métricas + correlation/MDC; sem span export nesta fase
- Mudanças de stack / config: revisar com Arquiteto + OK do Rodrigo
- Promoção a PRD: **obrigatório OK do Rodrigo**
- Monitorar recursos VPS: CPU, RAM, disco — ajustar limites conforme baseline

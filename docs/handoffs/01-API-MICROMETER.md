# 01 — API Micrometer → OTLP (Arquiteto de Soluções)

**Track:** Arquiteto de Soluções  
**Repo alvo:** `condohome-api`  
**Gate:** PR review nesta track; **sem merge/deploy sem OK do Rodrigo**

## Objetivo

Expor telemetria da API via **Micrometer nativo** com registry **OTLP** apontando para `otel-collector:4318` (HTTP).  
**Não** usar Java agent.

## Checklist — dependências / config

- [ ] Dependências Micrometer OTLP (ex.: `micrometer-registry-otlp`) + Actuator
- [ ] Config (env / `application-*.yml`):
  - OTLP endpoint: `http://otel-collector:4318` (ou path `/v1/metrics` conforme registry)
  - Service name: `condohome-api`
  - Intervalo de export razoável (ex.: 15–30s)
- [ ] Garantir que a API resolve o hostname `otel-collector` na rede Docker do compose de obs

## Actuator

- [ ] Endpoint Prometheus (se mantido para scrape local/debug): `/api/v1/actuator/prometheus` — **só rede interna**
- [ ] Health / readiness: `/api/v1/actuator/health` (e readiness se existir)
  - **Nunca expor publicamente** (Traefik/reverse proxy bloqueia — ver `03-OPS-PILOTO.md`)
  - Scrape / probes **somente** na rede interna
- [ ] Não logar tokens, secrets ou PII em health details

## Structured logging (campos mínimos)

Incluir nos logs estruturados (JSON preferível), quando disponíveis:

| Campo | Notas |
|-------|--------|
| `correlation_id` | valor de `X-Correlation-Id` / gerado |
| `trace_id` | de W3C `traceparent` quando presente |
| `span_id` | opcional |
| `organization_id` | **só em logs de request**, não em labels de métricas de alto volume |
| `request_path` / `http_method` | sem query strings com tokens |
| `outcome` / `error_code` | códigos estáveis, sem stack completa com PII |

**Safe logs only:** sem secrets, tokens, senhas, CPF, e-mail completo desnecessário, payloads sensíveis.

## Métricas custom — import uCondo

Baixa cardinalidade. **Não** colocar `organization_id` em todo counter.

| Métrica | Tipo | Labels |
|---------|------|--------|
| `charges_import_total` | counter | `result` (ex.: `success`, `failure`, `partial`) |
| `charges_import_duration_seconds` | timer/histogram | mínimo necessário (evitar org_id) |
| `charges_import_parse_errors_total` | counter | preferir label estável tipo `error_type` se preciso; sem org_id |

## Correlation filter / interceptor

- [ ] Aceitar W3C `traceparent` e/ou header `X-Correlation-Id`
- [ ] Se ausente: **gerar** correlation id (UUID)
- [ ] Ecoar nos headers de resposta (`X-Correlation-Id`; respeitar/`traceparent` conforme propagação)
- [ ] Disponibilizar no MDC / contexto de log e para outgoing calls (quando API chamar outros serviços)
- [ ] n8n tech client deve propagar (handoff: `02-N8N-CORRELATION.md`) — API não assume que sempre vem preenchido

## Critérios de aceite

1. Com compose de obs no ar e API na mesma rede: métricas da API aparecem no Prometheus (via OTel Collector) ou scrape acordado
2. Em Grafana: dashboard/explore mostra `charges_import_*` após um import de teste
3. Request sem headers → resposta traz `X-Correlation-Id`; logs da mesma request têm o mesmo id
4. Request com `X-Correlation-Id` / `traceparent` → valor propagado/ecoado; aparece nos logs
5. `/api/v1/actuator/health` **não** responde via URL pública (verificado com OPS)
6. Nenhum secret/PII em logs de sample das rotas críticas

## Como verificar

```text
# Prometheus (UI ou API)
# query: charges_import_total
# query: charges_import_duration_seconds_bucket (ou sum)

# Grafana Explore → Prometheus / Loki
# Loki: filtrar por correlation_id=...
```

Fluxos a exercitar: import uCondo, GET charges, auth+RLS, automation-config/executions, health/JVM.

## Explicit

- PR na track do **Arquiteto de Soluções**
- **Não mergear / não deployar** sem OK do Rodrigo

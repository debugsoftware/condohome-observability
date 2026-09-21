# 03 — OPS Piloto (Hostinger VPS)

**Só após OK do Rodrigo.** Não deployar antes.

Pré-requisito: repo **condohome-observability** com compose validado.

## Checklist — subir stack

- [ ] Clone/pull `condohome-observability` no VPS
- [ ] Secrets/env: Grafana admin, sem defaults fracos em produção
- [ ] `docker compose up -d` (OTel Collector, Prometheus, Grafana, Loki, Alloy)
- [ ] Confirmar containers healthy / ports só internas onde aplicável

## Rede — API ↔ obs

- [ ] Rede Docker compartilhada (ex.: `condohome-obs`)
- [ ] Conectar container da API à rede: resolve `otel-collector:4318`
- [ ] Confirmar que Collector recebe OTLP da API (logs do collector / métricas no Prometheus)

## Traefik — bloquear actuator

- [ ] **Bloquear** acesso público a `/api/v1/actuator/**` (health, prometheus, etc.)
- [ ] Probes/scrapes apenas na rede interna ou via sidecar/mesh interno
- [ ] Testar de fora: URL pública do actuator deve falhar (403/404/timeout — não 200)

## Grafana — auth / proteção

- [ ] Auth obrigatória (admin forte ou SSO se já existir)
- [ ] Não expor Grafana sem auth na internet
- [ ] Preferir path/host interno ou IP allowlist + TLS
- [ ] Datasources Prometheus + Loki apontando para serviços do compose

## Smoke tests

- [ ] Prometheus UI/API: targets ou queries básicas OK
- [ ] Grafana login OK; Explore Prometheus: `up` / métricas do collector
- [ ] Após API instrumentada: `charges_import_total` (após um import de teste)
- [ ] Loki: logs da API com `correlation_id` (Alloy/pipeline ok)
- [ ] Health interno da API responde na rede Docker; **não** na URL pública
- [ ] Fluxos críticos exercitados: import uCondo, GET charges, auth+RLS, automation-config/executions, Flyway/health/JVM/Postgres (métricas/logs conforme disponível)

## Rollback

- [ ] `docker compose down` (ou stop dos serviços de obs) sem derrubar a API de produção se redes forem desconectáveis com segurança
- [ ] Desconectar API da rede `condohome-obs` se OTLP causar impacto
- [ ] Reverter env OTLP da API (feature flag / unset endpoint) se necessário
- [ ] Traefik: manter bloqueio do actuator mesmo com obs desligado
- [ ] Documentar horário (America/Sao_Paulo) e o que foi revertido

## Explicit

- Sem OK do Rodrigo → **não** executar este checklist em produção

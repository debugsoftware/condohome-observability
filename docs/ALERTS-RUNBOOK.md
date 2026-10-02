# Alertas mínimos — CondoHome piloto (CH-45 / H4.2d)

> ⚠️ **Staging-vps-piloto only.** Sem produção, sem H4.3/Tempo, sem Alertmanager SaaS.
> Fonte de verdade: Prometheus `prometheus/rules/condohome-piloto.yml`.
> Grafana UI alerts **não** provisionados nesta fase (ver `grafana/alerts/README.md`).

**Horário de evidências / operação:** America/Belem (UTC-3).

## Como inspecionar

```bash
# Dentro do host VPS (rede Docker)
docker exec condohome-prometheus wget -qO- http://localhost:9090/api/v1/rules | python3 -m json.tool
docker exec condohome-prometheus wget -qO- http://localhost:9090/api/v1/alerts | python3 -m json.tool

# Reload após editar rules (lifecycle já habilitado)
docker exec condohome-prometheus wget -qO- --post-data='' http://localhost:9090/-/reload
# Se o mount de rules for novo: recreate
cd ~/condohome-observability/compose && docker compose up -d prometheus
```

## Alertas

### 1. OtelCollectorDown (critical)

| | |
|---|---|
| **Expr** | `up{job="otel-collector"} == 0` for 1m |
| **Significado** | Scrape do exporter Prometheus do collector (porta 8889) falhou. Sem collector, a API não entrega OTLP → métricas somem. |
| **Checar** | `docker ps -a --filter name=condohome-otel-collector`; `docker logs --tail=80 condohome-otel-collector`; `docker exec condohome-prometheus wget -qO- http://otel-collector:8889/metrics \| head` |
| **Mitigar** | `docker start condohome-otel-collector` ou `cd compose && docker compose up -d otel-collector`. Confirmar `up{job="otel-collector"}==1` e séries `condohome_jvm_*` voltando. |
| **Escalar** | Se collector crash-loop / OOM: ver RUNBOOK § "Collector OOM". Se persistir >15m no piloto → OPS + Arquiteto. |

### 2. ApiOtlpMetricsAbsent (warning)

| | |
|---|---|
| **Expr** | `absent(condohome_jvm_memory_used_bytes{service_name="condohome-api"})` for 2m |
| **Significado** | Telemetria JVM da API via OTLP sumiu. API pode estar down, fora da rede `condohome-obs`, OTLP desligado, ou collector sem scrape. |
| **NÃO fazer** | Tratar `up{job="condohome-api"}==0` sozinho como "API down" — o scrape Actuator fica 401/down de propósito. |
| **Checar** | `docker ps --filter name=condohome-api-staging`; `docker network inspect condohome-obs` (API listada?); logs API por OTLP; `up{job="otel-collector"}`; query JVM no Prometheus. |
| **Mitigar** | Subir API se caiu; `docker network connect condohome-obs <api>` se desconectada; confirmar env OTLP (`management.otlp.metrics.export.url` → `http://otel-collector:4318/v1/metrics`); restart collector se necessário. |
| **Escalar** | Se API healthy mas métricas ausentes >15m → Arquiteto (instrumentação) + OPS. |

### 3. ApiHttpErrorRateHigh (warning)

| | |
|---|---|
| **Expr** | Fração 5xx > 25% **e** taxa absoluta 5xx > 0.05/s por ≥5m (janela 5m). Ignora 4xx (401 de smoke OK). |
| **Significado** | API respondendo muitos 5xx de forma sustentada no piloto. |
| **Checar** | Grafana dashboard `CondoHome — API RED + Import`; Loki `{container="condohome-api-staging"}`; PromQL de `status=~"5.."` por `uri`. |
| **Mitigar** | Identificar endpoint dominante; corrigir causa (DB, enum, fixture, bug); confirmar taxa 5xx caindo. |
| **Escalar** | Se 5xx generalizado / API degradada → Arquiteto + Rodrigo (gate). |

## Teste controlado (validação)

Para validar que as regras disparam (somente staging):

```bash
# 1) Baseline
docker exec condohome-prometheus wget -qO- http://localhost:9090/api/v1/alerts

# 2) Parar collector (~2–3 min até OtelCollectorDown FIRING)
docker stop condohome-otel-collector

# 3) Aguardar ≥1m + evaluation; capturar FIRING
docker exec condohome-prometheus wget -qO- http://localhost:9090/api/v1/alerts

# 4) Restaurar IMEDIATAMENTE
docker start condohome-otel-collector
# Confirmar up=1, JVM series presentes, alert RESOLVED/ausente
```

**Não deixar o collector parado.** Não tocar containers de produção.

## Limites do lote EP4-v2 / H4.2

- Sem produção (`production_touched: false`)
- Sem H4.3 / Tempo
- Sem novo gasto / Alertmanager pago
- Credenciais Grafana só no `.env` do VPS (nunca em evidência `/workspace`)

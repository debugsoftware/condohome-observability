# CondoHome Observability

Padrões, stack, dashboards, alerts e docs de observabilidade do **CondoHome** (SaaS multi-tenant).

> ⚠️ **Sem merge/deploy em produção sem ok explícito do Rodrigo.**  
> Este repositório contém configuração de piloto — não constitui validação de capacidade do VPS.

## Stack (fase 1)

| Serviço | Função | Imagem (pin) |
|---------|--------|--------------|
| **otel-collector** | Recebe OTLP (métricas + logs); exporta métricas p/ Prometheus scrape; logs → Loki | `otel/opentelemetry-collector-contrib:0.160.0` |
| **prometheus** | Time-series; scrape collector + Actuator interno da API | `prom/prometheus:v3.13.2` |
| **loki** | Logs (filesystem, retenção 14d) | `grafana/loki:3.7.7` |
| **alloy** | Coleta logs de containers → Loki (+ redação PII básica) | `grafana/alloy:v1.19.2` |
| **grafana** | UI / dashboards | `grafana/grafana:13.2.1` |

**Não incluso:** Tempo / traces (ver [Fase 2](#fase-2--tempotraces)).  
**Não requerido no v1:** Cloudflare Logpush.

Rede Docker dedicada: **`condohome-obs`**.

## Quick Start

```bash
cd compose/
cp .env.example .env   # edite GRAFANA_ADMIN_PASSWORD
docker compose up -d
docker compose ps
```

Grafana: `http://<host>:3000` (user/senha do `.env`).

Dashboard provisionado: **CondoHome — API RED + Import**.

### Anexar a API à rede de obs

No compose da aplicação (ou `docker network connect`):

```bash
docker network connect condohome-obs <container-da-api>
```

Ajuste o target em `prometheus/prometheus.yml` (`condohome-api:8080`) para o hostname/porta reais.

## Portas e exposição

| Serviço | Porta | Exposição piloto |
|---------|-------|------------------|
| Grafana | **3000** (host) | UI no host — **proteger com Traefik + auth** em deploy real |
| OTel Collector OTLP gRPC | 4317 | Só rede `condohome-obs` |
| OTel Collector OTLP HTTP | 4318 | Só rede interna |
| OTel Prometheus exporter | 8889 | Só rede interna (scrape Prometheus) |
| Prometheus | 9090 | Só rede interna (opcional `127.0.0.1` p/ debug) |
| Loki | 3100 | Só rede interna |
| Alloy UI | 12345 | Só rede interna |

**Actuator da API (`/api/v1/actuator/*`) NÃO deve ser publicado na internet.**  
Prometheus faz scrape apenas pelo hostname Docker (ex.: `condohome-api:8080`) na rede interna. Traefik deve bloquear routers públicos para Actuator.

## Como a API deve enviar métricas (Micrometer → OTLP)

Preferência CondoHome: **Micrometer nativo com registry OTLP** — **não** Java agent.

Exemplo (Spring Boot / properties — ajuste ao projeto):

```properties
# application-pilot.properties (ilustrativo)
management.endpoints.web.exposure.include=health,prometheus,info
management.endpoint.health.show-details=when_authorized
management.otlp.metrics.export.url=http://otel-collector:4318/v1/metrics
management.otlp.metrics.export.step=30s
```

Dependência típica: `micrometer-registry-otlp` (+ Actuator).

### Correlação

- Aceitar/propagar **W3C `traceparent`** e/ou **`X-Correlation-Id`**
- Incluir `trace_id` / `correlation_id` nos logs estruturados
- Logs OK: `organization_id`, `condominium_id`, `client_id` público, `import_id`, `charge_id` (UUID), `error_code`, `route`, `method`, `status`, `duration_ms`
- **Nunca logar:** `client_secret`, JWT/Bearer completos, CPF/CNPJ claro, nomes/emails de moradores em alto volume, bodies de import xlsx, senhas de DB, tokens Resend/WhatsApp

### Cardinalidade

Use **apenas labels de baixa cardinalidade** em métricas. Evite `organization_id` em todo counter se houver muitos tenants — prefira logs / agregações.

## Segurança Traefik / Actuator

1. Health da API: `/api/v1/actuator/health` — **somente rede Docker / probe interno**
2. Prometheus path: `/api/v1/actuator/prometheus` — scrape interno apenas
3. No Traefik: sem router externo para `/api/v1/actuator/**` (ou middleware deny)
4. Grafana: Basic Auth Traefik / OAuth / VPN; troque a senha default do `.env`
5. Não exponha 9090/3100/4317/4318 no firewall do VPS

## Recursos (piloto 2–4 GB)

Limites no compose (~1.5 GB RAM somados, orientativos):

- otel-collector 256m, prometheus 512m, loki 384m, alloy 192m, grafana 256m

**Retenção Loki: 14 dias** (filesystem). Impacto em disco estimado no piloto: ~1–5 GB conforme volume de logs — **monitorar disco do VPS**. Prometheus TSDB: 15d / cap 2GB.

Capacidade real do VPS **não foi validada** por estes artefatos.

## Documentação

- **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** — Arquitetura e fluxo de dados
- **[docs/STANDARDS.md](docs/STANDARDS.md)** — Padrões de instrumentação
- **[docs/RUNBOOK.md](docs/RUNBOOK.md)** — Operação e troubleshooting
- **[docs/CHECKLIST-FLUXOS.md](docs/CHECKLIST-FLUXOS.md)** — 5 fluxos críticos (import uCondo, GET charges, Auth/RLS, automations n8n, infra)

### Handoffs para implementação

**Antes de iniciar:** ler **[docs/handoffs/01-API-MICROMETER.md](docs/handoffs/01-API-MICROMETER.md)** — instrumentação da API.

1. **[docs/handoffs/01-API-MICROMETER.md](docs/handoffs/01-API-MICROMETER.md)** — Arquiteto de Soluções: Micrometer → OTLP
2. **[docs/handoffs/02-N8N-CORRELATION.md](docs/handoffs/02-N8N-CORRELATION.md)** — N8N Developer: propagação de correlation
3. **[docs/handoffs/03-OPS-PILOTO.md](docs/handoffs/03-OPS-PILOTO.md)** — OPS: deploy VPS Hostinger (só com OK do Rodrigo)

## Estrutura

```
condohome-observability/
├── README.md
├── docs/
│   ├── ARCHITECTURE.md          # Arquitetura e fluxo de dados
│   ├── STANDARDS.md             # Padrões de instrumentação
│   ├── RUNBOOK.md               # Operação e troubleshooting
│   ├── CHECKLIST-FLUXOS.md      # 5 fluxos críticos
│   └── handoffs/                # Docs para implementadores
│       ├── 01-API-MICROMETER.md
│       ├── 02-N8N-CORRELATION.md
│       └── 03-OPS-PILOTO.md
├── grafana/
│   ├── dashboards/              # JSON dashboards
│   ├── alerts/                  # (placeholder fase 2)
│   └── provisioning/
│       ├── datasources/
│       └── dashboards/
├── loki/
│   ├── local-config.yaml
│   └── queries/                 # Queries úteis LogQL
├── prometheus/
│   ├── prometheus.yml
│   └── rules/                   # Alertas (fase 2)
├── otel-collector/
│   └── config.yaml
├── alloy/
│   └── config.alloy
├── compose/
│   ├── docker-compose.yml
│   └── .env.example
└── scripts/
    └── README.md                # Scripts auxiliares
```

## Fase 2 — Tempo/traces

Na fase 2:

- Adicionar Grafana Tempo (ou backend de traces compatível)
- Pipeline `traces` no OTel Collector
- Ligar Micrometer Tracing / Brave/OTel na API
- Datasource Tempo no Grafana + links Log↔Trace

**Não incluir Tempo no `docker-compose.yml` até a fase 2 estar aprovada.**

## Governança

- Artefatos **piloto**, production-ready na estrutura, lean no footprint
- Qualquer promoção a PRD: **ok do Rodrigo** obrigatório
- Stack aprovada H4.1: métricas + correlation/MDC; sem span export nesta fase
- API: soft-fail se collector down; safe logs only; Actuator nunca público

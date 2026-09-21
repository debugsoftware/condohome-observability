# Runbook Operacional — CondoHome Observability

Guia de operação, troubleshooting e manutenção do stack de observabilidade.

> ⚠️ **Sem mudanças em produção sem OK do Rodrigo.**

## Quick Reference

| Serviço | Container | Port | Health Check |
|---------|-----------|------|--------------|
| Grafana | `condohome-grafana` | 3000 | `http://localhost:3000/api/health` |
| Prometheus | `condohome-prometheus` | 9090 (interno) | `http://prometheus:9090/-/healthy` |
| Loki | `condohome-loki` | 3100 (interno) | `http://loki:3100/ready` |
| OTel Collector | `condohome-otel-collector` | 4318 (interno) | `http://otel-collector:13133` |
| Alloy | `condohome-alloy` | 12345 (interno) | `http://alloy:12345/-/ready` |

## Operações Básicas

### Subir o stack

```bash
cd /caminho/para/condohome-observability/compose
cp .env.example .env
# Edite .env: GRAFANA_ADMIN_PASSWORD (senha forte!)
docker compose up -d
docker compose ps
```

**Verificar:**
```bash
docker compose logs -f grafana      # verificar login OK
docker compose logs otel-collector  # verificar receivers listening
docker compose logs prometheus      # verificar scrape targets
```

**Primeira vez:**
- Grafana: `http://<host>:3000` (user/senha do `.env`)
- Dashboard provisionado: `CondoHome — API RED + Import`

### Parar o stack

```bash
cd compose/
docker compose down
```

**Dados persistem** nos volumes Docker:
- `prometheus-data` (TSDB)
- `loki-data` (logs)
- `grafana-data` (dashboards customizados)
- `alloy-data` (state)

### Restart de um serviço específico

```bash
docker compose restart grafana
docker compose restart otel-collector
docker compose restart prometheus
```

### Ver logs

```bash
docker compose logs -f <service-name>
docker compose logs --tail=100 otel-collector
```

### Atualizar configuração

**Prometheus:**
```bash
# Editar prometheus/prometheus.yml
docker compose restart prometheus
# ou reload sem restart (se --web.enable-lifecycle):
curl -X POST http://localhost:9090/-/reload
```

**Loki:**
```bash
# Editar loki/local-config.yaml
docker compose restart loki
```

**OTel Collector:**
```bash
# Editar otel-collector/config.yaml
docker compose restart otel-collector
```

**Alloy:**
```bash
# Editar alloy/config.alloy
docker compose restart alloy
```

**Grafana datasources/dashboards:**
- Provisioning é automático (60s interval)
- ou: restart `docker compose restart grafana`

## Conectar a API à rede de observability

A API precisa estar na rede Docker `condohome-obs` para enviar OTLP ao Collector.

### Método 1: Via `docker network connect`

```bash
docker network connect condohome-obs <api-container-name>
```

**Verificar:**
```bash
docker inspect <api-container-name> | grep condohome-obs
```

### Método 2: No compose da API

```yaml
# docker-compose.yml da API
services:
  condohome-api:
    # ...
    networks:
      - default
      - condohome-obs

networks:
  condohome-obs:
    external: true
    name: condohome-obs
```

### Verificar conectividade

De dentro do container da API:
```bash
docker exec -it <api-container> curl http://otel-collector:4318
# Deve retornar 405 Method Not Allowed (endpoint existe, só aceita POST)
```

## Troubleshooting

### 1. API não envia métricas ao Collector

**Sintomas:**
- Dashboard Grafana sem dados de `charges_import_*`
- Prometheus query `up{job="otel-collector"}` = 1 mas sem séries da API

**Diagnóstico:**
1. Logs da API:
   ```bash
   docker compose -f <api-compose> logs | grep -i otlp
   ```
   Procurar erros de conexão ou timeout.

2. Verificar hostname resolve:
   ```bash
   docker exec -it <api-container> ping otel-collector
   docker exec -it <api-container> curl http://otel-collector:4318
   ```

3. Logs do Collector:
   ```bash
   docker compose logs otel-collector | grep -i error
   ```

4. Verificar rede:
   ```bash
   docker network inspect condohome-obs | grep <api-container>
   ```

**Soluções:**
- Se API não está na rede: `docker network connect condohome-obs <api-container>`
- Se Collector está down: `docker compose restart otel-collector`
- Se config OTLP da API errada: corrigir endpoint para `http://otel-collector:4318/v1/metrics`
- Se Collector rejeitando (ex.: corpo inválido): verificar versão registry OTLP da API compatível

### 2. Prometheus scrape failing

**Sintomas:**
- Prometheus UI → Status → Targets: job `otel-collector` ou `condohome-api` DOWN

**Diagnóstico:**
```bash
# Dentro do container Prometheus
docker exec -it condohome-prometheus wget -O- http://otel-collector:8889/metrics
docker exec -it condohome-prometheus wget -O- http://condohome-api:8080/api/v1/actuator/prometheus
```

**Soluções:**
- Se timeout: verificar que target está na mesma rede (`condohome-obs`)
- Se 404: corrigir `metrics_path` no `prometheus/prometheus.yml`
- Se Actuator da API desabilitado: verificar `management.endpoints.web.exposure.include=prometheus` na API

### 3. Logs não aparecem no Loki

**Sintomas:**
- Grafana Explore → Loki: sem logs de `{job="docker"}`
- Dashboard logs panel vazio

**Diagnóstico:**
1. Alloy logs:
   ```bash
   docker compose logs alloy | grep -i error
   ```

2. Verificar Alloy descobrindo containers:
   ```bash
   docker compose logs alloy | grep discovery
   ```

3. Loki logs:
   ```bash
   docker compose logs loki | grep -i push
   ```

4. Testar push manual:
   ```bash
   curl -X POST http://loki:3100/loki/api/v1/push \
     -H "Content-Type: application/json" \
     -d '{"streams": [{"stream": {"job": "test"}, "values": [["'$(date +%s)000000000'", "test log"]]}]}'
   ```

**Soluções:**
- Se Alloy não descobre containers: verificar mount `/var/run/docker.sock`
- Se Loki rejeitando push: verificar formato / timestamp (não pode ser muito antigo, vide `reject_old_samples_max_age`)
- Se PII redaction muito agressiva: ajustar regex em `alloy/config.alloy`

### 4. Grafana não carrega dashboards

**Sintomas:**
- Dashboard folder `CondoHome` vazio
- Datasources não aparecem

**Diagnóstico:**
```bash
docker compose logs grafana | grep -i provisioning
docker compose logs grafana | grep -i error
```

**Soluções:**
- Se datasources não provisionados: verificar `grafana/provisioning/datasources/datasources.yml` montado corretamente
- Se dashboard JSON inválido: validar JSON em `grafana/dashboards/*.json`
- Restart: `docker compose restart grafana`

### 5. Disco VPS cheio

**Sintomas:**
- Loki/Prometheus parando de gravar
- Docker compose falha ao subir

**Diagnóstico:**
```bash
df -h
du -sh /var/lib/docker/volumes/
docker system df -v
```

**Soluções:**
1. **Limpeza temporária:**
   ```bash
   docker system prune -a --volumes  # ⚠️ remove volumes não usados
   ```

2. **Reduzir retenção Loki:**
   Editar `loki/local-config.yaml`:
   ```yaml
   limits_config:
     retention_period: 168h  # 7 dias (era 14)
   ```
   Restart: `docker compose restart loki`

3. **Reduzir retenção Prometheus:**
   Editar `compose/docker-compose.yml`:
   ```yaml
   prometheus:
     command:
       - "--storage.tsdb.retention.time=7d"  # era 15d
   ```
   Restart: `docker compose restart prometheus`

4. **Aumentar disco VPS** (requer planejamento com infra).

### 6. Collector OOM / crash loop

**Sintomas:**
- Container `condohome-otel-collector` reiniciando constantemente
- `docker compose ps` → `Restarting`

**Diagnóstico:**
```bash
docker compose logs otel-collector --tail=100
# Procurar: memory_limiter processor dropping data, OOM killed
```

**Soluções:**
1. **Aumentar mem_limit** (se VPS tiver RAM):
   Editar `compose/docker-compose.yml`:
   ```yaml
   otel-collector:
     mem_limit: 512m  # era 256m
   ```

2. **Reduzir batch size:**
   Editar `otel-collector/config.yaml`:
   ```yaml
   processors:
     batch:
       send_batch_size: 256  # era 512
       timeout: 10s  # era 5s
   ```

3. **Desabilitar exporter debug** (se ativado).

### 7. Actuator exposto publicamente (CRITICAL)

**Sintomas:**
- `curl https://<dominio-publico>/api/v1/actuator/health` retorna 200 OK (não deveria!)

**Diagnóstico:**
```bash
curl -I https://<dominio>/api/v1/actuator/health
curl -I https://<dominio>/api/v1/actuator/prometheus
```

**Solução IMEDIATA (P1):**
1. Bloquear no Traefik:
   ```yaml
   # Traefik middleware
   http:
     middlewares:
       deny-actuator:
         ipWhiteList:
           sourceRange:
             - "127.0.0.1/32"
             - "172.16.0.0/12"  # Docker internal
   ```

2. Ou remover router público de `/actuator/**` completamente.

3. Verificar:
   ```bash
   curl -I https://<dominio>/api/v1/actuator/health
   # Deve retornar 403/404/timeout — **não** 200
   ```

### 8. Grafana login falha

**Sintomas:**
- Admin user/senha do `.env` não funciona

**Soluções:**
1. Reset senha admin:
   ```bash
   docker exec -it condohome-grafana grafana-cli admin reset-admin-password <nova-senha>
   ```

2. Ou recriar volume (perde dashboards customizados):
   ```bash
   docker compose down
   docker volume rm condohome-observability_grafana-data
   docker compose up -d
   ```

## Manutenção

### Atualizar imagens (com cautela)

1. **Testar em staging primeiro:**
   ```bash
   cd compose/
   # Editar docker-compose.yml: atualizar tag de imagem
   docker compose pull <service>
   docker compose up -d <service>
   ```

2. **Verificar compatibilidade:**
   - OTel Collector: [changelog](https://github.com/open-telemetry/opentelemetry-collector-contrib/releases)
   - Prometheus: [release notes](https://github.com/prometheus/prometheus/releases)
   - Loki: [changelog](https://github.com/grafana/loki/releases)
   - Grafana: [what's new](https://grafana.com/docs/grafana/latest/whatsnew/)
   - Alloy: [release notes](https://github.com/grafana/alloy/releases)

3. **Backup antes de atualizar produção:**
   ```bash
   docker compose down
   tar -czf backup-obs-$(date +%Y%m%d).tar.gz \
     /var/lib/docker/volumes/condohome-observability_prometheus-data \
     /var/lib/docker/volumes/condohome-observability_loki-data \
     /var/lib/docker/volumes/condohome-observability_grafana-data
   docker compose up -d
   ```

### Backup de dados

**Manual:**
```bash
# Parar serviços
docker compose down

# Backup volumes
tar -czf obs-backup-$(date +%Y%m%d).tar.gz \
  /var/lib/docker/volumes/condohome-observability_*

# Subir novamente
docker compose up -d
```

**Restaurar:**
```bash
docker compose down
# Extrair backup
tar -xzf obs-backup-YYYYMMDD.tar.gz -C /
docker compose up -d
```

**Automação (cron semanal):**
```bash
# /etc/cron.weekly/backup-obs
#!/bin/bash
cd /caminho/para/condohome-observability/compose
/usr/local/bin/docker-compose down
tar -czf /backup/obs-$(date +%Y%m%d).tar.gz /var/lib/docker/volumes/condohome-observability_*
/usr/local/bin/docker-compose up -d
# Limpar backups > 30 dias
find /backup/obs-*.tar.gz -mtime +30 -delete
```

### Monitorar recursos VPS

```bash
# CPU / RAM
docker stats

# Disco
df -h /
du -sh /var/lib/docker/volumes/condohome-observability_*

# Network
docker exec condohome-prometheus wget -qO- localhost:9090/metrics | grep prometheus_tsdb_storage_blocks_bytes
```

**Alertar se:**
- RAM > 80% usage sustentado
- Disco > 80% (Loki/Prometheus crescendo)
- CPU > 90% sustentado (pode impactar API)

## Queries Úteis

### Prometheus (UI ou API)

**Métricas da API:**
```promql
# Request rate
rate(http_server_requests_seconds_count{job="condohome-api"}[5m])

# Error rate
rate(http_server_requests_seconds_count{job="condohome-api", status=~"5.."}[5m])

# Latência p95
histogram_quantile(0.95, 
  rate(http_server_requests_seconds_bucket{job="condohome-api"}[5m])
)

# Import success rate
rate(charges_import_total{result="success"}[5m])
/ 
rate(charges_import_total[5m])
```

**Infra:**
```promql
# Scrape targets up
up{job="otel-collector"}
up{job="condohome-api"}

# OTel Collector memory
process_resident_memory_bytes{job="otel-collector-internal"}

# Prometheus TSDB size
prometheus_tsdb_storage_blocks_bytes
```

### Loki (Grafana Explore)

**Logs da API:**
```logql
{job="docker", container=~"condohome-api.*"}
```

**Erros (5xx):**
```logql
{job="docker", container=~"condohome-api.*"} |= "status" |= "5"
```

**Correlation ID lookup:**
```logql
{job="docker"} | json | correlation_id="550e8400-e29b-41d4-a716-446655440000"
```

**Import failures:**
```logql
{job="docker", container=~"condohome-api.*"} 
  | json 
  | route="/api/v1/charges/imports/ucondo" 
  | outcome="FAILURE"
```

**Auth failures:**
```logql
{job="docker", container=~"condohome-api.*"} 
  | json 
  | status=~"40[13]"
```

**Rate de erros (últimos 5m):**
```logql
sum(rate({job="docker", container=~"condohome-api.*"} 
  | json 
  | status=~"5.." [5m]))
```

Ver também: [loki/queries/README.md](../loki/queries/README.md) com queries prontas.

## Alertas (fase 2)

Quando Prometheus alerting rules forem ativados (`prometheus/rules/condohome-alerts.yml`):

**Testar alert rule:**
```bash
# Prometheus UI → Alerts → verificar pending/firing
# ou API:
curl http://localhost:9090/api/v1/alerts
```

**Silence alert (temporário):**
- Grafana → Alerting → Silences
- Ou Prometheus Alertmanager (se configurado)

## Rollback

Se stack de obs causar problemas em produção:

1. **Parar stack (API continua funcionando):**
   ```bash
   docker compose down
   ```

2. **Desconectar API da rede (se necessário):**
   ```bash
   docker network disconnect condohome-obs <api-container>
   ```

3. **Desabilitar OTLP na API (se impactando performance):**
   ```yaml
   # application.yml da API
   management:
     otlp:
       metrics:
         export:
           enabled: false
   ```
   Restart API.

4. **Manter Traefik bloqueando Actuator** (mesmo com obs down).

5. **Documentar:**
   - Horário (America/Sao_Paulo)
   - O que foi revertido
   - Motivo (ex.: disco cheio, OOM, performance)
   - Link para incident postmortem

## Contatos / Escalação

| Responsável | Scope | Ação |
|-------------|-------|------|
| **Rodrigo** | Gate final merge/deploy | OK obrigatório antes de mudanças PRD |
| **Arquiteto de Soluções** | Instrumentação API | Issues de métricas/logs da API |
| **N8N Developer** | Correlation n8n | Issues de correlation em automations |
| **OPS** | Infra VPS / Traefik | Disco cheio, firewall, Traefik |

**Incidente P1 (CRITICAL):**
- Actuator exposto publicamente → OPS + Arquiteto
- PII vazada em logs → Arquiteto + Rodrigo
- Cross-tenant leak detected → Rodrigo + Arquiteto (immediate)

## Governança

- **Runbook atualizado** a cada mudança de stack/config
- Mudanças operacionais em produção: **OK do Rodrigo** obrigatório
- Postmortems de incidents: adicionar seção "Lessons Learned" + update runbook

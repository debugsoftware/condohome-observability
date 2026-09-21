# Scripts — CondoHome Observability

Scripts auxiliares para operação e manutenção do stack de observabilidade.

> ⚠️ **Scripts devem ser testados em staging antes de usar em produção.**

## Placeholder

Este diretório está reservado para scripts operacionais futuros.

Exemplos de scripts úteis (a adicionar conforme necessidade):

### 1. Backup automático

```bash
#!/bin/bash
# scripts/backup-obs.sh
# Backup volumes Prometheus/Loki/Grafana

set -e

BACKUP_DIR="${BACKUP_DIR:-/backup}"
DATE=$(date +%Y%m%d-%H%M%S)
COMPOSE_DIR="${COMPOSE_DIR:-/opt/condohome-observability/compose}"

cd "$COMPOSE_DIR"

echo "Stopping services..."
docker compose down

echo "Creating backup..."
tar -czf "$BACKUP_DIR/obs-$DATE.tar.gz" \
  /var/lib/docker/volumes/condohome-observability_prometheus-data \
  /var/lib/docker/volumes/condohome-observability_loki-data \
  /var/lib/docker/volumes/condohome-observability_grafana-data

echo "Starting services..."
docker compose up -d

echo "Cleaning old backups (>30d)..."
find "$BACKUP_DIR"/obs-*.tar.gz -mtime +30 -delete

echo "Backup complete: $BACKUP_DIR/obs-$DATE.tar.gz"
```

**Cron (semanal):**
```bash
# /etc/cron.weekly/backup-obs
0 2 * * 0 /opt/condohome-observability/scripts/backup-obs.sh >> /var/log/obs-backup.log 2>&1
```

### 2. Health check de todos os serviços

```bash
#!/bin/bash
# scripts/health-check.sh
# Verifica health de todos os serviços obs

set -e

SERVICES=(
  "grafana:3000/api/health"
  "prometheus:9090/-/healthy"
  "loki:3100/ready"
  "otel-collector:13133"
  "alloy:12345/-/ready"
)

for svc in "${SERVICES[@]}"; do
  name="${svc%%:*}"
  url="http://${svc#*:}"
  
  echo -n "Checking $name... "
  if docker exec condohome-grafana wget -qO- "$url" > /dev/null 2>&1; then
    echo "✓ OK"
  else
    echo "✗ FAIL"
    exit 1
  fi
done

echo "All services healthy"
```

### 3. Limpar dados antigos (emergência disco cheio)

```bash
#!/bin/bash
# scripts/emergency-cleanup.sh
# Limpa dados antigos de Prometheus/Loki em caso de disco cheio

set -e

echo "⚠️  EMERGENCY CLEANUP — will reduce retention"
read -p "Continue? (yes/no): " confirm
[[ "$confirm" != "yes" ]] && exit 0

COMPOSE_DIR="${COMPOSE_DIR:-/opt/condohome-observability/compose}"
cd "$COMPOSE_DIR"

# Reduzir retenção Loki para 7d
echo "Reducing Loki retention to 7 days..."
sed -i 's/retention_period: 336h/retention_period: 168h/' ../loki/local-config.yaml
docker compose restart loki

# Reduzir retenção Prometheus para 7d
echo "Reducing Prometheus retention to 7 days..."
sed -i 's/retention.time=15d/retention.time=7d/' docker-compose.yml
docker compose restart prometheus

# Limpar Docker system
echo "Cleaning Docker system..."
docker system prune -f

echo "Cleanup complete. Monitor disk usage: df -h"
```

### 4. Verificar conectividade API → Collector

```bash
#!/bin/bash
# scripts/check-api-connectivity.sh
# Verifica se API consegue alcançar OTel Collector

set -e

API_CONTAINER="${1:-condohome-api}"

echo "Checking connectivity from $API_CONTAINER to otel-collector..."

# Ping
if docker exec "$API_CONTAINER" ping -c 1 otel-collector > /dev/null 2>&1; then
  echo "✓ Ping OK"
else
  echo "✗ Ping FAIL — API not in condohome-obs network?"
  exit 1
fi

# HTTP check (4318 deve retornar 405 para GET)
if docker exec "$API_CONTAINER" sh -c 'command -v curl > /dev/null' 2>/dev/null; then
  status=$(docker exec "$API_CONTAINER" curl -s -o /dev/null -w '%{http_code}' http://otel-collector:4318)
  if [[ "$status" == "405" ]]; then
    echo "✓ HTTP OK (405 Method Not Allowed — expected for GET)"
  else
    echo "✗ HTTP FAIL (got $status)"
    exit 1
  fi
else
  echo "⚠ curl not available in API container, skipping HTTP check"
fi

echo "Connectivity OK"
```

### 5. Reload Prometheus config sem restart

```bash
#!/bin/bash
# scripts/reload-prometheus.sh
# Reload Prometheus config via API (se --web.enable-lifecycle ativo)

set -e

if docker exec condohome-prometheus wget -qO- --post-data='' http://localhost:9090/-/reload > /dev/null 2>&1; then
  echo "✓ Prometheus config reloaded"
else
  echo "✗ Reload failed — try docker compose restart prometheus"
  exit 1
fi
```

### 6. Export dashboard Grafana para JSON

```bash
#!/bin/bash
# scripts/export-dashboard.sh <dashboard-uid>
# Exporta dashboard Grafana para JSON (backup manual)

set -e

UID="${1:?Dashboard UID required}"
OUTPUT="${2:-dashboard-${UID}.json}"

GRAFANA_URL="http://localhost:3000"
GRAFANA_USER="${GRAFANA_ADMIN_USER:-admin}"
GRAFANA_PASS="${GRAFANA_ADMIN_PASSWORD:-changeme}"

curl -s -u "$GRAFANA_USER:$GRAFANA_PASS" \
  "$GRAFANA_URL/api/dashboards/uid/$UID" \
  | jq '.dashboard' > "$OUTPUT"

echo "Dashboard exported to $OUTPUT"
```

### 7. Verificar PII vazada em logs (audit)

```bash
#!/bin/bash
# scripts/audit-pii-leaks.sh
# Audita logs em busca de PII não redacted

set -e

echo "Scanning logs for PII leaks..."

# Bearer tokens não redacted
echo -n "Checking Bearer tokens... "
if docker exec condohome-loki logcli query \
  '{job="docker"} |~ "Bearer\\s+[A-Za-z0-9._-]{30,}" !~ "REDACTED"' \
  --since=1h --limit=1 --quiet 2>/dev/null | grep -q 'Bearer'; then
  echo "⚠️  FOUND — incident P1"
  exit 1
else
  echo "✓ OK"
fi

# CPF/CNPJ em claro
echo -n "Checking CPF/CNPJ... "
if docker exec condohome-loki logcli query \
  '{job="docker"} |~ "\\d{3}\\.\\d{3}\\.\\d{3}-\\d{2}|\\d{2}\\.\\d{3}\\.\\d{3}/\\d{4}-\\d{2}"' \
  --since=1h --limit=1 --quiet 2>/dev/null | grep -qE '\d{3}\.\d{3}\.\d{3}-\d{2}'; then
  echo "⚠️  FOUND — incident P1"
  exit 1
else
  echo "✓ OK"
fi

echo "No PII leaks detected in last 1h"
```

## Uso

1. **Tornar executável:**
   ```bash
   chmod +x scripts/*.sh
   ```

2. **Configurar variáveis de ambiente:**
   ```bash
   export COMPOSE_DIR=/opt/condohome-observability/compose
   export BACKUP_DIR=/backup
   export GRAFANA_ADMIN_USER=admin
   export GRAFANA_ADMIN_PASSWORD=...
   ```

3. **Executar:**
   ```bash
   ./scripts/health-check.sh
   ./scripts/backup-obs.sh
   ```

## Governança

- Scripts devem ser **idempotentes** (safe para executar múltiplas vezes)
- Scripts destrutivos (cleanup, reload) devem ter **confirmação interativa**
- Logs de execução devem ir para `/var/log/obs-*.log`
- **Testar em staging** antes de usar em produção
- Scripts de backup devem notificar se falharem (email/Slack)

## Adicionar novos scripts

1. Criar arquivo em `scripts/`
2. Adicionar shebang: `#!/bin/bash`
3. Adicionar `set -e` (fail on error)
4. Documentar uso em comentário no topo
5. Adicionar entrada neste README
6. Testar em staging
7. Commit + PR (com OK do Rodrigo se crítico)

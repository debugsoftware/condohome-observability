# Loki Queries — CondoHome

Queries LogQL prontas para troubleshooting e análise de logs.

Usar em **Grafana → Explore → Loki datasource** ou na CLI `logcli`.

## Queries Básicas

### Ver logs da API (últimos 5m)

```logql
{job="docker", container=~"condohome-api.*"}
```

### Ver logs de todos os containers obs

```logql
{job="docker", container=~"condohome-.*"}
```

## Queries por Correlation ID

### Lookup por correlation_id (end-to-end trace)

```logql
{job="docker"} | json | correlation_id="550e8400-e29b-41d4-a716-446655440000"
```

Substitua UUID pelo `correlation_id` real do request.

### Lookup por trace_id (W3C)

```logql
{job="docker"} | json | trace_id="4bf92f3577b34da6a3ce929d0e0e4736"
```

## Queries de Erro

### Todos os erros (status 5xx)

```logql
{job="docker", container=~"condohome-api.*"} 
  | json 
  | status=~"5.."
```

### Erros por rota

```logql
{job="docker", container=~"condohome-api.*"} 
  | json 
  | status=~"5.." 
  | line_format "{{.route}} - {{.status}} - {{.error_code}}"
```

### Rate de 5xx (últimos 5m)

```logql
sum(rate({job="docker", container=~"condohome-api.*"} 
  | json 
  | status=~"5.." [5m]))
```

### Top rotas com erro (últimos 1h)

```logql
topk(10, 
  sum by (route) (
    count_over_time({job="docker", container=~"condohome-api.*"} 
      | json 
      | status=~"5.." [1h])
  )
)
```

## Import uCondo

### Imports com falha

```logql
{job="docker", container=~"condohome-api.*"} 
  | json 
  | route=~".*/charges/imports/ucondo" 
  | outcome="FAILURE"
```

### Imports com parse errors

```logql
{job="docker", container=~"condohome-api.*"} 
  | json 
  | error_code=~"INVALID_DATE|MISSING_FIELD|PARSE_ERROR"
```

### Import success rate (últimos 30m)

```logql
# Success
sum(count_over_time({job="docker", container=~"condohome-api.*"} 
  | json 
  | route=~".*/charges/imports/ucondo" 
  | outcome="SUCCESS" [30m]))
/
# Total
sum(count_over_time({job="docker", container=~"condohome-api.*"} 
  | json 
  | route=~".*/charges/imports/ucondo" [30m]))
```

### Duração de imports (p95)

```logql
quantile_over_time(0.95, 
  {job="docker", container=~"condohome-api.*"} 
    | json 
    | route=~".*/charges/imports/ucondo" 
    | unwrap duration_ms [30m])
```

## Auth / RLS

### Tentativas de autenticação falhadas (401)

```logql
{job="docker", container=~"condohome-api.*"} 
  | json 
  | status="401"
```

### Rate de 401 (últimos 15m)

```logql
sum(rate({job="docker", container=~"condohome-api.*"} 
  | json 
  | status="401" [15m]))
```

### Acesso negado (403)

```logql
{job="docker", container=~"condohome-api.*"} 
  | json 
  | status="403"
```

### Cross-tenant leak attempt (CRITICAL)

```logql
{job="docker", container=~"condohome-api.*"} 
  | json 
  | error_code="TENANT_MISMATCH"
```

**Alert:** qualquer match desta query = **P1 incident** imediato.

### RLS denials por tipo

```logql
{job="docker", container=~"condohome-api.*"} 
  | json 
  | error_code=~".*RLS.*|.*TENANT.*|FORBIDDEN"
  | line_format "{{.organization_id}} - {{.error_code}}"
```

## Automation (n8n)

### Automation executions com falha

```logql
{job="docker", container=~"n8n.*"} 
  | json 
  | outcome="FAILURE"
```

ou (se logs vêm da API):

```logql
{job="docker", container=~"condohome-api.*"} 
  | json 
  | route=~".*/automation.*" 
  | outcome="FAILURE"
```

### Automation success rate (últimos 1h)

```logql
sum(count_over_time({job="docker"} 
  | json 
  | route=~".*/automation.*" 
  | outcome="SUCCESS" [1h]))
/
sum(count_over_time({job="docker"} 
  | json 
  | route=~".*/automation.*" [1h]))
```

## Charges (GET)

### Latência alta em GET charges (> 1s)

```logql
{job="docker", container=~"condohome-api.*"} 
  | json 
  | route=~".*/charges" 
  | method="GET" 
  | duration_ms > 1000
```

### Top organizations por request volume (1h)

```logql
topk(10, 
  sum by (organization_id) (
    count_over_time({job="docker", container=~"condohome-api.*"} 
      | json 
      | route=~".*/charges" [1h])
  )
)
```

### Rate de 4xx em GET charges (client errors)

```logql
sum(rate({job="docker", container=~"condohome-api.*"} 
  | json 
  | route=~".*/charges" 
  | method="GET" 
  | status=~"4.." [15m]))
```

## Infra

### Flyway migration logs

```logql
{job="docker", container=~"condohome-api.*"} |~ "(?i)flyway"
```

### Health check failures (se não dropado pelo Alloy)

```logql
{job="docker", container=~"condohome-api.*"} 
  | json 
  | route=~".*/actuator/health" 
  | status!="200"
```

### JVM OOM / heap issues

```logql
{job="docker", container=~"condohome-api.*"} 
  |~ "(?i)OutOfMemoryError|heap|GC"
```

### Container restart logs

```logql
{job="docker"} |~ "(?i)restarting|exit|killed"
```

## PII / Security

### Logs com possível PII vazado (checklist)

```logql
{job="docker", container=~"condohome-api.*"} 
  |~ "(?i)password|secret|bearer\\s+[a-z0-9]{20,}|\\d{3}\\.\\d{3}\\.\\d{3}-\\d{2}"
```

**Não deveria retornar nada** — Alloy deveria ter redacted. Se retornar: **incident P1**.

### Tokens não redacted

```logql
{job="docker", container=~"condohome-api.*"} 
  |~ "Bearer\\s+[A-Za-z0-9._-]{30,}"
  !~ "REDACTED"
```

## Dicas de Performance

### Filtrar por container antes de parse JSON

✅ Bom:
```logql
{job="docker", container=~"condohome-api.*"} | json | status="500"
```

❌ Lento:
```logql
{job="docker"} | json | status="500"  # parse todos os containers
```

### Usar line filter antes de JSON parse

✅ Bom:
```logql
{job="docker", container=~"condohome-api.*"} 
  |= "FAILURE"   # line filter (rápido)
  | json 
  | outcome="FAILURE"
```

### Limitar time range em queries pesadas

- Queries com `count_over_time`, `rate`, `quantile_over_time`: preferir 5m–1h
- Queries com `|= "string"` (line filter): OK em ranges maiores (6h–24h)
- Queries com aggregation (`sum`, `topk`): limitar a 1h–6h para performance

### Usar logcli para queries batch

```bash
logcli query '{job="docker", container=~"condohome-api.*"} | json | status="500"' \
  --since=6h \
  --limit=100 \
  --output=jsonl > errors.jsonl
```

## Alertas (fase 2)

Queries candidatas para Loki alerts (quando Grafana alerting ativado):

1. **Import failure spike:**
   ```logql
   sum(rate({job="docker", container=~"condohome-api.*"} 
     | json | route=~".*/imports/ucondo" | outcome="FAILURE" [15m])) > 0.1
   ```

2. **5xx rate alto:**
   ```logql
   sum(rate({job="docker", container=~"condohome-api.*"} 
     | json | status=~"5.." [5m])) > 1
   ```

3. **Cross-tenant leak (CRITICAL):**
   ```logql
   count_over_time({job="docker", container=~"condohome-api.*"} 
     | json | error_code="TENANT_MISMATCH" [5m]) > 0
   ```

4. **401 spike (possible attack):**
   ```logql
   sum(rate({job="docker", container=~"condohome-api.*"} 
     | json | status="401" [5m])) > 10
   ```

## Recursos

- [LogQL docs](https://grafana.com/docs/loki/latest/query/)
- [LogQL cheat sheet](https://megamorf.gitlab.io/cheat-sheets/loki/)
- [logcli usage](https://grafana.com/docs/loki/latest/tools/logcli/)

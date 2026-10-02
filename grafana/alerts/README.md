# Grafana alerts (piloto)

**CH-45 / H4.2d:** a fonte de verdade dos alertas do piloto é o **Prometheus**
(`prometheus/rules/condohome-piloto.yml`), carregado via `rule_files` + mount
no compose.

Grafana Alerting UI / provisioned JSON **não** é usado nesta fase:
- sem Alertmanager / contact points pagos
- Grafana 13 + provisioning de alertas sem AM fica frágil no piloto

Para ver alertas: Prometheus `/api/v1/alerts` (ou UI Status → Alerts via
exec no container). Runbook: [`docs/ALERTS-RUNBOOK.md`](../../docs/ALERTS-RUNBOOK.md).

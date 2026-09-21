# 02 — n8n Correlation (N8N Developer)

**Track:** N8N Developer (automation)  
**Gate:** mudanças de workflow/tech client; **sem merge/deploy de apps sem OK do Rodrigo**

## Objetivo

Garantir que o **tech client** n8n → `condohome-api` propaga correlação end-to-end e que execuções de workflow registram o id para suporte/debug.

## Propagação de headers

Em **todas** as chamadas HTTP do tech client para a API:

- [ ] Enviar `X-Correlation-Id` se já existir no contexto do workflow / trigger
- [ ] Enviar W3C `traceparent` se disponível (não inventar formato inválido)
- [ ] Se a API devolver `X-Correlation-Id` na response: guardar e reutilizar nas chamadas seguintes do mesmo fluxo
- [ ] Se o trigger não trouxer correlation: gerar UUID no início do workflow e usar até o fim

Headers a preservar entre nodes: `X-Correlation-Id`, `traceparent` (quando presente).

## Logs de correlação nas executions

- [ ] Logar (ou gravar em campo de execução) o `correlation_id` no início e em falhas
- [ ] Em erros de HTTP para a API: incluir status + `correlation_id` (sem body com PII/secrets)
- [ ] Não logar Authorization, API keys, payloads com dados pessoais

## Métricas / sucesso-falha (expectativa)

- Regras de negócio **permanecem na API**, não no n8n
- n8n: apenas sinalizar outcome da execução (success/error) e correlation para troubleshooting
- Se houver métricas/contadores no lado n8n no futuro: labels de baixa cardinalidade; **sem** `organization_id` em todo counter
- Import uCondo / charges: a fonte de verdade das métricas `charges_import_*` é a **API** (`01-API-MICROMETER.md`)

## Critérios de aceite

1. Workflow de teste (ex.: automation-config / executions ou import via tech client) envia `X-Correlation-Id`
2. Logs/execution n8n mostram o mesmo id que a API ecoa / registra
3. Cadeia: trigger → n8n → API → logs Loki (quando OPS/piloto no ar) filtrável pelo mesmo id
4. Falha HTTP deliberada: execution registra correlation + status, sem secrets
5. Nenhuma regra de negócio nova (billing, RLS, parse uCondo) movida para n8n

## Dependências

- API com filter de correlation (track Arquiteto) — idealmente após ou em paralelo com `01`
- Compose obs opcional para validar em Loki; validação mínima: headers + logs n8n + response API

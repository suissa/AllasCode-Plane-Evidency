# Contrato OpenTelemetry e Grafana

## Sinais e correlação

O contrato serializa OTLP sem adaptar o domínio ao backend. Logs seguem o LogRecord do OpenTelemetry: `Timestamp`, `ObservedTimestamp`, `TraceId`, `SpanId`, `TraceFlags`, `SeverityText`, `SeverityNumber`, `Body`, `Resource`, `InstrumentationScope`, `Attributes` e `EventName`.

### Resource comum

```yaml
service.namespace: allascode
service.name: allascode-runtime
service.version: <runtime.version>
service.instance.id: <stable-process-instance-id>
deployment.environment.name: <dev|test|staging|production>
host.id: <non-personal-node-id>
allascode.plane.name: evidency
```

### InstrumentationScope

```yaml
name: org.allascode.runtime.stage
version: 1.0.0
schema_url: https://schemas.allascode.org/evidency/1.0.0
```

## Logs

| Transição | EventName | Severidade OTel | Body estável |
|---|---|---:|---|
| `started` | `allascode.stage.started` | INFO / 9 | `Stage execution started` |
| `completed` + `ok` | `allascode.stage.completed` | INFO / 9 | `Stage execution completed` |
| `completed` + `healed` | `allascode.stage.completed` | WARN / 13 | `Stage execution completed after healing` |
| `failed` recuperável | `allascode.stage.failed` | ERROR / 17 | `Stage execution failed and entered healing` |
| falha de processo | `allascode.runtime.crashed` | FATAL / 21 | `Runtime process crashed` |

A mensagem detalhada fica em atributo sanitizado. Labels Loki indexáveis ficam limitadas a `service_name`, `deployment_environment_name`, `stage_name`, `stage_kind`, `result` e `severity_text`. `trace_id`, `span_id`, `event_id`, Action dinâmica, mensagem e usuário permanecem em metadados estruturados pesquisáveis, não em labels.

## Métricas

| Nome | Tipo OTel | Unidade | Momento |
|---|---|---|---|
| `allascode.stage.executions` | Counter monotônico | `{execution}` | término |
| `allascode.stage.duration` | Histogram | `s` | término |
| `allascode.stage.active` | UpDownCounter | `{execution}` | início/término |
| `allascode.stage.errors` | Counter monotônico | `{error}` | falha |
| `allascode.stage.retries` | Counter monotônico | `{retry}` | nova tentativa |
| `allascode.healing.transitions` | Counter monotônico | `{transition}` | mudança de healing |
| `allascode.stage.input.size` | Histogram | `By` | início |
| `allascode.stage.output.size` | Histogram | `By` | término |
| `allascode.observability.store.size` | Gauge | `By` | coleta a cada 60 s |
| `allascode.observability.snapshot.age` | Gauge | `s` | coleta a cada 60 s |
| `allascode.observability.dropped` | Counter monotônico | `{record}` | descarte governado |

Temporality padrão: delta para exportação OTLP local; o Collector/backend pode converter para cumulative/Prometheus. Todo histograma terminal MAY carregar exemplar com `trace_id` e `span_id`; IDs nunca viram atributos da série.

Buckets de duração em segundos: `0.001, 0.0025, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10, 30, 60`.

Buckets de tamanho em bytes: `64, 256, 1024, 4096, 16384, 65536, 262144, 1048576`.

### Dimensões métricas permitidas

`stage.name`, `stage.kind`, `agent.name`, `intent.name`, `action.name`, `result`, `error.type`, `healing.outcome` e `deployment.environment.name`. Os valores precisam pertencer a catálogos carregados no deploy. O limite padrão é 2.000 séries ativas por instrumento e 10.000 séries no processo; acima disso a dimensão mais específica é removida e a violação é registrada.

## Spans

Nome: `allascode.stage <stage.name>`. Kind padrão `INTERNAL`; adapters de transporte usam `CLIENT`, `SERVER`, `PRODUCER` ou `CONSUMER`. Status fica `UNSET`/`OK` em sucesso e `ERROR` quando a etapa entrou em healing. O erro final do Intent continua governado pelo Runtime.

## Grafana

- logs: Loki;
- métricas: Mimir ou Prometheus compatível;
- traces: Tempo;
- correlação: derived fields de `trace_id` nos logs e exemplars nas métricas;
- retenção: Compactor/retention por stream/tenant no backend e `CompactEvidenceStore` no edge.

A política numérica deste repositório é do AllasCode, não um default do OpenTelemetry ou Grafana. A compatibilidade está no modelo, na temporality, nos tipos de instrumentos, no contexto compartilhado e no processo idempotente de compaction/retention.

## Referências normativas

- OpenTelemetry Logs Data Model: https://opentelemetry.io/docs/specs/otel/logs/data-model/
- OpenTelemetry Metrics Data Model: https://opentelemetry.io/docs/specs/otel/metrics/data-model/
- OpenTelemetry Trace API: https://opentelemetry.io/docs/specs/otel/trace/api/
- OpenTelemetry Resource semantic conventions: https://opentelemetry.io/docs/specs/semconv/resource/
- Grafana Loki retention: https://grafana.com/docs/loki/latest/operations/storage/retention/
- Grafana Mimir metrics retention: https://grafana.com/docs/mimir/latest/configure/configure-metrics-storage-retention/


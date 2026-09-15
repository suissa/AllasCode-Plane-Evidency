# ProjectStageLog

## Identity

- `action_id`: `evidency.project-stage-log.v1`
- owner Behavior: `EvidencyAgent.ObserveStage`
- purpose: projetar uma observação persistida em exatamente um OTel LogRecord.

## Contrato

Input: `{observation_id, record_hash}`. Output: `{log_record_id, log_watermark, duplicate}`. Lê somente a observação validada e a política; escreve somente na projeção de logs. Effect identity: `observation_id + /log/v1`.

Mapeie timestamps, trace context, severidade, Resource, InstrumentationScope, Attributes e EventName conforme `docs/otel-grafana-contract.md`. O Body é template estável; diagnóstico livre fica sanitizado nos atributos. Capabilities: leitura do ledger e append local na projeção.

## Invariantes, eventos e healing

- um LogRecord por observação/versão;
- `span_id` implica `trace_id`;
- IDs livres não viram labels Loki;
- o watermark só avança após escrita durável.

Evento terminal: `EvidencyAgent.ObserveStage.Ok|Error`. Códigos: `SOURCE_NOT_FOUND`, `LOG_MAPPING_INVALID`, `CARDINALITY_POLICY_VIOLATION`, `STORE_UNAVAILABLE`. Replay e reconstrução da projeção são seguros; corrupção da fonte é quarentenada.

## Acceptance

Started/info, completed/info, healed/warn, failed/error, replay, trace ausente permitido, `span_id` sem trace rejeitado, label proibida e crash/restart.


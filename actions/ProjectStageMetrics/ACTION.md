# ProjectStageMetrics

## Identity

- `action_id`: `evidency.project-stage-metrics.v1`
- owner Behavior: `EvidencyAgent.ObserveStage`
- purpose: aplicar uma única vez os deltas métricos derivados da transição.

## Contrato

Input: `{observation_id, record_hash}`. Output: `{metric_watermark, updated_instruments[], duplicate}`. Effect identity: `observation_id + /metrics/v1`. Lê a observação/política; escreve apenas no acumulador e journal de métricas.

Em `started`, some `+1` a `allascode.stage.active` e registre tamanho de input. Em término, some `-1`, incremente executions e, quando aplicável, errors/retries/healing; observe duração e tamanho de output. Vincule o exemplar ao trace/span sem criar atributos de série.

## Invariantes, eventos e healing

- o journal de effect identity é gravado atomicamente com os deltas;
- replay não altera soma, count ou buckets;
- `active` não permanece negativo; inconsistência abre reconciliação de spans órfãos;
- labels pertencem à allowlist e a valores de catálogo do deploy;
- duração é não negativa e calculada em nanos monotônicos quando disponível.

Evento terminal: `EvidencyAgent.ObserveStage.Ok|Error`. Códigos: `METRIC_MAPPING_INVALID`, `DUPLICATE_DELTA`, `NEGATIVE_ACTIVE_COUNT`, `CARDINALITY_LIMIT`, `STORE_UNAVAILABLE`.

## Acceptance

Início/término normal, erro, healed, retry, replay, dimensão proibida, estouro de cardinalidade, duração inválida e crash/restart atômico.


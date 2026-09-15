# PrepareOtlpExportBatch

## Identity

- `action_id`: `evidency.prepare-otlp-export-batch.v1`
- owner Behavior: `EvidencyAgent.PrepareTelemetryExport`
- purpose: materializar lote OTLP local pronto para um adapter autorizado.

## Contrato

Input: `{from_watermark, max_records, max_bytes, signals[]}`. Output: `{batch_id, to_watermark, record_count, byte_count, payload_ref, payload_hash}`. Effect identity: hash do intervalo, sinais e schema version.

Lê apenas projeções confirmadas; escreve outbox local. Limites padrão: 512 registros ou 1 MiB, o que ocorrer primeiro. Ordena por Resource, InstrumentationScope, signal e sequence. A Action não abre socket nem confirma envio; o adapter de transporte recebe capability separada e grava ACK por batch/hash.

## Invariantes, eventos e healing

- lote é imutável e reproduzível;
- watermark não pula registro elegível;
- ACK só vale para o mesmo `payload_hash`;
- falha remota nunca bloqueia o resultado de negócio;
- credenciais e endpoint não entram no lote.

Evento terminal: `EvidencyAgent.PrepareTelemetryExport.Ok|Error`. Códigos: `PROJECTION_GAP`, `OTLP_ENCODING_FAILED`, `BATCH_TOO_LARGE`, `STORE_UNAVAILABLE`. Retry produz o mesmo batch.

## Acceptance

Lote cheio por count, por bytes, intervalo vazio, replay, gap, hash de ACK divergente, adapter negado e restart com outbox pendente.

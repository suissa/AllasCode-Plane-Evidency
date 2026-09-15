# Catálogo de AtomicActions

| Ordem | Action | Responsabilidade única | Autoridade de escrita |
|---:|---|---|---|
| 1 | `AppendStageObservation` | anexar uma transição canônica idempotente | observation ledger local |
| 2 | `ProjectStageLog` | materializar um OTel LogRecord | log projection |
| 3 | `ProjectStageMetrics` | aplicar deltas OTel uma única vez | metric projection |
| 4 | `EvaluateSnapshotThreshold` | decidir deterministicamente se snapshot venceu | snapshot decision record |
| 5 | `CreateEvidenceSnapshot` | selar estado e watermarks | snapshot candidate |
| 6 | `VerifyEvidenceSnapshot` | provar integridade/replay fora da autoridade do criador | verification record |
| 7 | `CompactEvidenceStore` | marcar e varrer registros elegíveis | retention markers/local store |
| 8 | `PrepareOtlpExportBatch` | preparar lote local OTLP, sem rede | export outbox |

A composição pertence ao Agent/Actor. As Actions não fazem rede, não configuram retries globais e não alteram Intent ou evento de negócio.


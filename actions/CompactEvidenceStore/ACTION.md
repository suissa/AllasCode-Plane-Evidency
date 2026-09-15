# CompactEvidenceStore

## Identity

- `action_id`: `evidency.compact-evidence-store.v1`
- owner Behavior: `EvidencyAgent.CompactEvidence`
- purpose: liberar armazenamento apenas para registros cobertos e expirados.

## Contrato

Input: `{verified_snapshot_id, policy_version, now}`. Output: `{marker_id, eligible_count, swept_count, bytes_reclaimed, watermark}`. Effect identity: `snapshot_id + policy_version + retention_epoch`.

Fase 1 marca registros que simultaneamente: estão `<= snapshot.watermark`, ultrapassaram a retenção efetiva para o nível atual de pressão, possuem todas as projeções, não estão em quarentena/não finalizados e, para error/fatal/security/audit/healing, têm ACK de exportação. Em pressão >= 85%, somente debug, info-success e raw metrics podem usar o floor reduzido da política. Fase 2 varre após `deletion_delay: 2h`. O marker é durável e retomável.

## Invariantes, eventos e healing

- candidato ou snapshot inválido nunca autoriza delete;
- manter ao menos dois snapshots verificados;
- mark e sweep são idempotentes e têm agendas independentes;
- índices/referências são atualizados antes do payload ser varrido;
- dados após watermark, quarentena e spans abertos são intocáveis.

Evento terminal: `EvidencyAgent.CompactEvidence.Ok|Error`. Códigos: `SNAPSHOT_NOT_VERIFIED`, `RETENTION_NOT_MET`, `EXPORT_NOT_ACKED`, `REFERENCE_STILL_LIVE`, `SWEEP_FAILED`.

## Acceptance

Nada elegível, sucesso, replay, candidato não verificado, ACK ausente, referência viva, cancelamento dentro de 2h, crash entre mark/sweep e preservação dos dois snapshots.

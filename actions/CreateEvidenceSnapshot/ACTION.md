# CreateEvidenceSnapshot

## Identity

- `action_id`: `evidency.create-evidence-snapshot.v1`
- owner Behavior: `EvidencyAgent.SnapshotEvidence`
- purpose: selar uma visão consistente do ledger, projeções e watermarks.

## Contrato

Input: `{decision_id, upper_sequence, policy_version}`. Output: `EvidenceSnapshot` conforme schema. Effect identity: `partition + upper_sequence + policy_version`. Lê ledger/projeções até uma barreira MVCC; escreve somente `snapshot_candidate` comprimido em zstd. Sem apagar dados.

O snapshot inclui contagens de logs, agregados métricos, raízes de traces, watermarks, manifesto de quarentena, hash anterior e BLAKE3 do conteúdo canônico.

## Invariantes, eventos e healing

- todos os watermarks são `<= upper_sequence`;
- a leitura representa uma única barreira consistente;
- replay produz o mesmo content hash;
- status inicial é `candidate`, nunca `verified`;
- no mínimo dois snapshots verificados anteriores são preservados.

Evento terminal: `EvidencyAgent.SnapshotEvidence.Ok|Error`. Códigos: `INCONSISTENT_BARRIER`, `HASH_MISMATCH`, `COMPRESSION_FAILED`, `STORE_UNAVAILABLE`. Retry é seguro; candidato parcial é ignorado e posteriormente varrido.

## Acceptance

Snapshot normal, replay, writes concorrentes após a barreira, candidato parcial, hash anterior ausente, capability negada e restart.


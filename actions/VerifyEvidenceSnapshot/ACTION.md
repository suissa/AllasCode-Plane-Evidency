# VerifyEvidenceSnapshot

## Identity e independência

- `action_id`: `evidency.verify-evidence-snapshot.v1`
- owner Behavior: `EvidenceVerifierAgent.VerifySnapshot`
- purpose: provar que um candidato representa fielmente o ledger e suas projeções.

O `EvidenceVerifierAgent` não possui capability para criar snapshot, mover watermark ou compactar. O criador não possui capability para marcar `verified`.

## Contrato

Input: `{snapshot_id, expected_policy_version}`. Output: `{snapshot_id, status, verification_hash, checks[], verified_at}`. Effect identity: `snapshot_id + verifier_version`. Lê candidato, ledger e projeções; escreve somente o registro de verificação.

Recalcule hash e cadeia, reexecute agregados de uma amostra determinística e, no primeiro snapshot e depois a cada 24 snapshots, faça replay integral do intervalo. Verifique watermarks, contagens, quarentena e ausência de sequence gaps.

## Invariantes, eventos e healing

`verified` exige todos os checks obrigatórios. Falha nunca reclassifica silenciosamente o candidato. Evento terminal: `EvidenceVerifierAgent.VerifySnapshot.Ok|Error`. Códigos: `CONTENT_HASH_MISMATCH`, `CHAIN_BROKEN`, `WATERMARK_INVALID`, `AGGREGATE_MISMATCH`, `SEQUENCE_GAP`.

Falha bloqueia compactação, preserva o intervalo e abre Human-in-the-Healing-Loop com hashes, intervalo e checks — nunca dados secretos.

## Acceptance

Candidato válido, bit flip, hash anterior errado, watermark adiantado, aggregate errado, gap de sequência, verifier sem capability e replay integral periódico.


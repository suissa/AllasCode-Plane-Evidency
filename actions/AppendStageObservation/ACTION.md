# AppendStageObservation

## Identity

- `action_id`: `evidency.append-stage-observation.v1`
- owner Behavior: `EvidencyAgent.ObserveStage`
- purpose: anexar exatamente uma transição de execução ao ledger local.

## Micro-skill e contrato

Use exclusivamente no hook automático do Runtime. Valide `schemas/stage-observation.schema.json`, calcule a chave idempotente, recuse sequência regressiva, redija campos proibidos, calcule o hash encadeado e faça append atômico.

Input: `StageObservation`. Output: `{observation_id, sequence, record_hash, duplicate}`. Lê o último sequence/hash da partição; escreve um registro append-only. Capabilities: RAM limitada e append/sync no ledger local. Sem rede.

## Invariantes

- mesma chave produz mesmo `observation_id` e não cria segundo efeito;
- `sequence = previous.sequence + 1` por partição;
- `record_hash` cobre o registro canônico e `previous_record_hash`;
- uma transição terminal exige início correspondente ou marcação `orphan_recovered=true` após restart;
- input original e segredos nunca são incorporados.

## Eventos e healing

O Runtime emite `EvidencyAgent.ObserveStage.Ok` ao fim do Behavior. Erros usam `EvidencyAgent.ObserveStage.Error` com `{code, safe_diagnostic, original_input_ref, healing_context}`. Códigos: `INVALID_OBSERVATION`, `SEQUENCE_CONFLICT`, `CAPABILITY_DENIED`, `STORE_UNAVAILABLE`, `REDACTION_FAILED`.

Retry é seguro com a mesma chave. Conflito de sequência reabre leitura e revalida. Schema/redação não resolvidos vão para quarentena e Human-in-the-Healing-Loop.

## Acceptance

Sucesso, replay duplicado, input inválido, sequência concorrente, capability negada, crash antes/depois do fsync e detecção de segredo.


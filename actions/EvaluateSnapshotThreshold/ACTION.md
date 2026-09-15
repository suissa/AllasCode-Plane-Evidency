# EvaluateSnapshotThreshold

## Identity

- `action_id`: `evidency.evaluate-snapshot-threshold.v1`
- owner Behavior: `EvidencyAgent.ObserveStage`
- purpose: decidir se qualquer limite de snapshot foi atingido sem criar o snapshot.

## Contrato

Input: `{policy_version, current_counters, store_size_bytes, last_snapshot}`. Output: `{due, trigger, evaluated_at, policy_version}`. Effect identity: hash do input normalizado. Lê contadores e política; escreve apenas o registro de decisão.

`due=true` quando existir ao menos um registro novo e ocorrer primeiro: 10.000 etapas concluídas, 15 minutos, 64 MiB não compactados ou 70% de 256 MiB. `trigger` registra todos os limites alcançados e elege por prioridade `store_percent > uncompacted_bytes > elapsed > count`.

## Invariantes, eventos e healing

Não cria, verifica nem apaga dados. Mesma visão/política produz a mesma decisão. Relógio regressivo usa monotonic elapsed. Evento terminal: `EvidencyAgent.ObserveStage.Ok|Error`; códigos `POLICY_INVALID`, `COUNTER_REGRESSION`, `CLOCK_INVALID`.

## Acceptance

Teste imediatamente abaixo/em cada limite, limites simultâneos, replay e clock regressivo.

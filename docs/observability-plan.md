# Plano normativo de observabilidade

## 1. Escopo e autoridade

O Plane Evidency observa o runtime; ele não altera Intent, payload de negócio nem resultado de Action. O Runtime é a única autoridade que intercepta o ciclo de vida das etapas e injeta automaticamente o Behavior `EvidencyAgent.ObserveStage`.

São observadas as etapas `Intake`, `Resolver`, `Binding`, `Healing`, `Proof`, `Governor`, `Orchestration`, `Acceptance` e `Persistence`, além de cada execução de Agent, Actor, Action, passo de healing e obrigação de prova. Uma etapa produz exatamente uma transição `started` e exatamente uma transição terminal `completed` ou `failed` por `attempt`.

A observabilidade é efeito colateral supervisionado: não muda a linearidade do fluxo de negócio, não decide sucesso e não inventa evento de domínio.

## 2. Modelo de execução automática

Para cada transição, o hook do Runtime MUST:

1. propagar o `trace_id` criado pelo `GatewayAgent` ou `UIAgent`;
2. criar um `span_id` por tentativa de etapa e preservar `parent_span_id`;
3. executar `AppendStageObservation` antes de qualquer projeção;
4. projetar log em toda transição;
5. incrementar `stage.active` em `started` e decrementar na transição terminal;
6. projetar contadores, histogramas e exemplar apenas na transição terminal;
7. avaliar os limites de snapshot após persistir as projeções;
8. encaminhar falhas de instrumentação ao self-healing sem substituir o resultado de negócio.

A chave idempotente é:

```text
BLAKE3(trace_id || span_id || stage.name || attempt || transition)
```

Repetições com a mesma chave retornam o `observation_id` existente e não incrementam métricas novamente.

## 3. Valores obrigatórios por observação

### Identidade e causalidade

| Campo | Regra |
|---|---|
| `observation_id` | ULID/UUID ordenável localmente; nunca usado como label métrica |
| `event_id` | ID do evento causal quando existir |
| `trace_id` | 16 bytes/32 hex; nasce no GatewayAgent ou UIAgent |
| `span_id` | 8 bytes/16 hex; único por tentativa |
| `parent_span_id` | span causal imediato, quando existir |
| `attempt` | inteiro `>= 1` |
| `sequence` | inteiro monotônico por partição/Actor |
| `idempotency_key` | hash determinístico da transição |

### Tempo e resultado

| Campo | Regra |
|---|---|
| `timestamp_unix_nano` | decimal string do tempo no ponto de origem |
| `observed_timestamp_unix_nano` | decimal string do tempo em que o Plane Evidency recebeu o registro |
| `start_time_unix_nano` | decimal string obrigatório no término |
| `duration_nano` | decimal string obrigatório no término e igual a `timestamp - start_time` |
| `transition` | `started`, `completed` ou `failed` |
| `result` | ausente em `started`; `ok`, `error` ou `healed` no término |
| `error.type` | código estável de catálogo; nunca mensagem livre |
| `error.message` | diagnóstico sanitizado, somente em log/evidência |

### Identidade semântica AllasCode

`agent.name`, `intent.name`, `behavior.canonical_label`, `actor.name`, `action.name`, `stage.name`, `stage.kind`, `supervisor.name`, `runtime.version`, `schema.version`, `deployment.environment.name` e `tenant.scope`.

O input original não é copiado. Use `original_input_ref` e hashes. Segredos, tokens, chaves, conteúdo de mensagem e PII não podem ser armazenados sem política explícita de classificação/redação.

## 4. Limites por registro

- corpo do log: 4 KiB;
- stack sanitizada: 8 KiB somente para erro;
- no máximo 32 atributos por log e 12 atributos por ponto métrico;
- chave: 64 bytes; valor textual: 256 bytes;
- payload total da observação: 16 KiB;
- valores excedentes são truncados com `allascode.telemetry.truncated=true` e hash do valor integral; nunca ocorre truncamento silencioso.

## 5. Ordem e consistência

`AppendStageObservation` é a fonte local da projeção. `ProjectStageLog` e `ProjectStageMetrics` mantêm watermarks independentes e são replayáveis. O sucesso do negócio não depende de exportação remota. Um adaptador OTLP autorizado consome lotes preparados localmente e confirma seu próprio watermark.

A diferença entre os dois timestamps é preservada para detectar relógio incorreto. Ordenação causal usa `sequence` e parentesco de spans, nunca apenas wall clock.

Este snapshot pertence à observabilidade e não substitui o snapshot de estado do Actor/Event Sourcing configurado com `snapshot_every=10`. Os dois possuem contadores, finalidades e políticas de retenção independentes.

## 6. Falhas e healing

- duplicata: retornar o registro existente;
- schema inválido: colocar em quarentena com referência ao input original;
- falta de espaço: compactar dados elegíveis, usar a reserva de emergência e reduzir retenção de sucesso, nunca omitir a geração de erro/healing;
- exporter indisponível: manter outbox local e backoff; não bloquear o Behavior de negócio;
- cardinalidade acima do limite: remover dimensão não permitida e registrar `CARDINALITY_POLICY_VIOLATION`;
- snapshot inválido: proibir compactação e encaminhar ao `EvidenceVerifierAgent`/Human-in-the-Healing-Loop.

## 7. Aceitação sistêmica

1. Toda etapa iniciada possui um término ou é classificada como órfã após restart.
2. Todo log de etapa contém contexto de trace quando esse contexto existe.
3. Uma tentativa concluída incrementa uma única vez o contador e um bucket de duração.
4. Replay não duplica logs nem métricas.
5. IDs livres jamais aparecem como labels de métricas ou labels indexadas do Loki.
6. Compactação falha fechada sem snapshot verificado.
7. Restart retoma a partir dos watermarks persistidos.

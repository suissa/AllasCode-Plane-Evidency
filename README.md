# AllasCode Plane Evidency

Plano normativo de evidência e observabilidade do runtime AllasCode. Cada transição de etapa é capturada automaticamente pelo runtime e projetada nos sinais OpenTelemetry, pronta para consulta e correlação no Grafana.

## Garantias

- nenhuma Action de negócio precisa chamar logger ou cliente de métricas;
- toda etapa gera observação de início e de término (`completed` ou `failed`);
- logs, métricas e spans compartilham `trace_id`, `span_id` e identidade semântica;
- métricas não recebem IDs livres nem mensagens como labels;
- snapshot, verificação e compactação são autoridades separadas;
- nenhum registro é apagado antes de snapshot verificado e watermark seguro;
- o orçamento local padrão é adequado ao baseline de 1 vCPU/1 GB de RAM.

## Documentos

- [Plano de observabilidade](docs/observability-plan.md)
- [Contrato OpenTelemetry e Grafana](docs/otel-grafana-contract.md)
- [Behaviors declarativos consumidos pelo Runtime](docs/config-driven-behaviors.md)
- [Política de armazenamento, snapshot e retenção](policies/observability.yaml)
- [Catálogo de Actions](actions/README.md)
- [Behavior declarativo de observação](behaviors/EvidencyAgent.ObserveStage/behavior.yaml)
- [Contrato de integração com o Runtime Zig](runtime/zig-behavior-runtime.md)
- [Schema da observação de etapa](schemas/stage-observation.schema.json)
- [Schema do snapshot](schemas/evidence-snapshot.schema.json)
- [Schema da config de Behavior](schemas/behavior-config.schema.json)
- [Schema do plano compilado](schemas/compiled-behavior-plan.schema.json)

## Fluxo obrigatório

```text
-> stage_transition
->> AppendStageObservation
<<- observation_id
->> [ProjectStageLog, ProjectStageMetrics]
<<- projection_result
->> EvaluateSnapshotThreshold
<<- snapshot_decision
<- EvidencyAgent.ObserveStage.Ok
```

Se qualquer Action retornar `Error`, o Runtime a encaminha ao pipeline de self-healing. O evento terminal do Behavior permanece exclusivamente `EvidencyAgent.ObserveStage.Ok|Error`; nomes de Actions nunca entram no tipo do evento.

## Compatibilidade

O modelo lógico acompanha OTLP/OpenTelemetry para logs, métricas, traces, Resource e InstrumentationScope. Loki, Mimir, Tempo e Grafana são projeções/backends substituíveis, não autoridades do domínio.

Behaviors são definidos declarativamente em YAML. O Runtime Zig valida e compila a config em um plano imutável, resolve apenas Actions canônicas já registradas e injeta dinamicamente o Agent anterior a partir do 2flow do Intent.


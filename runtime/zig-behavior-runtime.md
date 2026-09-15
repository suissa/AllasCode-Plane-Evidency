# Contrato do Runtime Zig para Behaviors

Este documento define a fronteira que o Runtime Zig 0.16 implementará. Não adiciona outro runtime nem transfere execução para TypeScript.

## Tipos conceituais

```zig
pub const BehaviorRuntime = struct {
    registry: *const ActionRegistry,
    policy: *const RuntimePolicy,
    plans: PlanStore,

    pub fn loadBehavior(
        self: *BehaviorRuntime,
        config_path: []const u8,
    ) LoadBehaviorResult;

    pub fn activatePlan(
        self: *BehaviorRuntime,
        plan_id: PlanId,
    ) ActivatePlanResult;

    pub fn dispatch(
        self: *BehaviorRuntime,
        canonical_label: CanonicalLabel,
        trigger: TriggerEnvelope,
        injected: InjectedContext,
    ) BehaviorOutcome;
};
```

Resultados são unions explícitas de estado interno, nunca exceptions não governadas. Um resultado interno de erro entra no Healing; o chamador externo recebe apenas o evento canônico governado.

## Pipeline de load

```text
read bytes
-> parse YAML
-> validate schema
-> canonicalize
-> verify hash/signature
-> parse 2flow
-> resolve ActionRegistry
-> intersect capabilities
-> type-check edges
-> build DAG
-> persist CompiledBehaviorPlan
```

O parser YAML e o compilador 2flow são componentes do Runtime. Eles não executam tags YAML, aliases recursivos, includes arbitrários ou callbacks.

## Estruturas obrigatórias

`CompiledBehaviorPlan` contém somente dados resolvidos:

- `plan_id`, `config_hash`, `flow_hash`, `registry_hash`;
- canonical label e versão;
- nodes com `action_id`, versão, schema hashes e capabilities efetivas;
- edges causais e grupos paralelos;
- contrato de trigger/input;
- política de supervision/healing;
- contrato fixo de eventos;
- modo de observabilidade e limite de reentrada.

O plano não contém ponteiro serializado, endereço de função, endpoint ou credencial.

## Dispatch

Pseudocódigo normativo:

```text
plan = plans.pinActive(canonical_label)
input = validateWithoutMutating(trigger.payload, plan.input_schema)
context = inject(trace, event, upstream_agent, plan_id, config_hash)

for frontier in plan.readyFrontiers():
  results = supervisor.executeBounded(frontier, input, context)
  observeEveryResultNonRecursively(results)
  if any Error:
    healed = healing.run(last_error, context)
    if not healed:
      emit Error to injected upstream_agent
      return

assert plan.hasNoRemainingActions()
emit Ok
```

## Persistência e cache

O cache do plano é uma otimização, não autoridade. Após restart, o Runtime só reutiliza um plano quando config, flow, registry, schemas, skills, policy e compiler version continuam com os mesmos hashes. Caso contrário, recompila.

O EventStore registra `plan_id`, `config_hash` e versões das Actions em todo início de Behavior. Isso torna replay e auditoria independentes da config atualmente ativa.

## Invariantes do executor

- uma instância de Action tem um Supervisor;
- cada node é consumido no máximo uma vez por effect identity;
- nenhum node executa antes de todas as dependências;
- grupos paralelos respeitam o limite do plano;
- o Intent original nunca é normalizado durante execução;
- o upstream Agent vem somente do contexto injetado;
- nenhum Behavior desliga a observação automática;
- somente o Behavior de Evidency assinado pode usar sink não reentrante.


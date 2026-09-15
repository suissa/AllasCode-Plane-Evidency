# Behaviors declarativos

## Decisão

Um Behavior AllasCode é uma composição versionada de Actions canônicas. Sua estrutura pode ser declarada em config, mas a config não contém código executável, endereço de função, comando shell, endpoint, credencial ou nome estático do Agent anterior.

A config informa **o que compor**. O Runtime continua sendo a única autoridade capaz de resolver, supervisionar e executar Actions.

## Artefatos

Cada Behavior possui:

```text
behaviors/<Agent>.<Intent>/
  behavior.yaml
flows/<Agent>.<Intent>.2flow
```

`behavior.yaml` segue `schemas/behavior-config.schema.json`. O 2flow é compilado para `schemas/compiled-behavior-plan.schema.json`.

## Ciclo do Runtime

### 1. Descoberta

No startup ou reload governado, o Runtime lê `behaviors/**/behavior.yaml` exclusivamente de uma raiz autorizada em disco. Symlinks que escapem da raiz, URLs e paths absolutos são rejeitados.

### 2. Validação

Antes de resolver qualquer Action, o Runtime MUST:

1. validar o JSON Schema e `api_version`;
2. verificar que `canonical_label == agent_name + "." + intent_name`;
3. preservar o documento original para auditoria;
4. normalizar apenas dentro desta fronteira de validação;
5. calcular BLAKE3 sobre a representação canônica da config e dos arquivos referenciados;
6. exigir assinatura Ed25519 em produção;
7. rejeitar campos desconhecidos.

### 3. Resolução

Cada token chamado pelo 2flow precisa existir em `action_bindings`. O Runtime resolve `action_id + version` no `ActionRegistry`; o registro só é válido quando possui contrato, micro-skill, schemas e capabilities.

A config nunca carrega uma biblioteca nem escolhe uma função por nome. Ela referencia uma Action que já faz parte do build/registro confiável do Runtime.

### 4. Compilação

O compilador 2flow produz um DAG imutável e MUST provar:

- todas as chamadas possuem binding;
- todos os bindings são usados;
- inputs/outputs adjacentes são compatíveis;
- paralelismo não viola linearidade do Actor;
- nenhum ciclo existe sem construção explícita e limitada da linguagem;
- nenhuma Action recebe capability de rede;
- o limite `max_parallel_actions` é respeitado;
- eventos são derivados do Behavior e não configurados livremente.

O resultado é identificado por:

```text
plan_id = BLAKE3(config_hash || action_registry_hash || compiler_version)
```

### 5. Injeção de contexto

Ao instanciar o Behavior dentro do 2flow de um Intent, o Runtime injeta:

- `upstream_agent_name`, obtido da aresta anterior do Intent;
- `trace_id`, criado pelo GatewayAgent ou UIAgent;
- `event_id`, quando houver evento causal;
- `plan_id` e `config_hash`;
- capabilities efetivas.

`upstream_agent_name` não existe no arquivo de Behavior. Portanto, o mesmo Behavior pode ser reutilizado em diferentes Intents sem conhecer quem o chamará.

### 6. Autoridade efetiva

Capabilities são reduzidas por interseção:

```text
effective =
  behavior_declared
  ∩ action_manifest
  ∩ runtime_policy
  ∩ caller_delegation
```

Config só reduz autoridade; nunca cria autoridade que não exista no manifesto da Action ou na política do Runtime.

### 7. Execução

Para cada trigger aceito, o Runtime:

1. seleciona o plano ativo e fixa `plan_id` no trace;
2. valida o payload contra `input.schema_ref`;
3. cria um Actor e Supervisor por Action chamada;
4. executa nós prontos na ordem causal do DAG;
5. executa em paralelo somente os nós declarados entre `[]`;
6. persiste effect identity antes de confirmar efeito;
7. injeta automaticamente observação de início e término em cada nó;
8. encaminha `Error` de Action ao pipeline de self-healing;
9. emite `{Agent}.{Intent}.Ok` somente quando não restar Action;
10. após healing esgotado, emite `{Agent}.{Intent}.Error` ao Agent anterior injetado.

O Runtime não lê novamente a config durante uma execução. Todo trace termina com a mesma versão do plano com que começou.

## Reload sem mutação de Intent

Alterar o YAML cria uma nova revisão de Behavior; não altera a instância em execução nem o Intent original.

```text
discovered -> validated -> compiled -> staged -> active -> retired
                  \-> rejected
```

Somente um plano fica `active` por `canonical_label + version`. Novas execuções recebem o plano ativo; execuções existentes mantêm o plano fixado. Planos antigos só são removidos quando não possuem traces ativos e seu período de auditoria terminou.

## Recursão da observabilidade

`EvidencyAgent.ObserveStage` observa todas as etapas, inclusive suas próprias Actions. Reexecutá-lo para observar a si mesmo criaria recursão infinita.

O Runtime mantém `observation_depth`:

- `0`: executa normalmente o Behavior configurado;
- `1` dentro de `EvidencyAgent.ObserveStage`: grava log, métrica e span em um sink bootstrap interno, append-only e não reentrante;
- `>1`: invariant violation, interrompe reentrada e aciona self-healing.

O sink bootstrap usa o mesmo schema de observação, mas não executa Behavior nem Action. Ele é uma primitiva mínima do Runtime, assim como parser, scheduler e supervisor. A config só pode solicitar `non_reentrant_bootstrap_sink` quando o `canonical_label` é exatamente `EvidencyAgent.ObserveStage` e a assinatura é confiável.

## Erros estáveis

| Código | Significado |
|---|---|
| `BEHAVIOR_SCHEMA_INVALID` | config não satisfaz o schema |
| `CANONICAL_LABEL_MISMATCH` | label não deriva de Agent + Intent |
| `FLOW_BINDING_MISSING` | 2flow chama Action não vinculada |
| `UNUSED_ACTION_BINDING` | binding não utilizado |
| `ACTION_NOT_REGISTERED` | Action canônica não existe no registry |
| `SKILL_NOT_FOUND` | Action sem micro-skill |
| `CAPABILITY_ESCALATION` | config solicitou autoridade não concedida |
| `FLOW_TYPE_MISMATCH` | output não alimenta input seguinte |
| `PLAN_SIGNATURE_INVALID` | integridade/autoria não verificável |
| `OBSERVABILITY_REENTRY` | tentativa de recursão acima do limite |

Todos seguem o pipeline de self-healing; nenhum erro bruto do parser ou registry é exposto como resultado final.

## Critérios de aceitação

1. config válida compila sempre no mesmo `plan_id`;
2. campo desconhecido falha antes de resolver Actions;
3. Action sem skill não pode entrar no plano;
4. capability excedente é rejeitada;
5. mudança de config não altera trace em andamento;
6. upstream Agent não pode ser persistido na config;
7. evento Ok só aparece após esgotar o DAG;
8. reentrada da observabilidade usa o sink bootstrap e termina em profundidade 1;
9. restart recompila ou carrega cache apenas se todos os hashes coincidirem;
10. replay fixa a mesma config e versões de Actions.


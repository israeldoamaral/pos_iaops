# Questão 08 - Postmortem técnico de incidente em produção

## Entrega

---

# Prompt (Framework R-T-F)

```text
[ROLE]
Você é um SRE Sênior com experiência em sistemas financeiros de alta
disponibilidade e domínio da metodologia Google SRE de postmortem.
Seu output deve ser direto, técnico e orientado a decisão —
sem floreios, sem disclaimers.

[TASK]
Incidente ativo em produção. Pico de tráfego. Doc Brown precisa de
um postmortem técnico em 20 minutos para decidir entre duas ações:
  - Opção A: Rollback do deploy v2.48.0 (subiu ontem, 18:42 UTC)
  - Opção B: Scaling emergencial (aumentar limits do RDS e pool
             de conexões)

Analise os artefatos abaixo e produza o postmortem.

--- ARTEFATOS ---

Deploy (2026-04-23 18:42 UTC):
  chronos-api: v2.47.0 → v2.48.0
  - Adicionado endpoint POST /v2/transactions/batch
  - Refatorado cliente do Ledger (pool movido para nova biblioteca interna)
  - Bump psycopg 3.1.18 → 3.2.0
  - Reduzido timeout do Ledger de 5s para 2s

Métricas Beacon (últimos 30 min):
  timestamp               p99_ms   req/s   err%
  2026-04-24 13:30 UTC     420     1200    0.2
  2026-04-24 13:45 UTC     510     1450    0.3
  2026-04-24 14:00 UTC     780     1780    0.8
  2026-04-24 14:10 UTC    2400     2100    4.5
  2026-04-24 14:15 UTC    5200     2400    8.2
  2026-04-24 14:20 UTC    8100     2650   11.7

Logs do pod chronos-api-79c4d8b9-xk2jp:
  14:19:48 [ERROR] connection pool exhausted (max=20, active=20, waiting=147)
  14:19:49 [WARN]  query timeout after 2000ms
  14:19:49 [ERROR] POST /v2/transactions/batch failed: context deadline exceeded
  14:19:50 [ERROR] connection reset by peer
  14:19:51 [WARN]  circuit-breaker ledger-client OPEN (threshold 50%, current 87%)
  14:19:52 [ERROR] reactor: failed to publish message: chronos-api upstream error

Estado do cluster:
  - Chronos: 12/12 pods running, HPA no máximo
  - CPU: 62% | RAM: 71%
  - Conexões ativas ao Ledger: 240/250 (limite RDS)
  - Reactor (fila chronos-transactions): 50.127 msgs acumuladas,
    crescendo ~800/min, consumer lag: 18 min e aumentando

[FORMAT]
Produza o postmortem nas seções abaixo, nesta ordem, em português:

1. SUMÁRIO EXECUTIVO (máximo 3 linhas — Doc Brown lê primeiro)
2. LINHA DO TEMPO (tabela: timestamp | evento | evidência)
3. ROOT CAUSE ANALYSIS (cadeia causal completa)
4. ANÁLISE DAS OPÇÕES (tabela comparativa: Rollback vs Scaling —
   prós, contras, riscos e tempo estimado de recuperação)
5. RECOMENDAÇÃO (escolha justificada + primeiros 3 passos de execução)
6. ACTION ITEMS PÓS-INCIDENTE (mínimo 5, com owner placeholder e prazo)
```

---

# Modelo

**Modelo utilizado:** Claude Sonnet 4.6 (Anthropic)

**Justificativa da escolha:** O Claude Sonnet 4.6 foi escolhido pela combinação de raciocínio causal sobre artefatos técnicos heterogêneos (métricas, logs, changelog, estado de cluster) e geração de output estruturado sob restrições rígidas de formato. O postmortem exige que o modelo construa a cadeia causal a partir de quatro fontes distintas de evidência, produza uma análise comparativa objetiva entre duas opções de mitigação e entregue uma recomendação justificada — tudo em seis seções ordenadas consumíveis em 20 minutos por um engenheiro sob pressão de incidente. O Claude Sonnet 4.6 demonstra precisão na correlação de evidências técnicas e consistência no seguimento de estruturas de output definidas no componente Format do R-T-F.

---

# Output

## Postmortem técnico — chronos-api v2.48.0

**Data do incidente:** 2026-04-24 · **Status:** Ativo · **Severidade:** P1

---

### 1. Sumário executivo

O deploy v2.48.0 introduziu um pool de conexões subdimensionado (`max=20`) e timeout reduzido para 2 s no cliente do Ledger. Com o crescimento do tráfego, o pool saturou, o circuit-breaker abriu e a fila do Reactor acumulou 50 k mensagens em cascata. **Rollback imediato para v2.47.0 é a ação recomendada.**

---

### 2. Linha do tempo

| Timestamp (UTC) | Evento | Evidência |
|---|---|---|
| 2026-04-23 18:42 | Deploy v2.48.0 aplicado via Argo CD | Changelog: novo pool, psycopg 3.2.0, timeout 2 s |
| 2026-04-24 13:30 | Início do pico de tráfego (1.200 req/s) | Beacon: p99=420 ms, err=0.2% |
| 14:00 | Latência sobe para 780 ms, erros a 0.8% | Pool começando a saturar silenciosamente |
| 14:10 | p99 atinge 2.400 ms; erros saltam para 4.5% | Pool esgotado, timeouts de 2 s disparando em massa |
| 14:19:48 | Pool totalmente saturado (waiting=147) | Log: `connection pool exhausted max=20` |
| 14:19:51 | Circuit-breaker abre (87% de falhas) | Log: `ledger-client OPEN threshold 50%` |
| 14:19:52 | Reactor para de publicar mensagens | Log: `failed to publish — upstream error` |
| 14:20 | p99=8.100 ms, err=11.7%, lag=18 min | Degradação sistêmica em cascata confirmada |

---

### 3. Root cause analysis

**Causa raiz:** o pool de conexões do `ledger-client` foi recriado na v2.48.0 com `max=20` — valor insuficiente para a carga de pico. Com 12 pods e até 2.650 req/s, o sistema exigia até ~220 conexões simultâneas ao Ledger.

**Cadeia causal completa:**

1. **Pool subdimensionado (`max=20`):** nova biblioteca interna não herdou o dimensionamento anterior; refatoração não foi validada sob carga.
2. **Timeout reduzido de 5 s para 2 s:** queries lentas sob carga agora falham antes de liberar o slot no pool, aumentando a pressão sobre as 20 conexões disponíveis.
3. **psycopg 3.2.0 (possível agravante):** bump de versão altera o comportamento de reconexão automática — regressão não testada sob carga pode ter acelerado o esgotamento.
4. **Saturação do pool:** `waiting=147` com `active=20` confirma que 147 goroutines ficaram bloqueadas aguardando conexão.
5. **Circuit-breaker abre (87%):** com 87% de falhas, o threshold de 50% foi ultrapassado — `ledger-client` entra em estado OPEN.
6. **Reactor não consegue publicar:** sem upstream funcional, mensagens acumulam na fila `chronos-transactions` a ~800/min.
7. **Degradação total:** p99 de 420 ms → 8.100 ms em 50 minutos; taxa de erro de 0.2% → 11.7%.

---

### 4. Análise das opções

| | Rollback para v2.47.0 | Scaling emergencial (RDS + pool) |
|---|---|---|
| **Prós** | Remove a causa raiz; restaura pool e timeout anteriores; reversível e testado em produção | Não requer novo deploy; pode aliviar capacidade sem rollback de código |
| **Contras** | Requer sync Argo CD (~5 min); mensagens acumuladas no Reactor precisam ser drenadas após estabilização | Não remove a causa raiz (pool mal configurado continua no código); escalar RDS pode exigir restart ou mudança de classe de instância |
| **Riscos** | Se o endpoint `/batch` processou transações antes do rollback, verificar consistência de estado é obrigatório | psycopg 3.2.0 pode ter bug de reconexão que piora com mais conexões; tempo de efeito incerto (15–30 min) |
| **Tempo de recuperação** | ~5–10 min após trigger | ~15–30 min, com resultado incerto |

---

### 5. Recomendação

**Ação: Rollback imediato para v2.47.0.**

A causa raiz é clara e está contida no deploy de ontem. O scaling emergencial não endereça o pool subdimensionado e introduz incerteza adicional com o psycopg 3.2.0. O rollback é determinístico, reverte todas as mudanças do v2.48.0 de uma vez e tem tempo de recuperação previsível.

**Primeiros 3 passos:**

1. **Imediato — trigger do rollback via Argo CD:**
```bash
argocd app set chronos-api --revision v2.47.0
argocd app sync chronos-api --force
```

2. **Paralelo — monitorar o circuit-breaker do `ledger-client`:** aguardar transição `OPEN → HALF-OPEN → CLOSED`; validar p99 descendo abaixo de 500 ms no Beacon.

3. **Após estabilização — drenar e validar o Reactor:** o consumer lag deve reduzir naturalmente após o circuit-breaker fechar; verificar consistência das transações processadas pelo endpoint `/batch` antes do rollback.

---

### 6. Action items pós-incidente

| # | Ação | Owner | Prazo |
|---|---|---|---|
| 1 | Definir `max_pool` com fórmula baseada em carga real: `pods × req_peak / avg_query_time`; documentar e adicionar como parâmetro obrigatório no review de PRs que alterem cliente de DB | SRE + Dev | 48 h |
| 2 | Adicionar alerta no Beacon para `pool_waiting > 50` com severidade P2, antes de atingir esgotamento | SRE | 48 h |
| 3 | Reverter timeout do Ledger para 5 s ou calibrar com dados reais de p99 do Ledger em horário de pico | Dev | Próximo deploy |
| 4 | Validar psycopg 3.2.0 em staging com carga sintética equivalente ao pico de produção antes de reintroduzir | Dev | 1 semana |
| 5 | Adicionar teste de carga ao pipeline CI/CD como gate obrigatório para merges que alterem biblioteca cliente de DB ou parâmetros de pool/timeout | Platform Eng | 1 semana |
| 6 | Revisar processo de changelog review: mudanças em `pool_size` e `timeout` exigem aprovação explícita de SRE antes do merge | EM + SRE Lead | 2 semanas |

---

# Justificativa

O prompt foi estruturado utilizando o framework **R-T-F (Role, Task e Format)** — o único dos cinco frameworks do capítulo que opera simultaneamente nas três dimensões críticas para um postmortem técnico orientado à decisão em tempo real.

## Framework escolhido: R-T-F

### Role

O papel da IA é definido logo no início do prompt:

> "Você é um SRE Sênior com experiência em sistemas financeiros de alta disponibilidade e domínio da metodologia Google SRE de postmortem."

Essa definição ancora o contexto epistêmico: calibra o modelo para usar terminologia técnica precisa (circuit-breaker, pool exhaustion, consumer lag), raciocinar sobre trade-offs de disponibilidade e não desperdiçar tokens com disclaimers ou explicações introdutórias.

### Task

A seção Task entrega os quatro artefatos brutos do incidente e a pergunta de decisão binária (rollback vs. scaling emergencial) — exatamente o que Doc Brown precisa resolver em 20 minutos. Fornecer os dados brutos em vez de uma descrição narrativa permite que o modelo construa a cadeia causal a partir das evidências, sem introduzir viés interpretativo.

### Format

O Format é o componente decisivo neste cenário. A decisão não pode ser entregue como bloco de texto corrido durante um incidente ativo. O Format impõe seis seções ordenadas com hierarquia de leitura definida — sumário primeiro, análise comparativa em tabela, recomendação com passos executáveis — tornando o output consumível sob pressão.

## Comparação com frameworks candidatos

| Framework | O que se ganharia | O que se perderia | Por que preterido |
|---|---|---|---|
| **C-A-R-E** (Context · Action · Result · Example) | O componente *Example* poderia ancorar o postmortem com um incidente análogo (ex.: falha de pool em produção), adicionando contexto histórico à decisão. *Context* tem semântica explícita para dados de situação, mais legível para equipes não-técnicas. | Sem componente de *Format*, o output tende a prosa narrativa. Em 20 minutos de incidente, Doc Brown não tem tempo para ler narrativa — precisa de tabela e bullet. Sem *Role*, o modelo pode adotar tom generalista, diluindo a precisão técnica. | A ausência de *Format* é fatal para um postmortem com seis seções obrigatórias. |
| **B-A-B** (Before · After · Bridge) | A estrutura Before/After é naturalmente alinhada com a comparação de estado pré e pós-deploy — forte para comunicar o impacto de uma regressão. O componente *Bridge* forçaria o modelo a propor o caminho de recuperação de forma explícita. | B-A-B é intrinsecamente linear e sequencial. Um postmortem real tem múltiplas seções paralelas (timeline, RCA, opções, action items) que não cabem bem na tríade Before→After→Bridge. Sem *Role* e sem *Format*, o modelo não sabe o nível de detalhe esperado nem que a saída deve ser estruturada para leitura sob pressão. O framework induz o modelo a narrar o problema, não a decidir sobre ele. | A estrutura linear não comporta a complexidade do output exigido; o foco em narrativa é o errado para esse cenário. |
| **T-A-G** (Task · Action · Goal) | Útil para definir OKRs e tarefas de projeto com objetivo claro. | Sem *Format*, o output tende a texto corrido. Em um postmortem com seis seções obrigatórias, a ausência de estrutura de saída é fatal. | Descartado por não estruturar o output. |
| **R-I-S-E** (Role · Input · Steps · Expectation) | Segundo colocado. O componente *Steps* poderia enumerar as seções do postmortem; *Input* tem semântica explícita para os artefatos. | A diferença decisiva: *Steps* descreve o processo de raciocínio do modelo, enquanto *Format* descreve a estrutura do output entregue ao leitor — exatamente o que importa aqui. R-T-F é mais preciso para esse objetivo. | Preterido por uma distinção semântica: o cenário exige controle do output, não do processo de raciocínio. |

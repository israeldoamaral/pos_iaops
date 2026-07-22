# Questão 07 - Runbook para alerta recorrente

## Entrega

---

# Prompt (Framework R-I-S-E)

```text
[ROLE]
Você é um SRE sênior com expertise em Kubernetes/EKS e gestão de incidentes de produção. Sua especialidade é criar runbooks procedurais que qualquer plantonista — inclusive sem conhecimento prévio do sistema — consiga executar de ponta a ponta durante um incidente às 3h da manhã.

[INPUT]
Existe um alerta recorrente que dispara ~4x por semana no canal #oncall-chronos:

  [CRITICAL] High memory usage on Chronos API pods (>85% for 10min)

Contexto técnico do ambiente:
- Serviço: Chronos API · Namespace: production · Cluster: EKS
- Réplicas: 6 com HPA configurado (min: 4, max: 12, CPU target: 70%)
- Deploy: Argo CD a partir do repositório hvt/chronos-api
- Dependências diretas: Ledger (PostgreSQL) e Reactor (filas SQS)
- Observabilidade: métricas em /metrics, logs no Beacon, dashboards no Grafana
- Ferramentas disponíveis: kubectl, aws cli, argocd cli
- Canal de plantão: #oncall-chronos no Slack
- Escalação: @chronos-core · SLA 15min (horário comercial) / 30min (fora)

[STEPS]
Produza um runbook completo seguindo obrigatoriamente esta sequência:

1. Confirmação do alerta: comandos para validar que o problema é real e não um falso positivo transitório.
2. Diagnóstico — HPA e escalonamento: verificar se o cluster está tentando compensar e se atingiu o limite máximo de réplicas.
3. Diagnóstico — dependências: verificar se Ledger ou Reactor estão causando pressão de memória indireta.
4. Mitigação imediata: ações que o plantonista pode tomar sem aprovação (restart seguro via kubectl e verificação Argo CD).
5. Observabilidade: como usar Grafana e Beacon para confirmar estabilização.
6. Critérios de escalação: lista objetiva de condições que exigem acionar @chronos-core.
7. Critério de encerramento: condições mensuráveis para fechar o incidente com segurança.

Para cada passo inclua:
- O comando exato a executar (copy-paste ready)
- O output ou estado esperado ao final do passo
- O critério binário: passou (siga para o próximo passo) ou falhou (o que fazer)

[EXPECTATION]
Entregue o runbook em Markdown estruturado com headers claros por seção.
- Comandos em blocos de código com sintaxe literal — sem placeholders como <seu-cluster>.
- Verificações binárias — sem "verifique se parece OK", sempre com threshold numérico ou estado exato.
- Critérios de escalação como lista de condições objetivas (não "se achar necessário").
- Critério de encerramento com checklist de no mínimo 4 itens verificáveis.
- Tom operacional, direto, sem explicações didáticas. O leitor está em plantão, não em treinamento.
```

---

# Modelo

**Modelo utilizado:** Claude Sonnet 4.6 (Anthropic)

**Justificativa da escolha:** O Claude Sonnet 4.6 foi escolhido por sua capacidade de gerar documentação operacional com comandos literais e condicionais precisos para ambientes Kubernetes/EKS. O runbook exige `kubectl`, `aws cli` e `argocd cli` com flags exatas, thresholds numéricos verificáveis (`> 80%`, `> 500ms`) e um checklist de encerramento funcional — não prosa explicativa. O Claude Sonnet 4.6 tende a produzir runbooks acionáveis em uma única geração quando o prompt R-I-S-E define explicitamente o critério de qualidade na seção `[EXPECTATION]`, evitando o erro comum de modelos que geram texto de tutorial onde se espera documento de on-call.

---

# Output

## Runbook: [CRITICAL] High memory usage on Chronos API pods

**Namespace:** production · **Cluster:** EKS · **Versão:** 1.0 · **Canal:** #oncall-chronos

---

### 1. Confirmação do alerta

**Verificar pods em alto consumo de memória**

```bash
kubectl top pods -n production -l app=chronos-api --sort-by=memory
```

**Esperado:** Um ou mais pods com uso >85% do limite configurado. Se nenhum pod estiver acima de 70%, o alerta pode ter sido transitório — anote e monitore por 5 minutos antes de fechar.

**Verificar status dos pods e reinícios recentes**

```bash
kubectl get pods -n production -l app=chronos-api -o wide
```

**Esperado:** Todos os pods em `Running`. RESTARTS acima de 3 nas últimas 1h indica OOMKill — vá diretamente para o passo 4.

---

### 2. Diagnóstico — HPA e escalonamento

**Verificar status do HPA**

```bash
kubectl get hpa chronos-api-hpa -n production
```

**Esperado:** REPLICAS entre 4 e 12. Se MAXREPLICAS já atingido (12/12), o sistema não consegue escalar mais — escalação obrigatória para @chronos-core.

**Verificar eventos de escalonamento**

```bash
kubectl describe hpa chronos-api-hpa -n production | tail -30
```

**Esperado:** Eventos de `ScaleUp` recentes. `FailedGetScale` ou `unable to fetch metrics` indica problema no metrics-server — anote para escalação.

---

### 3. Diagnóstico — dependências externas

**Checar conectividade com Ledger (PostgreSQL)**

```bash
kubectl logs -n production -l app=chronos-api --since=15m | grep -i "connection\|pool\|timeout\|postgres" | tail -20
```

**Esperado:** Sem erros de conexão. Erros de pool exhaustion ou connection timeout indicam gargalo no Ledger — verifique a instância RDS no console AWS.

**Checar filas SQS (Reactor)**

```bash
aws sqs get-queue-attributes \
  --queue-url $(aws sqs get-queue-url --queue-name chronos-reactor --query QueueUrl --output text) \
  --attribute-names ApproximateNumberOfMessages \
  --region us-east-1
```

**Esperado:** Fila abaixo de 1.000 mensagens. Acima de 5.000 indica backlog severo acumulando carga nos pods.

---

### 4. Mitigação imediata

**Forçar rollout para reiniciar pods com vazamento de memória**

```bash
kubectl rollout restart deployment/chronos-api -n production
```

**Esperado:** Pods reiniciados com memória zerada. Monitore por 10 minutos. Se o uso voltar a subir rapidamente, o problema é estrutural — escale para @chronos-core.

**Acompanhar o progresso do rollout**

```bash
kubectl rollout status deployment/chronos-api -n production
```

**Esperado:** `successfully rolled out` ao final. Qualquer `error` ou timeout indica falha no deploy — não force novamente, escale.

**Verificar saúde do deploy via Argo CD**

```bash
argocd app get chronos-api --grpc-web
```

**Esperado:** `Health: Healthy`, `Sync: Synced`. Status `Degraded` ou `OutOfSync` indica divergência entre o repositório `hvt/chronos-api` e o cluster — não faça rollout manual sem investigar.

---

### 5. Observabilidade

**Verificar métricas no Grafana**

```bash
# Acesse o dashboard "Chronos API — Memory" no Grafana.
# Filtre pelo namespace: production
# Endpoint de métricas dos pods: /metrics
```

**Esperado:** Gráfico de memória estabilizando após o reinício. Tendência de subida contínua após o restart indica memory leak ativo — escale para @chronos-core.

**Verificar logs no Beacon**

```bash
# No Beacon, filtre: service=chronos-api level=ERROR since=30m
```

**Esperado:** Ausência de erros novos após o rollout. Erros repetidos de heap ou GC indicam problema na aplicação, não na infraestrutura.

---

### 6. Critérios de escalação para @chronos-core

Acione `@chronos-core` no canal `#oncall-chronos` imediatamente se qualquer condição abaixo for verdadeira:

- HPA já está em 12/12 réplicas e o uso de memória continua subindo
- Pods com RESTARTS > 5 em menos de 30 minutos (OOMKill em loop)
- Erro de conexão com Ledger (PostgreSQL) confirmado nos logs
- Fila Reactor com > 5.000 mensagens acumuladas
- Argo CD em estado `Degraded` sem causa identificada
- Uso de memória volta a >85% em menos de 10 minutos após o rollout

**SLA de resposta:** 15 minutos em horário comercial · 30 minutos fora do horário

---

### 7. Critério de encerramento do incidente

Feche o incidente somente quando **todas** as condições abaixo forem satisfeitas:

- [ ] `kubectl top pods` mostra todos os pods abaixo de 70% de memória por no mínimo 15 minutos consecutivos
- [ ] Nenhum RESTART novo registrado nos últimos 15 minutos
- [ ] HPA com réplicas estáveis (sem flutuação a cada 2 minutos)
- [ ] Alerta no Beacon fechado ou em estado `resolved`

**Ao encerrar**, registre no `#oncall-chronos`: pods afetados, ação tomada e hora de resolução.

---

# Justificativa

| Elemento R-I-S-E | Como aparece no Prompt |
|---|---|
| **Role** | Seção `[ROLE]` — define o agente como SRE sênior com expertise em Kubernetes/EKS e resposta a incidentes, orientando o modelo a produzir comandos literais e linguagem operacional de on-call, não prosa didática. A restrição "qualquer plantonista sem conhecimento prévio" impede o modelo de assumir contexto implícito. |
| **Input** | Seção `[INPUT]` — fornece o contexto técnico completo: nome exato do alerta, stack (EKS, namespace production, HPA min4/max12/CPU70%), ferramentas disponíveis (kubectl, aws cli, argocd cli), dependências (Ledger/PG, Reactor/SQS), observabilidade (Beacon, Grafana, /metrics), canal e SLA de escalação. Sem isso, o modelo geraria comandos genéricos ou inventaria parâmetros. |
| **Steps** | Seção `[STEPS]` — impõe a sequência lógica do runbook em 7 etapas: confirmação → HPA → dependências → mitigação → observabilidade → escalação → encerramento. Isso previne que o modelo salte para mitigação antes do diagnóstico ou omita a verificação de dependências, que é a causa raiz mais comum em incidentes de memória. |
| **Expectation** | Seção `[EXPECTATION]` — define o formato e o critério de qualidade do output: Markdown estruturado, comandos literais copy-paste, verificações binárias com threshold numérico, critérios objetivos de escalação e checklist de encerramento. Sem esta seção, o modelo entregaria prosa explicativa útil para estudo, mas inoperável às 3h da manhã durante um incidente. |

---

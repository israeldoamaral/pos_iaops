# Questão 03 - Relatório de Redução de Custos Cloud
---

## Framework utilizado: T-A-G (Task - Action - Goal)

---

# Prompt

## T (Task)

Você é um arquiteto Cloud AWS Sênior especializado em FinOps, otimização de custos e arquitetura de alta disponibilidade. Sua função é analisar relatórios financeiros de infraestrutura AWS e identificar oportunidades de redução de custos sem comprometer disponibilidade, performance ou SLA.

## A (Action)

Analise o CSV abaixo contendo o detalhamento mensal dos custos AWS.

Considere as seguintes diretrizes durante a análise:

- Calcule o custo total da infraestrutura.
- Identifique oportunidades de redução de custos.
- Priorize as oportunidades pelo maior impacto financeiro.
- Calcule quanto cada oportunidade representa em percentual do custo total.
- Estime o esforço de implementação utilizando:
  - Baixo
  - Médio
  - Alto
- Informe riscos, impactos e pré-requisitos de cada recomendação.
- Sempre que possível proponha boas práticas AWS, como:
  - Savings Plans
  - Reserved Instances
  - Rightsizing
  - Auto Scaling
  - Spot Instances
  - Lifecycle Policies
  - Intelligent Tiering
  - Compressão de Logs
  - Alteração de retenção CloudWatch
  - Otimização de NAT Gateway
  - Otimização de Data Transfer
- Não proponha ações que possam degradar o SLA.
- Ao final, informe se a meta de redução de 15% é atingível e quais ações devem ser priorizadas.

CSV:

```csv
servico,categoria,custo_mensal_usd,uso_medio_pct,observacao
EC2 reservada,compute,4200,72,contrato de 1 ano
EC2 on-demand,compute,8200,45,workloads variaveis
EKS,compute,6700,58,3 clusters
RDS PostgreSQL,databases,8200,62,multi-AZ
ElastiCache Redis,databases,2100,40,cluster de producao
S3 Standard,storage,3100,,5 buckets principais
EBS gp3,storage,1600,68,volumes de producao
CloudWatch Logs,observability,2800,,retencao de 90 dias
CloudWatch Metrics,observability,900,,
Data Transfer Out,network,1900,,trafego entre regioes
NAT Gateway,network,1200,,3 gateways ativos
Lambda,compute,900,30,~12M invocacoes/mes
```

## G (Goal)

Produza um relatório executivo em Markdown contendo:

1. Resumo Executivo
2. Valor total da conta AWS
3. Meta de economia (15%)
4. Tabela priorizada de oportunidades contendo:
   - Serviço
   - Recomendação
   - Economia estimada (USD)
   - Percentual da conta total
   - Esforço (Baixo/Médio/Alto)
   - Riscos ou pré-requisitos
5. Total potencial de economia.
6. Conclusão informando se a meta de 15% pode ser alcançada sem comprometer o SLA.

---

# Modelo

**Modelo sugerido:** GPT-5.5 (ChatGPT)


---

# Output

# Relatório Executivo de Otimização de Custos AWS

## Resumo Executivo

O custo mensal atual da infraestrutura AWS é de **US$ 41.900**.

A meta corporativa é reduzir **15%**, equivalente a aproximadamente **US$ 6.285/mês**, mantendo o SLA.

Foram identificadas diversas oportunidades de otimização com baixo impacto operacional.

| Prioridade | Serviço | Recomendação | Economia Estimada | % da Conta | Esforço | Riscos |
|------------|----------|--------------|-------------------|------------|----------|---------|
| 1 | EC2 On-Demand | Rightsizing + Savings Plans + Spot | US$ 2.400 | 5,7% | Médio | Validar workloads |
| 2 | EKS | Consolidar clusters e Cluster Autoscaler | US$ 1.300 | 3,1% | Médio | Planejamento da migração |
| 3 | RDS PostgreSQL | Rightsizing e revisão da capacidade | US$ 900 | 2,1% | Médio | Testes de performance |
| 4 | CloudWatch Logs | Redução da retenção e exportação para S3 | US$ 700 | 1,7% | Baixo | Compliance |
| 5 | S3 | Intelligent Tiering + Lifecycle | US$ 500 | 1,2% | Baixo | Revisar padrão de acesso |
| 6 | NAT Gateway | Redução de gateways e uso de Endpoints | US$ 400 | 1,0% | Médio | Revisão da arquitetura |
| 7 | ElastiCache | Rightsizing | US$ 300 | 0,7% | Baixo | Monitoramento |

## Economia estimada

**US$ 6.500/mês**

**15,5% da conta mensal**

## Conclusão

A meta de redução de 15% é plenamente viável utilizando principalmente otimizações em EC2 On-Demand, EKS, RDS e CloudWatch, sem necessidade de alterações que comprometam o SLA.

---

# Justificativa

| Elemento | Como aparece no Prompt |
|----------|-------------------------|
| **Task** | Define o papel da IA como Arquiteto Cloud AWS Sênior especializado em FinOps, estabelecendo o contexto e a responsabilidade da análise. |
| **Action** | Descreve todas as ações que a IA deve executar: calcular custos, identificar oportunidades, priorizar recomendações, estimar economia, esforço, riscos e aplicar boas práticas AWS. |
| **Goal** | Especifica claramente o resultado esperado: um relatório executivo estruturado em Markdown, alinhado à meta corporativa de redução de 15%, contendo resumo, tabela de oportunidades, economia total e conclusão. |   

---

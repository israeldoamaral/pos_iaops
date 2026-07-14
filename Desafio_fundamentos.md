# Questão 01 – Dockerfile para o Lift

## Entrega

---

# Prompt (Framework R-T-F)

```text
# Role

Você é um Engenheiro DevOps Sênior especializado em Docker, Kubernetes, Python e segurança de containers. Possui ampla experiência na criação de imagens Docker para ambientes de produção, seguindo as melhores práticas da OCI, Docker, Kubernetes e CIS Benchmark.

# Task

Crie um Dockerfile para uma aplicação Python/Flask considerando as seguintes informações:

## Estrutura do projeto

lift/
├── app.py
├── requirements.txt
├── lib/
│   ├── auth.py
│   └── storage.py
└── tests/
    └── test_app.py

## Conteúdo do requirements.txt

Flask==3.0.0
gunicorn==21.2.0
requests==2.31.0
python-dotenv==1.0.0
psycopg2-binary==2.9.9

## Informações da aplicação

- A aplicação utiliza Flask.
- O servidor de produção é executado através do Gunicorn.
- O comando utilizado é:

gunicorn --bind 0.0.0.0:8080 --workers 4 app:app

## Requisitos obrigatórios

- Utilizar uma imagem oficial do Python.
- Preferir uma imagem slim para reduzir o tamanho final.
- Otimizar o cache de build copiando inicialmente apenas o requirements.txt.
- Instalar somente as dependências necessárias.
- Utilizar pip com --no-cache-dir.
- Não incluir a pasta tests na imagem final.
- Criar um usuário não-root para execução da aplicação.
- Definir o diretório de trabalho como /app.
- Copiar apenas os arquivos necessários para execução da aplicação.
- Expor a porta 8080.
- Declarar as variáveis de ambiente DATABASE_URL e API_KEY sem definir valores, pois serão fornecidas em tempo de execução.
- Adicionar um HEALTHCHECK adequado.
- Utilizar CMD em formato JSON para iniciar o Gunicorn.

# Format

Retorne somente
1. O Dockerfile completo.
2. Uma breve explicação (máximo de 10 linhas) descrevendo as boas práticas utilizadas.
```

---

# Modelo

**Modelo utilizado:** GPT-5.5 (ChatGPT)

---

# Output

```dockerfile
# Imagem oficial e otimizada do Python
FROM python:3.12-slim

# Evita geração de arquivos .pyc e melhora o logging
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Diretório da aplicação
WORKDIR /app

# Cria usuário sem privilégios
RUN addgroup --system app && \
    adduser --system --ingroup app app

# Copia primeiro as dependências para aproveitar o cache
COPY requirements.txt .

# Instala dependências
RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Copia apenas os arquivos necessários
COPY app.py .
COPY lib ./lib

# Ajusta permissões
RUN chown -R app:app /app

# Executa como usuário não-root
USER app

# Variáveis de ambiente fornecidas em runtime
ENV DATABASE_URL=""
ENV API_KEY=""

# Porta utilizada pela aplicação
EXPOSE 8080

# Healthcheck simples
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
CMD python -c "import requests; requests.get('http://localhost:8080')" || exit 1

# Inicialização da aplicação
CMD ["gunicorn","--bind","0.0.0.0:8080","--workers","4","app:app"]
```

---

# Justificativa

O prompt foi estruturado utilizando o framework **R-T-F (Role, Task e Format)** para fornecer contexto suficiente ao modelo e obter uma resposta objetiva e alinhada às boas práticas.

## Role

O papel da IA é definido logo no início do prompt:

> "Você é um Engenheiro DevOps Sênior especializado em Docker, Kubernetes, Python e segurança de containers."

Essa definição orienta o modelo a responder como um especialista no assunto, aumentando a qualidade técnica da solução.

## Task

A tarefa descreve claramente o problema a ser resolvido e todas as restrições do cenário:

- Estrutura do projeto.
- Dependências da aplicação.
- Comando de execução.
- Requisitos obrigatórios.
- Boas práticas esperadas.

Isso reduz ambiguidades e garante que todos os requisitos do enunciado sejam considerados na geração do Dockerfile.

## Format

A seção **Format** determina exatamente como a resposta deve ser apresentada:

1. Dockerfile completo.
2. Breve explicação das boas práticas utilizadas.

Dessa forma, evita respostas excessivamente longas ou com informações que não fazem parte da entrega solicitada.

---
---

# Questão 02 – Script de backup do Ledger

## Entrega

---

# Prompt (Framework R-T-F)

```text
# Role

Você é um Engenheiro de SRE/DevOps Sênior, especialista em administração de PostgreSQL em produção, automação Bash para Linux (Ubuntu 22.04 LTS) e integração com serviços AWS (S3, IAM Instance Profiles, Secrets Manager). Seu padrão de trabalho segue práticas de produção: tratamento de erro explícito, idempotência, logging estruturado e uso responsável de recursos de disco.

# Task

Escreva um script bash de backup automatizado do banco PostgreSQL "ledger_prod" para ser executado via cron diário, cumprindo os seguintes requisitos:

- Conectar em ledger-db.internal.hvt.io:5432, banco ledger_prod, usuário backup_user, autenticando via variável de ambiente PGPASSWORD (já populada externamente por AWS Secrets Manager através da IAM role da instância EC2, região us-east-1).
- Gerar o dump com pg_dump e compactar com gzip, sem gerar uma cópia intermediária não compactada em disco (o diretório de trabalho /var/backups/ledger tem 80GB livres e o dump compactado médio é ~12GB, então o dump bruto pode exceder o espaço disponível).
- Enviar o arquivo compactado para o bucket S3 hvt-ledger-backups via aws s3 cp.
- Aplicar retenção de 30 dias no S3, removendo automaticamente backups mais antigos que esse período.
- Registrar cada execução (início, sucesso ou falha de cada etapa) em /var/log/ledger-backup.log, com timestamp em cada linha.
- Encerrar com exit codes distintos para cada tipo de falha (ex.: pré-checagem, dump, upload, limpeza), e exit 0 apenas em sucesso completo.
- Incluir validações de pré-execução (dependências instaladas, PGPASSWORD definida, espaço em disco mínimo).

# Format

Entregue apenas o script bash completo, pronto para produção, com:
- Shebang e `set -euo pipefail`.
- Seção de configuração (variáveis no topo, em maiúsculas, com `readonly`).
- Funções nomeadas para log e tratamento de erro.
- Comentários curtos por seção (não linha a linha).
- Sem explicações fora do código — apenas o script comentado, pronto para salvar como .sh e agendar via crontab.
```

---

# Modelo

**Modelo utilizado:** Claude Sonnet 5 (Anthropic)

---

# Output

```bash
#!/usr/bin/env bash
#
# ledger-backup.sh
# Backup diário do PostgreSQL "ledger_prod" (Ledger) com upload para S3
# e retenção de 30 dias. Projetado para execução via cron.
#
# Exit codes:
#   0 - sucesso total
#   2 - pré-checagem falhou (dependência ausente / PGPASSWORD não definida)
#   3 - espaço em disco insuficiente
#   4 - falha no pg_dump / gzip
#   5 - falha no upload para S3
#   6 - upload ok, mas falha na limpeza de retenção

set -euo pipefail
umask 077

# ------------------------------------------------------------------
# CONFIGURAÇÃO
# ------------------------------------------------------------------
readonly DB_HOST="ledger-db.internal.hvt.io"
readonly DB_PORT="5432"
readonly DB_NAME="ledger_prod"
readonly DB_USER="backup_user"

readonly AWS_REGION="us-east-1"
readonly S3_BUCKET="hvt-ledger-backups"
readonly RETENTION_DAYS=30

readonly BACKUP_DIR="/var/backups/ledger"
readonly LOG_FILE="/var/log/ledger-backup.log"
readonly MIN_FREE_GB=20   # margem de segurança dentro dos 80GB disponíveis

readonly TIMESTAMP
TIMESTAMP="$(date +%Y%m%d_%H%M%S)"
readonly DUMP_BASENAME="ledger_prod_${TIMESTAMP}.sql.gz"
readonly DUMP_PATH="${BACKUP_DIR}/${DUMP_BASENAME}"

# ------------------------------------------------------------------
# LOG E TRATAMENTO DE ERRO
# ------------------------------------------------------------------
log() {
    local level="$1"; shift
    printf '%s [%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$level" "$*" >> "$LOG_FILE"
}

fail() {
    local msg="$1"
    local code="${2:-1}"
    log "ERROR" "$msg"
    exit "$code"
}

trap 'log "ERROR" "Interrupção inesperada na linha $LINENO (comando: $BASH_COMMAND)"' ERR

# ------------------------------------------------------------------
# PRÉ-CHECAGENS
# ------------------------------------------------------------------
mkdir -p "$BACKUP_DIR"
touch "$LOG_FILE" 2>/dev/null || true

if [[ -z "${PGPASSWORD:-}" ]]; then
    fail "PGPASSWORD não definida (esperado que Secrets Manager/IAM role a populasse)." 2
fi

for bin in pg_dump aws gzip df date; do
    command -v "$bin" >/dev/null 2>&1 || fail "Dependência ausente no PATH: $bin" 2
done

free_gb="$(df --output=avail -BG "$BACKUP_DIR" | tail -1 | tr -dc '0-9')"
if (( free_gb < MIN_FREE_GB )); then
    fail "Espaço livre insuficiente em ${BACKUP_DIR}: ${free_gb}GB (mínimo ${MIN_FREE_GB}GB)." 3
fi

log "INFO" "Início do backup do Ledger. Espaço livre: ${free_gb}GB."

# ------------------------------------------------------------------
# DUMP + COMPRESSÃO (pipe direto, sem arquivo intermediário não compactado)
# ------------------------------------------------------------------
export PGPASSWORD

if pg_dump -h "$DB_HOST" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" --no-password \
    | gzip -c > "$DUMP_PATH"; then
    dump_size="$(du -h "$DUMP_PATH" | cut -f1)"
    log "INFO" "Dump gerado e compactado: ${DUMP_PATH} (${dump_size})."
else
    rm -f "$DUMP_PATH"
    fail "Falha ao gerar o dump (pg_dump/gzip)." 4
fi

# ------------------------------------------------------------------
# UPLOAD PARA S3
# ------------------------------------------------------------------
if aws s3 cp "$DUMP_PATH" "s3://${S3_BUCKET}/${DUMP_BASENAME}" \
    --region "$AWS_REGION" --only-show-errors; then
    log "INFO" "Upload concluído: s3://${S3_BUCKET}/${DUMP_BASENAME}"
else
    fail "Falha ao enviar o dump para s3://${S3_BUCKET}" 5
fi

rm -f "$DUMP_PATH"
log "INFO" "Cópia local removida após upload confirmado."

# ------------------------------------------------------------------
# RETENÇÃO DE 30 DIAS NO S3
# ------------------------------------------------------------------
cutoff_epoch="$(date -d "-${RETENTION_DAYS} days" +%s)"
deleted_count=0
cleanup_failed=0

while IFS= read -r line; do
    [[ -z "$line" ]] && continue
    obj_date="$(awk '{print $1}' <<< "$line")"
    obj_key="$(awk '{print $4}' <<< "$line")"
    [[ -z "$obj_key" ]] && continue

    obj_epoch="$(date -d "$obj_date" +%s 2>/dev/null)" || continue

    if (( obj_epoch < cutoff_epoch )); then
        if aws s3 rm "s3://${S3_BUCKET}/${obj_key}" --region "$AWS_REGION" --only-show-errors; then
            deleted_count=$((deleted_count + 1))
            log "INFO" "Backup expirado removido: ${obj_key}"
        else
            cleanup_failed=1
            log "ERROR" "Falha ao remover backup expirado: ${obj_key}"
        fi
    fi
done < <(aws s3 ls "s3://${S3_BUCKET}/" --region "$AWS_REGION")

log "INFO" "Retenção concluída. ${deleted_count} arquivo(s) removido(s)."

if (( cleanup_failed == 1 )); then
    fail "Backup e upload OK, mas houve falha na limpeza de retenção." 6
fi

log "INFO" "Backup do Ledger finalizado com sucesso."
exit 0
```

---

# Justificativa

O prompt foi estruturado utilizando o framework **R-T-F (Role, Task e Format)** para fornecer contexto suficiente ao modelo e obter uma resposta objetiva e alinhada às boas práticas de produção.

## Role

O papel da IA é definido logo no início do prompt:

> "Você é um Engenheiro de SRE/DevOps Sênior, especialista em administração de PostgreSQL em produção, automação Bash para Linux e integração com serviços AWS."

Essa definição orienta o modelo a gerar um script com padrão de produção (tratamento de erro, logging, idempotência) em vez de um script simplificado de tutorial.

## Task

A tarefa descreve claramente o cenário e todas as restrições do enunciado:

- Dados de conexão do banco (host, porta, banco, usuário).
- Origem da credencial (PGPASSWORD via Secrets Manager/IAM role).
- Fluxo obrigatório: dump → compressão → upload → retenção → log → exit code.
- Uma restrição inferida a partir dos dados do ambiente (80GB livres vs. ~12GB de dump compactado): evitar gerar uma cópia intermediária não compactada em disco, que poderia esgotar o espaço disponível.

Isso reduz ambiguidades e garante que todos os requisitos do enunciado original sejam considerados na geração do script.

## Format

A seção **Format** determina exatamente como a resposta deve ser apresentada:

1. Apenas o script bash completo e comentado por seção.
2. Estrutura de produção obrigatória: `set -euo pipefail`, variáveis `readonly`, funções nomeadas de log/erro.
3. Sem explicações fora do código.

Dessa forma, o output é entregue pronto para uso — copiável diretamente para um arquivo `.sh` e agendável via `cron` — sem a necessidade de retrabalho.

---
---

# Questão 03 - Relatório de Redução de Custos Cloud

## Entrega

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
---
# Questão 04 - Relatório mensal de transações do Ledger
## Entrega

---

## Framework utilizado: T-A-G (Task - Action - Goal)

---

# Prompt

## T (Task)

Sua tarefa é escrever uma query SQL compatível com PostgreSQL para gerar um relatório mensal consolidado de crescimento de transações por categoria.

## A (Action)

Para construir a query, utilize as informações do banco de dados e siga as regras de negócio abaixo:

Schemas das tabelas:
- transactions (id, customer_id, category, amount_cents, status, payment_method, created_at, completed_at)
- customers (id, segment, country, signup_at)

Regras de filtragem, cálculo e formatação:
1. Filtre apenas transações que tenham status = 'completed'.
2. Defina o período para os últimos 6 meses corridos a partir de '2026-04-24' (ou seja, de '2025-10-24' até '2026-04-24'). Use o campo `created_at`.
3. Converta o valor do campo `amount_cents` (que está em centavos) para Reais (com duas casas decimais).
4. Agrupe os dados por mês (no formato YYYY-MM) e por categoria.
5. A saída deve conter as seguintes colunas por linha:
   - Mês (formatado como YYYY-MM)
   - Categoria (category)
   - Quantidade de transações
   - Volume total em Reais (com duas casas decimais)
6. Ordene o resultado final por mês em ordem crescente e, em seguida, por categoria em ordem crescente.

## G (Goal)

O objetivo final é obter uma query SQL otimizada (aproveitando os índices criados na tabela) que retorne exatamente os dados consolidados que a Jennifer precisa para a apresentação de negócios da Goldie, sem a necessidade de tratamento de dados posterior.

---

# Modelo

**Modelo sugerido:** GPT-4o (OpenAI)


---

# Output

```
SELECT 
    TO_CHAR(created_at, 'YYYY-MM') AS mes,
    category,
    COUNT(id) AS quantidade_transacoes,
    ROUND(SUM(amount_cents) / 100.0, 2) AS volume_total_reais
FROM 
    transactions
WHERE 
    status = 'completed'
    AND created_at >= '2026-04-24'::TIMESTAMPTZ - INTERVAL '6 months'
    AND created_at <= '2026-04-24'::TIMESTAMPTZ
GROUP BY 
    TO_CHAR(created_at, 'YYYY-MM'),
    category
ORDER BY 
    mes ASC,
    category ASC;
```

---

# Justificativa

| Elemento | Como aparece no Prompt |
|----------|-------------------------|
| **Task** | Define claramente o papel e a entrega esperada da IA (escrever uma query SQL para PostgreSQL). Isso evita que o modelo tente explicar conceitos de banco de dados ou crie códigos em outras linguagens, focando estritamente na geração do código SQL. |
| **Action** | É o coração do prompt. Aqui passamos o contexto do banco (schemas) e as regras lógicas de transformação passo a passo. É onde instruímos a IA sobre a conversão de centavos (/ 100.0), a formatação de data (TO_CHAR), os filtros de status/período e as métricas de agregação (COUNT e SUM).. |
| **Goal** | Alinha a expectativa de qualidade e o contexto de uso. Ao explicar que o objetivo é gerar dados limpos para uma apresentação executiva (Jennifer para Goldie) e que a query precisa ser otimizada para os índices existentes, a IA prioriza o uso das colunas indexadas no WHERE e entrega um resultado com formatação visualmente limpa (usando ROUND e apelidos amigáveis nas colunas).

---   

# Questão 05 - Modernizar deployment legado

## Entrega

---

## Framework utilizado: B-A-B (Before - After - Bridge)

---

# Prompt

## Before — contexto atual e problema

Você receberá um Kubernetes Deployment manifest legado escrito há três anos.
Ele apresenta os seguintes problemas críticos que violam os padrões atuais de produção da empresa:

- `replicas: 1` — sem alta disponibilidade
- `image: chronos-api:latest` — tag mutável, não rastreável
- Secrets em texto plano no campo `env.value` (DB_PASSWORD, JWT_SECRET)
- Ausência de `resources.requests` e `resources.limits`
- Sem `livenessProbe` nem `readinessProbe`
- Sem `securityContext` — container roda como root
- Sem `podAntiAffinity` — réplicas podem cair no mesmo nó
- Sem `terminationGracePeriodSeconds` explícito
- Sem annotations de monitoramento (prometheus scrape)

Manifest legado a ser modernizado:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      containers:
      - name: api
        image: chronos-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          value: "P@ssw0rd2023!"
        - name: JWT_SECRET
          value: "hvt-jwt-prod-secret"
```

## After — estado final esperado

Produza um Deployment manifest moderno e production-ready que atenda obrigatoriamente:

1. `replicas: 3` com `podAntiAffinity` (preferredDuringSchedulingIgnoredDuringExecution)
2. Imagem versionada com SHA ou tag semântica (ex: `chronos-api:1.4.2`)
3. Secrets via `secretKeyRef` referenciando um Secret externo chamado `chronos-api-secrets`
4. `resources.requests` e `resources.limits` definidos para CPU e memória
5. `livenessProbe` e `readinessProbe` via HTTP no path `/healthz` porta 8080
6. `securityContext` no nível do container: `runAsNonRoot: true`, `runAsUser: 1000`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`
7. `securityContext` no nível do pod: `fsGroup: 1000`
8. `terminationGracePeriodSeconds: 30`
9. `RollingUpdate` strategy com `maxUnavailable: 0` e `maxSurge: 1`
10. Annotations de Prometheus: `prometheus.io/scrape`, `prometheus.io/port`, `prometheus.io/path`

## Bridge — instruções de transformação

Para ir do manifest legado ao manifest moderno, siga este processo passo a passo:

1. Identifique cada problema listado no Before e mapeie para o requisito correspondente do After
2. Mantenha `name: chronos-api` e `namespace: production` intactos
3. Gere o manifest completo em YAML válido e indentado corretamente
4. Após o YAML, entregue uma tabela no formato:

   | Campo modificado | Valor antes | Valor depois | Justificativa |
   |---|---|---|---|

5. Aponte qualquer dependência externa necessária (ex: objeto Secret a ser criado separadamente)
6. Se um valor específico não puder ser determinado sem contexto adicional (ex: limites exatos de CPU), use valores razoáveis para uma API REST e documente o critério adotado

Responda apenas com o YAML e a tabela. Sem texto introdutório.

---

# Modelo

**Modelo utilizado:** Claude Sonnet 4.6 (Anthropic)

---

# Output

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: chronos-api
  template:
    metadata:
      labels:
        app: chronos-api
    spec:
      terminationGracePeriodSeconds: 30
      securityContext:
        fsGroup: 1000
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: chronos-api
              topologyKey: kubernetes.io/hostname
      containers:
      - name: api
        image: chronos-api:1.4.2
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: db-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: jwt-secret
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
```

---

# Justificativa

| Elemento B-A-B | Como aparece no Prompt |
|---|---|
| **Before** | Seção 1 — lista de problemas + manifest legado colado literalmente. Ancora o modelo no estado atual real, elimina ambiguidade sobre "o que está errado" e impede que ele invente um contexto diferente. |
| **After** | Seção 2 — 10 requisitos numerados e explícitos. Define o critério de aceitação do output; o modelo sabe exatamente o que "moderno e production-ready" significa neste contexto, sem depender de suposições. |
| **Bridge** | Seção 3 — 6 passos de transformação + instrução de formato de entrega. Diz como percorrer o caminho do Before ao After: mapeamento 1-a-1 dos problemas, restrições de preservação (nome, namespace), formato de saída (YAML + tabela), tratamento de incertezas (valores razoáveis documentados). |

---
---

# Questão 06 - Módulo Terraform no padrão interno

## Entrega

---

## Framework utilizado: C-A-R-E (Context - Action - Result - Example)

---

# Prompt

```text
# Context

Você é um engenheiro de infraestrutura sênior trabalhando em uma
empresa que adotou o padrão interno de IaC "Strickland". Todo módulo
Terraform novo deve obrigatoriamente seguir esse padrão:

Regras do padrão Strickland:
- Tags obrigatórias em TODOS os recursos: Owner, CostCenter, Environment
- Prefixo "hvt-" em todos os nomes de recursos
- Buckets S3 devem ter: SSE-S3 (mínimo), versioning ativo,
  block public access total e logging habilitado
- variables.tf: toda variável com "description" e "type" obrigatórios

# Action

Crie um módulo Terraform reutilizável para provisionar buckets S3
aderentes ao padrão Strickland. O módulo deve conter:
  - variables.tf  (todas as variáveis de entrada documentadas)
  - main.tf       (recursos S3 e configurações de segurança)
  - outputs.tf    (bucket_id, bucket_arn, bucket_domain_name)
  - examples/basic/main.tf (exemplo completo de uso do módulo)

# Result

Entregue os quatro arquivos prontos para uso, com comentários
inline explicando cada bloco. O módulo deve ser consumível por outros
times sem necessitar de modificações nos arquivos do módulo em si.

# Example

Siga EXATAMENTE o mesmo estilo do módulo de VPC já existente
na empresa como referência:

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags = merge(local.common_tags, {
    Name = "hvt-vpc-${var.environment}"
  })
}
```

---

# Modelo

**Modelo utilizado:** Claude Sonnet 4.6 (Anthropic)

---

# Output

**`variables.tf`**

```hcl
variable "owner" {
  description = "Time ou pessoa responsável pelo recurso (tag Owner)"
  type        = string
}

variable "cost_center" {
  description = "Centro de custo para alocação financeira (tag CostCenter)"
  type        = string
}

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

variable "bucket_suffix" {
  description = "Sufixo único para o nome do bucket (ex: meu-time-dados)"
  type        = string
}

variable "log_bucket_id" {
  description = "ID do bucket S3 de destino dos logs de acesso"
  type        = string
}

variable "log_prefix" {
  description = "Prefixo do path de logs dentro do bucket de logging"
  type        = string
  default     = "s3-access-logs/"
}
```

**`main.tf`**

```hcl
locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}

resource "aws_s3_bucket" "this" {
  bucket = "hvt-${var.bucket_suffix}-${var.environment}"

  tags = merge(local.common_tags, {
    Name = "hvt-${var.bucket_suffix}-${var.environment}"
  })
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_logging" "this" {
  bucket        = aws_s3_bucket.this.id
  target_bucket = var.log_bucket_id
  target_prefix = var.log_prefix
}
```

**`outputs.tf`**

```hcl
output "bucket_id" {
  description = "ID (nome) do bucket S3 criado"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "ARN do bucket S3"
  value       = aws_s3_bucket.this.arn
}

output "bucket_domain_name" {
  description = "Domain name do bucket para uso em políticas e DNS"
  value       = aws_s3_bucket.this.bucket_domain_name
}
```

**`examples/basic/main.tf`**

```hcl
module "s3_dados_financeiros" {
  source = "../../"

  owner         = "time-financeiro"
  cost_center   = "CC-4521"
  environment   = "production"
  bucket_suffix = "financeiro-dados"
  log_bucket_id = "hvt-logs-centrais-production"
}

output "bucket_arn" {
  value = module.s3_dados_financeiros.bucket_arn
}
```

---

# Justificativa

| Elemento C-A-R-E | Como aparece no Prompt |
|---|---|
| **Context** | Seção 1 — define o papel do agente ("engenheiro sênior") e o conjunto completo de regras do padrão Strickland. Ancora o modelo no domínio correto antes de qualquer instrução, evitando que ele produza um módulo genérico sem aderência ao padrão interno. |
| **Action** | Seção 2 — especifica os quatro entregáveis com nomes exatos de arquivo e a responsabilidade de cada um. Verbos imperativos ("Crie", "deve conter") eliminam ambiguidade sobre o que deve ser gerado. |
| **Result** | Seção 3 — define o critério de aceitação: arquivos prontos para uso por outros times sem modificação, com comentários inline. O modelo sabe exatamente quando a tarefa está concluída. |
| **Example** | Seção 4 — o módulo VPC existente é injetado literalmente no prompt. Elimina ambiguidade de estilo: indentação, convenção de nomes de variáveis (`var.owner`), uso de `locals`, padrão de `merge()` e prefixo `hvt-` são demonstrados, não apenas descritos. |

---
---
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
---
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

---
---

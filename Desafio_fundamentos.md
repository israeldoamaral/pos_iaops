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
# Questão 02 – Script de backup do Ledger
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

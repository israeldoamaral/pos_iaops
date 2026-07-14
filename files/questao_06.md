# Questão 06 - Módulo Terraform no padrão interno
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

# Resposta — Questão 06: Módulo Terraform no padrão interno

## Prompt

Context
Strickland, responsável por segurança e compliance da Hill Valley Tech, definiu o padrão para novos módulos Terraform:
- Tags Owner, CostCenter e Environment em todos os recursos que suportam tags.
- Prefixo hvt- nos nomes de recursos AWS.
- Todo bucket S3 deve ter SSE-S3 no mínimo, versioning habilitado, bloqueio público completo e server access logging configurado.
- Toda variável em variables.tf precisa de description e type.
Doc Brown quer um módulo S3 reutilizável por todos os times. As configurações auxiliares do bucket que não suportam tags no provider devem ser explicadas, sem inventar atributos tags inválidos.

Action 
Crie versions.tf, variables.tf, main.tf e outputs.tf para um módulo modules/s3-bucket, mais um exemplo de consumo. Use recursos separados para encryption, versioning, public access block, ownership e logging, com Terraform >=1.5 e provider hashicorp/aws na linha 6.x. Construa o nome `hvt-<name>-<environment>-<account_id>`, valide o slug, ambiente e tags não vazias. Tags adicionais não podem sobrescrever as obrigatórias. Não configure credenciais nem provider dentro do módulo. Não permita destruição forçada de bucket com objetos; bloqueie HTTP por bucket policy. Use BucketOwnerEnforced, sem ACLs. Receba um bucket central de logs já existente, na mesma conta e região, com nome hvt-. Ele deve ser diferente do bucket criado. Documente que o destino precisa de SSE-S3 e de policy para logging.s3.amazonaws.com, limitada ao prefixo, SourceArn e SourceAccount. Forneça exemplo dessa declaração para incorporar à policy existente, sem sobrescrevê-la. Não crie um bucket de logs por instância nem habilite logging para ele mesmo. Explique o conflito entre a regra literal de logging em todo bucket e a necessidade de um destino terminal sem ciclo; registre que a governança desse destino preexistente precisa ser definida por Strickland, sem afirmar que já existe uma exceção aprovada.

Result 
Entregue arquivos HCL completos identificados pelo caminho, um exemplo de uso e instruções breves de validação e pré-requisitos. O módulo deve criar um bucket privado, criptografado, versionado, com tags e destino de logs definidos. A entrega efetiva dos logs depende da policy do destino e deve ser verificada após implantação. Não afirme ter aplicado Terraform ou validado recursos AWS reais.

Example 
Siga o estilo deste módulo interno de VPC: locals.common_tags, merge e nomes hvt-. Adapte o padrão ao S3, sem copiar o recurso VPC:

```hcl
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

## Modelo

GPT-6, da OpenAI.

## Output

Resposta produzida nesta sessão a partir do prompt acima.

`modules/s3-bucket/versions.tf`:

```hcl
terraform {
  required_version = ">= 1.5.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

`modules/s3-bucket/variables.tf`:

```hcl
variable "name" {
  description = "Identificador do bucket, sem prefixo; 1 a 25 caracteres minúsculos, números ou hífens."
  type        = string
  nullable    = false

  validation {
    condition     = can(regex("^[a-z0-9]([a-z0-9-]{0,23}[a-z0-9])?$", var.name))
    error_message = "Use um slug de 1 a 25 caracteres, começando e terminando com letra ou número."
  }
}

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)."
  type        = string
  nullable    = false

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Ambiente deve ser dev, staging ou production."
  }
}

variable "owner" {
  description = "Time responsável pelo recurso, usado na tag Owner."
  type        = string
  nullable    = false

  validation {
    condition     = length(trimspace(var.owner)) > 0
    error_message = "Owner não pode ser vazio."
  }
}

variable "cost_center" {
  description = "Centro de custo usado na tag CostCenter."
  type        = string
  nullable    = false

  validation {
    condition     = length(trimspace(var.cost_center)) > 0
    error_message = "CostCenter não pode ser vazio."
  }
}

variable "log_bucket_name" {
  description = "Bucket central de logs preexistente, na mesma conta e região, com policy de entrega configurada."
  type        = string
  nullable    = false

  validation {
    condition = (
      length(var.log_bucket_name) <= 63 &&
      can(regex("^hvt-[a-z0-9]([a-z0-9-]*[a-z0-9])?$", var.log_bucket_name))
    )
    error_message = "O destino deve ter nome S3 válido sem pontos, prefixo hvt- e até 63 caracteres."
  }
}

variable "extra_tags" {
  description = "Tags adicionais; Owner, CostCenter, Environment e Name são controladas pelo módulo."
  type        = map(string)
  default     = {}
  nullable    = false
}
```

`modules/s3-bucket/main.tf`:

```hcl
data "aws_caller_identity" "current" {}
data "aws_partition" "current" {}

locals {
  bucket_name = "hvt-${var.name}-${var.environment}-${data.aws_caller_identity.current.account_id}"
  log_prefix  = "s3-access/${local.bucket_name}/"
  common_tags = {
    Owner       = trimspace(var.owner)
    CostCenter  = trimspace(var.cost_center)
    Environment = var.environment
  }
}

resource "aws_s3_bucket" "this" {
  bucket        = local.bucket_name
  force_destroy = false

  tags = merge(var.extra_tags, local.common_tags, {
    Name = local.bucket_name
  })

  lifecycle {
    precondition {
      condition     = var.log_bucket_name != local.bucket_name
      error_message = "Origem e destino de logs devem ser buckets diferentes."
    }
  }
}

resource "aws_s3_bucket_ownership_controls" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_logging" "this" {
  bucket        = aws_s3_bucket.this.id
  target_bucket = var.log_bucket_name
  target_prefix = local.log_prefix
}

data "aws_iam_policy_document" "tls" {
  statement {
    sid     = "DenyInsecureTransport"
    effect  = "Deny"
    actions = ["s3:*"]
    resources = [
      aws_s3_bucket.this.arn,
      "${aws_s3_bucket.this.arn}/*"
    ]
    principals {
      type        = "*"
      identifiers = ["*"]
    }
    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }
}

resource "aws_s3_bucket_policy" "this" {
  bucket = aws_s3_bucket.this.id
  policy = data.aws_iam_policy_document.tls.json
}
```

`modules/s3-bucket/outputs.tf`:

```hcl
output "bucket_name" {
  description = "Nome do bucket criado."
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "ARN do bucket criado."
  value       = aws_s3_bucket.this.arn
}

output "log_prefix" {
  description = "Prefixo exclusivo de entrega no bucket central de logs."
  value       = local.log_prefix
}

output "log_delivery_statement" {
  description = "Declaração a mesclar à policy do destino pela equipe responsável por esse bucket."
  value = {
    Sid       = "AllowS3Logs${replace(local.bucket_name, "-", "")}"
    Effect    = "Allow"
    Principal = { Service = "logging.s3.amazonaws.com" }
    Action    = "s3:PutObject"
    Resource  = "arn:${data.aws_partition.current.partition}:s3:::${var.log_bucket_name}/${local.log_prefix}*"
    Condition = {
      ArnEquals = {
        "aws:SourceArn" = "arn:${data.aws_partition.current.partition}:s3:::${local.bucket_name}"
      }
      StringEquals = {
        "aws:SourceAccount" = data.aws_caller_identity.current.account_id
      }
    }
  }
}
```

`examples/basic/main.tf` — supondo o exemplo dentro do repositório que contém `modules/`:

```hcl
terraform {
  required_version = ">= 1.5.0, < 2.0.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

module "ledger_exports" {
  source          = "../../modules/s3-bucket"
  name            = "ledger-exports"
  environment     = "production"
  owner           = "data-platform"
  cost_center     = "CC-LEDGER"
  log_bucket_name = "hvt-central-logs-production-123456789012"
  extra_tags = {
    Application = "ledger"
  }
}

output "bucket_name" {
  value = module.ledger_exports.bucket_name
}

output "log_delivery_statement" {
  value = module.ledger_exports.log_delivery_statement
}
```

O ID `123456789012` é ilustrativo: substituir o destino por um bucket real da mesma conta e região do provider. O nome da origem usa a conta autenticada e tem até 53 caracteres para os ambientes aceitos. Ele precisa continuar globalmente disponível; se necessário, escolher outro `name`. Usar nomes diferentes para implantações em regiões distintas na mesma conta, pois o nome não inclui a região.

O bucket central precisa estar previamente configurado com SSE-S3, bloqueio público e permissão de entrega. A declaração gerada em `log_delivery_statement` deve ser **mesclada** à policy central pelo seu único responsável, preservando declarações existentes; não criar uma policy concorrente a cada chamada do módulo. Os nomes e ARNs são determinísticos, permitindo provisionar a permissão antes de ativar a origem. `logging.s3.amazonaws.com` receberá apenas `s3:PutObject` no prefixo definido, restrito à origem e conta. O destino não pode usar Object Lock ou Requester Pays para essa entrega.

Existe uma tensão no padrão: a AWS recomenda que o destino terminal de server access logs não tenha esse mesmo logging habilitado, para evitar cadeias ou ciclos. Este módulo cria somente o bucket de dados, com logging configurado; **não certifica a conformidade do destino preexistente**. Strickland precisa definir a governança desse destino e eventual exceção/controle alternativo. Nenhuma exceção aprovada foi presumida.

`Owner`, `CostCenter` e `Environment` são aplicadas ao recurso bucket; os recursos auxiliares representam configurações desse mesmo bucket e não expõem `tags` ou nomes AWS independentes no provider. `this` é um identificador local do Terraform, não o nome de um recurso AWS. A ordem de `merge` impede tags extras de substituir as obrigatórias. `force_destroy=false` impede apagar automaticamente conteúdo ao destruir, mas **não impede a exclusão de um bucket vazio**.

Salvar os blocos nos caminhos indicados e, no exemplo, executar `terraform fmt -check -recursive`, `terraform init` e `terraform validate`; revisar `terraform plan` em conta de teste. Versionar o lockfile do projeto consumidor. Após implantação, verificar encryption, versioning, public access block, policy e configuração de logging; gerar acessos e confirmar a chegada de logs no prefixo. A entrega não é imediata. Não foram executados `init`, `plan` ou `apply` nesta resposta.

## Justificativa

**Context:** apresenta os controles de Strickland e a necessidade de reutilização pelos times.  
**Action:** especifica recursos, arquivos, validações, restrições de tags e integração com o destino de logs.  
**Result:** define o módulo, exemplo de consumo e critérios de conformidade verificáveis.  
**Example:** fornece o módulo VPC como referência concreta para `locals.common_tags`, `merge` e nomes `hvt-`.

## Referências técnicas

A separação entre bucket e configurações segue os recursos [aws_s3_bucket](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket), [encryption](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_server_side_encryption_configuration) e [logging](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/s3_bucket_logging). Os pré-requisitos do destino e as permissões de entrega estão em [server access logging do Amazon S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/enable-server-access-logging.html).

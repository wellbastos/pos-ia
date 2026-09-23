# Questão 06 - Módulo Terraform no padrão interno
Strickland, que fecha a ponta de segurança e compliance, publicou o padrão interno de IaC que todo módulo Terraform novo precisa seguir:

Tags obrigatórias em todo recurso: Owner, CostCenter, Environment.
Prefixo hvt- nos nomes de recursos.
Todo bucket S3 com: encryption habilitada (SSE-S3 mínimo), versioning ativo, block public access total, logging configurado.
Variáveis de entrada em variables.tf com description e type obrigatórios.
Doc Brown pediu um módulo Terraform reutilizável pra criar buckets S3 aderentes a esse padrão. O módulo vai ser consumido por todos os times da empresa, então precisa vir com exemplo de uso. Como referência de estilo, o módulo de VPC que já existe na empresa:

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
Tarefa. Aplicando o framework C-A-R-E, escrever o prompt de IA que produza o módulo Terraform S3 aderente ao padrão, no mesmo estilo do exemplo.

Entregue. Prompt, modelo, output e justificativa mostrando como Context, Action, Result e Example aparecem no prompt.
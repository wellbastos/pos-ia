# Questão 03 - Relatório de redução de custos cloud

Goldie apresentou a meta do próximo trimestre à diretoria: 15% de redução no custo cloud até o fim do período, sem degradar SLA. Doc Brown pegou o breakdown de custos AWS do último mês e repassou a análise inicial para o time. O CSV está abaixo.

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

O relatório que vai voltar pra Goldie precisa trazer as oportunidades de economia priorizadas por impacto, quanto cada uma representa em percentual da conta total, o esforço de implementação (baixo, médio, alto) e os riscos ou pré-requisitos envolvidos em cada uma.

Tarefa. Aplicando o framework T-A-G, escrever o prompt de IA que, a partir do CSV acima, produza esse relatório alinhado à meta de 15%.

Entregue. Prompt, modelo, output e justificativa mostrando como Task, Action e Goal aparecem no prompt.
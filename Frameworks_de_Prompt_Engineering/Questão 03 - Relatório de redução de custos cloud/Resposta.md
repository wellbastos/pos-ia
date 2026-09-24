# Resposta — Questão 03: Relatório de redução de custos cloud

## Prompt

Task

Produza, em português, um relatório executivo de oportunidades de redução de custos AWS para Goldie Wilson, CEO da Hill Valley Tech, usando somente o CSV abaixo. A meta é reduzir 15% da conta mensal até o fim do próximo trimestre, sem degradar SLA. Não use ferramentas ou acesse arquivos.

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

Action
1. Some a conta mensal, calcule 15% em USD/mês e o teto mensal desejado.
2. Proponha um cenário de planejamento com premissas explícitas de redução por serviço, o CSV não permite comprovar economias ou dimensionamento. Não interprete utilização média baixa como desperdício eliminável.
3. Apresente oportunidades em tabela, em ordem decrescente de economia mensal estimada. Inclua serviço, ação, premissa percentual sobre o serviço, USD/mês, percentual da conta total, esforço baixo/médio/alto e riscos ou pré-requisitos. Diferencie percentual sobre o serviço de percentual sobre a conta, não use preços externos ou invente medições.
4. Evite dupla contagem, inclusive entre EKS e EC2: declare que o cenário depende de rubricas sem sobreposição e valide a atribuição dos custos antes de somar. Preserve RDS Multi-AZ, redundância de NAT e observabilidade crítica. Não proponha cancelar compromissos EC2 já contratados como economia imediata. Não sugira compra de compromissos para carga variável antes de validar a linha de base.
5. Some o cenário uma única vez, mostre quanto cobre da meta e a margem ou lacuna. Inclua plano de 30/60/90 dias com responsáveis por função, validação de picos, latência, erros e rollback, não invente valores do SLA.

Goal
Dar à CEO uma decisão verificável sobre como atingir pelo menos 15% de redução mensal ao final do trimestre, distinguindo economia potencial de realizada e condicionando a adoção à preservação do SLA. O alvo é o custo mensal de saída do trimestre, não 15% de redução acumulada em todo o período. Entregue resumo, tabela, consolidação numérica e plano de execução, com no máximo 1.000 palavras e sem garantir economias ainda não medidas.

## Modelo

GPT-6 da OpenAI.

## Output

Resposta produzida nesta sessão a partir do prompt acima.

### Relatório executivo — Redução de custos AWS

A conta mensal informada soma **US$ 41.800,00**. Reduzir 15% exige **US$ 6.270,00/mês** de economia, levando o custo mensal de saída do trimestre a **US$ 35.530,00**. O cenário proposto estima **US$ 6.985,00/mês (16,71%)**, condicionado à validação técnica e financeira.Nenhuma economia foi medida ou realizada.

As taxas abaixo são hipóteses de planejamento sobre cada rubrica, não resultados inferidos da utilização média. Médias de 30% a 72% não informam picos, memória, latência, armazenamento mínimo ou capacidade necessária para cumprir o SLA. Valores ausentes no CSV não significam utilização zero.

### Oportunidades priorizadas

| Prioridade / serviço | Ação e hipótese sobre a rubrica | Economia estimada USD/mês | % da conta total | Esforço | Riscos e pré-requisitos |
|---|---|---:|---:|---|---|
| 1 — EC2 on-demand | Ajustar tamanhos e escalabilidade após análise de picos, hipótese de redução de 25%. | 2.050,00 | 4,90% | Médio | Validar CPU/memória, p99 e margem de falha. Compromissos só para parcela estável comprovada, sem reduzir redundância. |
| 2 — EKS | Otimizar capacidade ociosa de workers e requests, hipótese de 20% da rubrica. | 1.340,00 | 3,21% | Alto | Separar custo de control plane, workers e EC2 já contabilizada. Não presumir consolidação segura dos três clusters. Validar PDB, scheduling e capacidade de pico. |
| 3 — CloudWatch Logs | Reduzir ingestão redundante/verbosa e ajustar retenção por classe de log, hipótese de 35%. | 980,00 | 2,34% | Médio | Separar ingestão e armazenamento, reduzir retenção não reduz automaticamente ingestão. Preservar auditoria e diagnóstico, validar requisitos antes de alterar 90 dias. |
| 4 — S3 Standard | Aplicar Lifecycle a objetos comprovadamente frios, hipótese líquida de 20%. | 620,00 | 1,48% | Médio | Considerar acesso, latência de recuperação, permanência mínima e custos de transição/recuperação. Não migrar objetos críticos sem teste. |
| 5 — ElastiCache Redis | Rightsizing com base em memória, evictions e latência, hipótese de 20%. | 420,00 | 1,00% | Médio | Uso médio de 40% não prova excesso de memória. Preservar réplica/failover e evitar queda do cache hit rate que sobrecarregue RDS. |
| 6 — RDS PostgreSQL | Revisar capacidade de instância/IOPS superdimensionada, se confirmada, hipótese de 5%. | 410,00 | 0,98% | Alto | Manter Multi-AZ. Medir memória, conexões, waits e I/O, considerar janela de mudança, failover e rollback. Não presumir redução simples de armazenamento alocado. |
| 7 — Data Transfer Out | Reduzir transferências inter-regionais evitáveis e melhorar localidade, hipótese de 20%. | 380,00 | 0,91% | Alto | Validar fluxos e custos de mudança. Preservar disaster recovery, replicação necessária e requisitos de residência. |
| 8 — EBS gp3 | Remover volumes realmente órfãos e ajustar IOPS/throughput excedentes, hipótese de 20%. | 320,00 | 0,77% | Médio | Inventário e responsáveis precisam confirmar desuso. Não apagar volumes apenas pelo uso médio de 68%, validar backups e desempenho. |
| 9 — NAT Gateway | Avaliar rotas e endpoints para serviços AWS, hipótese líquida de 20%. | 240,00 | 0,57% | Médio | Manter os três gateways enquanto necessários à redundância. Contabilizar custo dos endpoints e evitar transferências entre AZs introduzidas pela mudança. |
| 10 — CloudWatch Metrics | Reduzir séries customizadas redundantes e cardinalidade desnecessária, hipótese de 15%. | 135,00 | 0,32% | Baixo | Inventariar métricas pagas, alarmes e dashboards. Manter SLIs e cobertura operacional. |
| 11 — Lambda | Ajustar memória/duração após benchmark, hipótese de 10%. | 90,00 | 0,22% | Médio | Uso de 30% não justifica reduzir memória: isso pode elevar duração, custo e latência. Validar cold starts e erros. |

**EC2 reservada:** economia imediata considerada igual a zero. O contrato de um ano não desaparece com menor utilização, avaliar cobertura e renovação separadamente, sem registrar compromisso já contratado como economia nova.

### Consolidação e condições da meta

- Total potencial: **US$ 6.985,00/mês**, ou **16,71%** da base. Soma calculada em dólares, percentuais da tabela foram arredondados individualmente.
- Conta projetada: **US$ 34.815,00/mês**. Margem sobre a meta: **US$ 715,00/mês**, equivalente a 1,71 ponto percentual da base, cobertura de **111,40%** da economia requerida.
- Cada rubrica foi contada uma vez. A soma só é válida se os custos forem mutuamente exclusivos e os ganhos não incidirem sobre o mesmo gasto. Validar atribuição de EKS/EC2 no Cost and Usage Report antes de aprovar o cenário, verificar também interações entre transferências e NAT.
- Se a oportunidade EKS não for elegível, as demais somam **US$ 5.645,00/mês (13,50%)**, faltando **US$ 625,00/mês**. Não declarar a meta atingida com sobreposição ou ganhos não comprovados.
- São economias recorrentes ao fim do trimestre. Não multiplicar automaticamente por três: execução gradual, esforço de migração, custos de recuperação e variação de demanda afetam o resultado acumulado.

### Plano de 30/60/90 dias

| Prazo | Responsáveis propostos | Entregas e decisão de avanço |
|---|---|---|
| Até 30 dias | FinOps + donos dos serviços + SRE | Conciliar conta/atribuição e picos, obter SLO/SLA reais. Validar hipóteses, inventariar recursos e testar métricas, logs e EBS em escopo pequeno. Registrar custo/latência/erros antes da mudança. |
| Até 60 dias | Plataforma + dados + SRE | Executar pilotos de EC2/EKS, Lifecycle e Redis. Reavaliar rede/RDS com especialistas. Expandir apenas com economia líquida demonstrada e p99/erros dentro dos limites vigentes, incluindo carga de pico e failover. |
| Até 90 dias | FinOps + responsáveis técnicos, decisão de Goldie | Consolidar otimizações aprovadas, comparar conta efetiva com baseline ajustado à demanda e comprovar custo mensal de saída <= US$ 35.530,00. Se houver lacuna, repriorizar ganhos medidos, não sacrificar SLA para fechar o número. |

Cada mudança deve ter responsável, janela, configuração anterior e critério de rollback baseado no SLO vigente. Se latência, erros, disponibilidade ou margem de failover piorarem, interromper e restaurar a configuração anterior. Não inventar valores de SLA ausentes. Reportar semanalmente economia **estimada, validada e realizada** separadamente, além de custos de implementação e custo por unidade de carga para distinguir eficiência de queda de demanda.

## Justificativa

**Task:** define o relatório para Goldie, fornece o CSV e a meta trimestral de redução sem degradar SLA.  
**Action:** orienta cálculos, priorização, esforço, riscos e validação de premissas e sobreposições.  
**Goal:** exige uma decisão verificável sobre redução mensal de pelo menos 15%, separando potencial e resultado realizado.
# Questão 07 - Runbook para alerta recorrente
Toda semana, em média 4 vezes, o Beacon dispara o mesmo alerta no canal de plantão: [CRITICAL] High memory usage on Chronos API pods (>85% for 10min). Quem assume o plantão gasta de 30 a 40 minutos até resolver, e o tempo varia muito porque não existe procedimento documentado. Lorraine quer um runbook que qualquer plantonista consiga seguir de ponta a ponta sem depender de quem conhece o sistema. O ambiente que o runbook precisa considerar:

Chronos roda no EKS, namespace production, 6 réplicas com HPA configurado (min 4, max 12, CPU target 70%).
Deploy via Argo CD a partir do repositório hvt/chronos-api.
Dependências diretas: Ledger (PostgreSQL) e Reactor (filas SQS).
Observabilidade: métricas expostas em /metrics, logs centralizados no Beacon, dashboards em Grafana.
Ferramentas disponíveis para o plantão: kubectl, aws cli, argocd cli.
Canal de plantão: #oncall-chronos no Slack.
Time sênior de escalação: @chronos-core (SLA de resposta: 15 minutos em horário comercial, 30 fora).
O runbook precisa cobrir os passos iniciais de diagnóstico (com os comandos específicos a rodar), a verificação esperada ao final de cada passo, os critérios objetivos para escalar para o time sênior e o critério para encerrar o incidente.

Tarefa. Aplicando o framework R-I-S-E, escrever o prompt de IA que produza esse runbook procedural completo.

Entregue. Prompt, modelo, output e justificativa mostrando como Role, Input, Steps e Expectation aparecem no prompt.

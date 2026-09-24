# Resposta — Questão 07: Runbook para alerta recorrente

## Prompt


Role
Atue como uma pessoa SRE sênior que escreve procedimentos claros para plantonistas com pouca familiaridade com o Chronos.

Input
O Beacon dispara cerca de quatro vezes por semana: [CRITICAL] High memory usage on Chronos API pods (>85% for 10min). A resolução hoje leva 30 a 40 minutos, sem procedimento documentado. Ambiente: EKS, namespace production, seis réplicas de Chronos; HPA min=4, max=12 e alvo de CPU=70%; deploy via Argo CD a partir de hvt/chronos-api; dependências Ledger (PostgreSQL) e Reactor (SQS). Métricas em /metrics, logs no Beacon, dashboards Grafana. Ferramentas: kubectl, aws cli e argocd cli. Canal: #oncall-chronos; escalação: @chronos-core, com resposta em até 15 minutos em horário comercial ou 30 minutos fora dele. Não foram fornecidos região, cluster, nome da Application Argo CD, identificador do RDS, URL SQS, nomes de métricas ou limites do SLA.

Steps
Organize um runbook numerado de ponta a ponta: assumir e registrar, conferir contexto, identificar pods/limites/restarts, correlacionar métricas e logs, verificar rollout e dependências, escolher mitigação, validar recuperação e escalar/encerrar. Para cada etapa, dê comandos específicos, evidência esperada e próxima ação se houver anomalia. Comece por inspeções de leitura. Mostre como descobrir identificadores, sem inventá-los, e marque variáveis que precisam ser preenchidas. Explique que kubectl top é uma amostra e não prova a persistência do alerta, que percentual depende da regra do Beacon, e que HPA de CPU não escala diretamente por memória. Considere OOMKilled, pressão no nó, vazamento, carga e dependências. Não prescreva aumento cego de réplicas/limits ou restart em massa. Mitigações devem respeitar capacidade, HPA, PDB, drenagem e GitOps; prefira commit/reversão e sync, sem permitir que Argo CD desfaça a ação. Forneça critérios objetivos de escalação, timeboxes e pacote de evidências. Defina critérios mensuráveis para encerrar, distinguindo limites propostos para este runbook de SLOs oficiais não informados.

Expectation
Entregue um runbook em português, executável por um plantonista, com diagnóstico inicial em até dez minutos, pontos de decisão claros e validação antes do encerramento. Não prometa resolução nesse prazo. Inclua modelo de comunicação no Slack, sem enviar mensagens, e ações de prevenção após o incidente. Não alegue que comandos foram executados.

## Modelo

GPT-6, da OpenAI.

## Output

Resposta produzida nesta sessão a partir do prompt acima.

### Runbook — Memória alta nos pods do Chronos

**Gatilho:** Beacon informa memória acima de 85% por dez minutos. **Responsável inicial:** plantonista de `#oncall-chronos`. Objetivo: produzir diagnóstico inicial em dez minutos e selecionar uma mitigação segura. Os limiares de recuperação e timeboxes abaixo são propostas do procedimento, sujeitos à validação da SRE; não representam SLOs fornecidos pela empresa.

#### 1. Assumir e confirmar o ambiente — minutos 0–2

Registrar horário UTC, link do alerta, pods afetados e sintomas percebidos por clientes. Publicar no canal, manualmente:

> Assumi o alerta de memória do Chronos às [UTC]. Impacto conhecido: [evidências ou em apuração]. Pods afetados: [lista]. Próxima atualização em dez minutos. Incidente: [link].

```bash
date -u
kubectl config current-context
kubectl cluster-info
kubectl -n production get deployments
aws sts get-caller-identity
argocd app list
```

**Esperado:** contexto, conta AWS e destino Argo CD correspondem à produção correta. Identificar o Deployment do Chronos e a Application ligada a `hvt/chronos-api`. Se houver contexto errado ou acesso negado, corrigir pelo procedimento de acesso e acionar suporte; não testar mutações para descobrir o ambiente.

Nos comandos seguintes, preencher os valores entre `<...>` e executar no mesmo shell. `chronos-api` e `app=chronos-api` são exemplos a confirmar pelo Deployment:

```bash
NS=production
DEPLOY='<deployment-confirmado>'
APP='<application-argo-confirmada>'
kubectl -n "$NS" get deployment "$DEPLOY" -o yaml
# Copiar o seletor real de spec.selector; exemplo: app=chronos-api.
SELECTOR='<seletor-confirmado>'
```

O manifesto pode conter configuração sensível; guardar apenas os trechos necessários e sem credenciais no registro do incidente.

#### 2. Medir alcance e identificar OOM — minutos 2–5

```bash
kubectl -n "$NS" get pods -l "$SELECTOR" -o wide
kubectl -n "$NS" top pods -l "$SELECTOR" --containers
kubectl -n "$NS" get pods -l "$SELECTOR" \
  -o custom-columns='POD:.metadata.name,RESTARTS:.status.containerStatuses[*].restartCount,LAST_REASON:.status.containerStatuses[*].lastState.terminated.reason,LIMITS:.spec.containers[*].resources.limits.memory'
kubectl -n "$NS" get events --sort-by=.lastTimestamp
kubectl -n "$NS" get hpa \
  -o custom-columns='NAME:.metadata.name,TARGET:.spec.scaleTargetRef.name,MIN:.spec.minReplicas,MAX:.spec.maxReplicas,CURRENT:.status.currentReplicas'
```

Selecionar um pod afetado e seu container:

```bash
POD='<pod-afetado>'
CONTAINER='<container-da-api>'
NODE=$(kubectl -n "$NS" get pod "$POD" -o jsonpath='{.spec.nodeName}')
kubectl -n "$NS" describe pod "$POD"
kubectl describe node "$NODE"
kubectl top node "$NODE"
```

**Esperado:** identificar se o problema atinge um pod, uma revisão ou todo o Deployment; registrar requests/limits, consumo, reinícios, `OOMKilled`, evictions e `MemoryPressure`. No Beacon/Grafana, conferir a expressão do alerta: 85% de limit, de request ou outro denominador? Usar a mesma definição para acompanhar a recuperação. `kubectl top` é uma amostra; se Metrics Server estiver indisponível, usar o histórico do Beacon e abrir a falha de coleta, sem concluir que a memória está normal.

**Decisão:** OOM recorrente, pressão no nó ou impacto em clientes exige escalação imediata enquanto o diagnóstico continua. Se ainda houver margem, seguir para correlação. O HPA com alvo de CPU em 70% não reage diretamente ao uso de memória.

#### 3. Correlacionar com deploy, carga e logs — minutos 5–8

```bash
kubectl -n "$NS" rollout history deployment/"$DEPLOY"
kubectl -n "$NS" get rs -l "$SELECTOR" -o wide
kubectl -n "$NS" logs -l "$SELECTOR" --all-containers=true \
  --since=30m --tail=300 --prefix=true --max-log-requests=12
argocd app get "$APP"
argocd app history "$APP"
argocd app diff "$APP"
```

Se o container tiver reiniciado e existir instância anterior:

```bash
kubectl -n "$NS" logs "$POD" -c "$CONTAINER" --previous --tail=300
```

No Beacon/Grafana, comparar memória por pod/revisão, taxa de requisições, p95/p99, erros e reinícios nos últimos 60 minutos e no último período equivalente de carga saudável. Correlacionar filas internas, timeouts do Ledger e falhas de publicação no Reactor. Inspecionar os gráficos existentes das métricas de `/metrics`; não presumir nomes de séries que não foram fornecidos.

**Esperado:** uma hipótese sustentada por evidências. Crescimento contínuo sem queda após redução de carga sugere retenção/vazamento, mas não o prova. Aumento alinhado à carga pode indicar concorrência/cache. Início após rollout sugere regressão. Pod isolado pode indicar distribuição desigual de tráfego. Registrar também evidências que contradigam a hipótese.

#### 4. Conferir dependências e margem de capacidade — minutos 8–10

Confirmar região pela configuração do ambiente; não assumir `us-east-1` neste cenário. Consultar a configuração não secreta da aplicação para mapear o Ledger e a fila correta, cruzando com o inventário:

```bash
AWS_REGION='<regiao-confirmada>'
aws rds describe-db-instances --region "$AWS_REGION" \
  --query 'DBInstances[].{Id:DBInstanceIdentifier,Endpoint:Endpoint.Address,Status:DBInstanceStatus,Class:DBInstanceClass}'
aws sqs list-queues --region "$AWS_REGION"
DB_ID='<identificador-confirmado-do-ledger>'
QUEUE_URL='<url-confirmada-da-fila-do-reactor>'
START='<inicio-UTC-ISO8601>'
END='<fim-UTC-ISO8601>'
aws cloudwatch get-metric-statistics --region "$AWS_REGION" \
  --namespace AWS/RDS --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value="$DB_ID" \
  --start-time "$START" --end-time "$END" --period 60 \
  --statistics Average Maximum
aws sqs get-queue-attributes --region "$AWS_REGION" --queue-url "$QUEUE_URL" \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible
kubectl -n "$NS" get pdb
HPA='<hpa-que-aponta-para-o-deployment>'
kubectl -n "$NS" describe hpa "$HPA"
```

**Esperado:** estabelecer se há saturação de conexões, lentidão do Ledger, fila crescendo e espaço real para mais pods/memória. Consultar CPU, memória livre e latências do RDS nos dashboards existentes, além de idade da mensagem mais antiga/lag do Reactor. Os atributos SQS acima são contagens aproximadas, não medem idade. Ausência de datapoints exige verificar região, dimensão e janela; não significa zero conexões.

**Decisão:** se dependências estiverem saturadas, não ampliar concorrência do Chronos automaticamente; mais pods podem abrir mais conexões e agravar o incidente. Coordenar com os responsáveis por Ledger/Reactor.

#### 5. Escolher uma mitigação e acompanhar

| Evidência | Ação condicionada | Verificação esperada |
|---|---|---|
| Regressão ligada a um deploy | Reverter o commit para a release conhecida como saudável, após verificar compatibilidade de schema/API, e sincronizar pelo Argo CD. | Nova revisão saudável, memória estabilizando, p99/erros melhorando. |
| Memória acompanha a carga; nós e dependências têm margem | Ajustar `minReplicas` do HPA no Git como medida temporária, dentro do máximo 12, e sincronizar. Calcular impacto dos pools no Ledger. | Menor carga/memória por pod, sem aumentar erros ou conexões além da margem aprovada. |
| Limit comprovadamente abaixo do uso saudável | Ajustar requests/limits no Git com capacidade disponível nos nós e teste/canário. | Ausência de novos OOM e scheduling viável; uso absoluto também estabiliza. |
| Um pod deteriorado, demais réplicas saudáveis | Após capturar evidências, avaliar substituição unitária controlada pelo procedimento de eviction, respeitando PDB e drenagem. | Reposição Ready antes de considerar qualquer outra substituição; sem perda de capacidade útil. |
| Ledger ou Reactor pressionado | Reduzir concorrência/entrada apenas por controles existentes e conhecidos, junto ao time responsável. | Recuperação da dependência e do Chronos, sem perda silenciosa de transações. |

Não aumentar limits apenas para diminuir matematicamente o percentual do alerta. Não reiniciar todos os pods. A exclusão direta de pods não é protegida pelo PDB; se não houver procedimento de eviction validado, escalar em vez de improvisar.

Depois que a alteração revisada estiver no Git, para uma Application simples cuja revisão de destino já aponta para o commit aprovado:

```bash
argocd app diff "$APP"
argocd app sync "$APP"
argocd app wait "$APP" --sync --health --timeout 600
kubectl -n "$NS" rollout status deployment/"$DEPLOY" --timeout=600s
```

Conferir que o diff contém somente a mitigação combinada. Em aplicações multi-source/ApplicationSet, seguir o fluxo específico da plataforma. Não usar `kubectl scale` contra HPA nem alteração ao vivo que o Argo CD possa desfazer. Se uma intervenção emergencial fora do Git for indispensável, coordenar formalmente a reconciliação e registrar depois o estado final no repositório. Se erros, p99, OOM ou backlog piorarem na primeira janela de cinco minutos, interromper novas mudanças e reverter a mitigação compatível pelo Git.

#### 6. Escalar com critérios e evidências

Acionar `@chronos-core` imediatamente se houver impacto confirmado em clientes/SLO, OOM em dois ou mais pods em cinco minutos, pressão/evictions nos nós, memória acima de 95% por cinco minutos, ou dependência saturada. Acionar também se não houver hipótese acionável ao completar dez minutos, se a primeira mitigação falhar, ou se for necessário alterar limites de capacidade sem segurança técnica. Os limiares adicionais são propostas do runbook.

Enviar horário UTC, ambiente, duração do alerta, pods/revisões afetados, uso absoluto e denominador do percentual, reinícios, request rate, latência, erros, situação do HPA/PDB, estado das dependências, ações e efeitos. Usar links e trechos sem credenciais ou dados de clientes.

> @chronos-core: escalação do incidente [link], iniciada às [UTC]. Critério: [gatilho]. Impacto: [evidências]. Hipótese: [evidências e lacunas]. Ação executada: [ação/resultado]. Precisamos de [decisão]. Próxima atualização às [UTC].

Registrar a hora da escalação. Se não houver resposta em 15 minutos no horário comercial ou 30 fora, acionar a liderança SRE/Lorraine e a rota alternativa da escala vigente. Não aguardar esse prazo para mobilizar a liderança se o impacto estiver crescendo. Atualizar `#oncall-chronos` a cada dez minutos, mesmo sem mudança de estado.

#### 7. Validar recuperação e encerrar

Manter observação por pelo menos 15 minutos sob carga representativa, exigindo todos os critérios:

- Memória abaixo de 80% pelo mesmo denominador do alerta, sem tendência crescente ou novos `OOMKilled`/evictions. Se limits mudaram, comparar também bytes consumidos e carga.
- Todos os pods desejados Ready, sem aumento de reinícios, rollout e Argo CD estáveis. HPA dentro dos limites e capacidade suficiente.
- p99 e erros dentro do SLO vigente; se ele não estiver disponível, validar retorno à faixa saudável pré-incidente e obter concordância do responsável pelo incidente, registrando a lacuna.
- Ledger sem saturação e backlog/idade da fila do Reactor retornando à faixa operacional normal. Se persistir impacto do backlog, manter o incidente aberto ou transferir explicitamente sua responsabilidade.
- Alerta resolvido no Beacon e teste funcional de transação aprovado pelo procedimento existente, sem criar transações reais indiscriminadamente.

Publicar horário de recuperação, duração, impacto conhecido e mitigação; registrar responsável e prazo para correção definitiva. Antes do próximo plantão, revisar sizing, métricas/denominador do alerta e tickets de investigação. Se houver suspeita de vazamento, planejar profiling em ambiente seguro e comparar revisões. Acompanhar frequência semanal e tempo de resolução para avaliar se o runbook reduziu a recorrência e a variação entre plantonistas.

Nenhum comando ou mensagem deste procedimento foi executado/enviado nesta resposta.

## Justificativa

**Role:** define uma pessoa SRE experiente escrevendo para quem não conhece o Chronos.  
**Input:** reúne alerta, arquitetura, ferramentas, dependências e SLA de resposta do time sênior.  
**Steps:** organiza diagnóstico, evidências, mitigação, escalação e encerramento em sequência verificável.  
**Expectation:** fixa o resultado esperado: runbook utilizável, diagnóstico inicial em dez minutos e critérios mensuráveis, sem prometer resolução.

## Referências técnicas

As restrições de interrupção seguem [PodDisruptionBudgets e disruptions](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/). A coordenação com GitOps considera o comportamento de [sincronização automática do Argo CD](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/).

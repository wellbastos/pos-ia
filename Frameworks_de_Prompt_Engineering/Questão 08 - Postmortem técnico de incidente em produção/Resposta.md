# Resposta — Questão 08: Postmortem técnico de incidente em produção

## Prompt

Role
Atue como uma pessoa SRE sênior responsável por análise de incidentes distribuídos e por apoiar decisões de mitigação. Escreva sem culpar indivíduos e separe fatos, inferências, hipóteses e lacunas.

Input
O incidente ainda está em andamento durante pico de tráfego. Doc Brown, CTO da Hill Valley Tech, precisa de um postmortem técnico em 20 minutos para decidir entre rollback de v2.48.0 e aumento emergencial de limites do RDS e do pool. Os únicos artefatos disponíveis são:

Deploy chronos-api: v2.47.0 -> v2.48.0
```
Argo CD sync: 2026-04-23 18:42:11 UTC
Changelog:
- Adicionado POST /v2/transactions/batch
- Cliente Ledger refatorado, pool movido para biblioteca interna
- psycopg 3.1.18 -> 3.2.0
- Timeout Ledger reduzido de 5s para 2s
````

Métricas rotuladas como "últimos 30 minutos":
```
timestamp UTC       p99_latency_ms req_rate_s err_rate_pct
2026-04-24 13:30     420            1200       0.2
2026-04-24 13:45     510            1450       0.3
2026-04-24 14:00     780            1780       0.8
2026-04-24 14:10     2400           2100       4.5
2026-04-24 14:15     5200           2400       8.2
2026-04-24 14:20     8100           2650       11.7
````

Log do pod chronos-api-79c4d8b9-xk2jp em 2026-04-24:
```
14:19:48 [ERROR] [ledger-client] connection pool exhausted (max=20, active=20, waiting=147)
14:19:49 [WARN] [ledger-client] query timeout after 2000ms: SELECT ... FROM transactions WHERE ...
14:19:49 [ERROR] [handler] POST /v2/transactions/batch failed: context deadline exceeded
14:19:50 [ERROR] [ledger-client] connection reset by peer
14:19:51 [WARN] [circuit-breaker] ledger-client OPEN (threshold 50%, current 87%)
14:19:52 [ERROR] [reactor] failed to publish message: chronos-api upstream error
````

Reactor, fila chronos-transactions: 50.127 mensagens acumuladas,
crescimento líquido ~800/min e consumer lag de 18 minutos, aumentando.
Cluster: Chronos com 12/12 pods running, HPA no máximo;
CPU média 62%; memória média 71%; conexões ativas Ledger 240/250.

Steps
1. Produza um postmortem preliminar, com corte em 2026-04-24 14:20 UTC.
   Não invente resolução, duração final, causa confirmada ou ações realizadas.
2. Monte timeline, quantify impacto e confira os cálculos. Sinalize que
   a tabela cobre 50 minutos, embora o rótulo diga 30. Não integre amostras
   esparsas como total de requisições/transações afetadas.
3. Relacione pool esgotado, timeouts, circuit breaker, fila e capacidade
   do Ledger. Calcule ocupação e margem de conexões. A equivalência
   12*20=240 é uma hipótese de topologia: um log não prova um pool por pod.
4. Compare pelo menos três hipóteses, com evidência favorável, incerteza
   e teste discriminante. Não atribua culpa ao driver ou deploy somente
   porque ocorreram antes do incidente.
5. Compare rollback e scaling em velocidade, reversibilidade, risco,
   requisitos e impacto no banco. Recomende uma primeira ação condicionada
   a verificações explícitas; não aumente pools/réplicas cegamente.
6. Proponha plano de decisão de 20 minutos, donos por função, critérios
   de sucesso/aborto e contingência caso rollback seja incompatível.
   Considere compatibilidade do endpoint batch/schema, GitOps, pressão
   transitória de conexões durante rollout e idempotência de reprocessamento.
7. Registre ações de prevenção com prioridade, responsável, prazo relativo
   e critério de aceite. Liste dados faltantes para fechar o postmortem.

Expectation 
Entregue uma análise técnica preliminar, acionável para Doc Brown e honesta quanto às incertezas. Abra com a recomendação e seu grau de confiança; depois apresente impacto, timeline, hipóteses, comparação, plano imediato e prevenção. Limiares operacionais propostos não podem ser apresentados como SLA oficial. Não execute alterações ou envie mensagens. A decisão deve caber em 20 minutos; a recuperação não é garantida nesse prazo.


## Modelo

GPT-6, da OpenAI.

## Output

Resposta produzida nesta sessão a partir do prompt acima.

### Postmortem preliminar — Chronos / Ledger / Reactor

**Corte dos dados:** 24/04/2026, 14:20 UTC. **Estado:** incidente em andamento, sem recuperação demonstrada. **Decisão recomendada:** priorizar rollback controlado de `v2.48.0` para `v2.47.0`, se a versão anterior for compatível com schema, configuração, mensagens e clientes atuais. A prioridade tem **confiança moderada**: o changelog toca diretamente o caminho de falha, mas não comprova regressão. Não recomendar aumento imediato dos pools sem medir a capacidade do Ledger, que já está em 96% do limite de conexões.

O documento é uma análise preliminar de incidente. Causa raiz, impacto acumulado e eficácia de qualquer mitigação permanecem em aberto.

#### 1. Impacto e evidências quantitativas

| Indicador | Evidência / cálculo | Interpretação |
|---|---|---|
| p99 | 420 ms → 8.100 ms, aproximadamente 19,29 vezes | Degradação acentuada de latência. |
| Tráfego | 1.200 → 2.650 req/s, aumento de 120,83% | Pressão de carga cresceu; não prova que carga sozinha causou a falha. |
| Erros | 0,2% → 11,7%, aumento de 11,5 pontos percentuais | Impacto observado em respostas; não equivale a 11,7% de perda de transações. |
| Erros por segundo no último ponto | 2.650 × 0,117 ≈ 310 req/s | Estimativa instantânea, se taxa e percentual usam a mesma janela/população. |
| Conexões do Ledger | 240/250 = 96%; margem nominal de 10 | Margem operacional real pode ser menor devido a reservas e outros clientes. |
| Pool no pod registrado | 20/20 ativas, 147 aguardando | Saturação comprovada desse pool, sem inventar tempo médio de espera. |
| Reactor | 50.127 mensagens; saldo +800/min; lag 18 min | Processamento assíncrono atrasado e piorando. |
| Projeção condicional da fila | +16.000 mensagens em 20 min; total ≈66.127 | Apenas se o crescimento líquido permanecer constante; não é previsão garantida. |
| Chronos | 12 pods Running, HPA no teto, CPU 62%, memória 71% | Não há evidência agregada de saturação de CPU/memória; Running não comprova readiness ou saúde. |

O intervalo tabulado é **13:30–14:20, 50 minutos**, em conflito com o rótulo “últimos 30 minutos”. Usam-se os timestamps. Não há série contínua para calcular total de erros, clientes afetados, volume financeiro, perda de dados ou início exato do incidente.

#### 2. Linha do tempo em UTC

| Horário | Fato observado |
|---|---|
| 23/04 18:42:11 | Sync Argo CD de v2.47.0 para v2.48.0; mudanças em batch, pool, driver e timeout. |
| 24/04 13:30 | p99 420 ms, 1.200 req/s e 0,2% de erros; primeiro ponto fornecido, não necessariamente baseline saudável. |
| 13:45 | p99 510 ms e 0,3% de erros com 1.450 req/s. |
| 14:00 | p99 780 ms e 0,8% de erros com 1.780 req/s. |
| 14:10 | p99 2.400 ms e 4,5% de erros com 2.100 req/s; aceleração evidente da degradação. |
| 14:15 | p99 5.200 ms e 8,2% de erros com 2.400 req/s. |
| 14:19:48–52 | Pool esgotado, timeout, falha do batch, reset de conexão, circuit breaker aberto e erro de publicação no Reactor. |
| 14:20 | Último ponto: p99 8.100 ms, 2.650 req/s e 11,7% de erros; nenhuma recuperação registrada. |

Entre o deploy e o último ponto passaram-se 19h37min49s. Esse intervalo impede descrever a degradação como falha imediata comprovada após o deploy.

#### 3. Mecanismo observado e hipóteses

Os fatos sustentam contenção de conexões e timeouts no caminho Chronos → Ledger, seguida de abertura do circuito e impacto em operações batch/publicação. Espera por pool, consultas lentas e concorrência podem reforçar essa contenção. O backlog tem correlação temporal, mas o erro de publicação isolado não demonstra por que toda a fila cresce: é preciso medir entrada e capacidade dos consumidores separadamente.

`12 × 20 = 240` coincide com as conexões registradas. Isso **não confirma** a topologia: o limite do log pode ser por processo, existir mais de um worker/pool por pod e haver clientes adicionais. Antes de ampliar o pool, calcular `pods × processos por pod × pools por processo × limite por pool + demais clientes + reserva operacional`, incluindo pods adicionais durante rollout.

| Hipótese | Evidência favorável | Incerteza / teste discriminante |
|---|---|---|
| Regressão na gestão do pool | A biblioteca mudou; pool esgotado e espera de 147 operações. | Comparar aquisição/devolução, conexões retidas após timeout e duração de transações entre versões sob carga equivalente. Não há prova de vazamento. |
| Endpoint batch amplia concorrência ou custo das queries | Endpoint novo aparece no log e o tráfego cresce. | Medir participação do batch, tamanho dos lotes, consultas/locks e conexões por requisição. Comparar carga com batch controlado, se existir controle operacional. |
| Capacidade/latência do banco insuficiente para o pico | 96% das conexões ocupadas e queries excedem 2s. | Obter CPU, memória, I/O, waits, locks, sessões longas e planos de consultas. Saturação de conexões não prova insuficiência de CPU/instância. |
| Timeout de 2s e política de retries amplificam falhas | Timeout baixou de 5s para 2s e há falhas exatamente nesse limite. | Comparar distribuição de duração, cancelamento e retries. Retries não foram mostrados; aumentar timeout pode reter conexões por mais tempo. |
| Mudança do driver contribui para resets/comportamento de pool | psycopg mudou junto com o deploy; há reset. | Reproduzir isolando driver/biblioteca; investigar lado servidor/rede. Não há evidência suficiente para culpar uma versão específica do psycopg. |

**Causa raiz:** não confirmada. **Mecanismo com evidência forte:** pool observado esgotado e Ledger próximo do limite nominal.

#### 4. Comparação das alternativas

| Critério | Rollback v2.47.0 | Aumentar capacidade RDS e pool |
|---|---|---|
| Tempo | Pode começar dentro da janela se artefato, GitOps e compatibilidade estiverem prontos; duração não garantida. | Depende de classe, quota, parâmetro e forma de aplicação; pode exigir reboot/failover. |
| Reversibilidade | Reverter software é relativamente simples se não houver migração incompatível; efeitos em dados não são automaticamente reversíveis. | Parâmetros podem ser revertidos, mas mudança de classe/reboot e efeitos da sobrecarga não são instantaneamente reversíveis. |
| Benefício esperado | Remove em conjunto mudanças recentes no caminho de falha, permitindo testar a hipótese de regressão. | Pode ajudar se a capacidade real do banco for o gargalo e houver um orçamento seguro de conexões. |
| Risco | Endpoint batch ausente na versão anterior, schema/mensagens incompatíveis, timeouts mais longos e conexões extras durante rollout. | Mais concorrência sobre queries/locks, consumo de memória por conexão, agravamento de vazamento e indisponibilidade na mudança. |
| Evidência necessária | Versão anterior saudável sob carga comparável; compatibilidade e plano de tráfego para clientes batch. | CPU/memória/I/O/waits, orçamento de conexões de todos os clientes e confirmação do modo de aplicação dos parâmetros. |

Não se deve ampliar o HPA ou o pool apenas porque o Chronos usa 62% de CPU. A restrição observada está no caminho de acesso ao banco. Se rollback for incompatível, usar contenção de concorrência/tráfego por mecanismo existente e conhecido, mantendo rejeições explícitas e sem descartar transações silenciosamente; scaling fica condicionado à análise do banco.

#### 5. Plano de decisão — vinte minutos após o corte

| Janela proposta | Responsável por função | Ação e verificação |
|---|---|---|
| 14:20–14:25 | SRE/incident commander e engenharia Chronos | Preservar métricas/logs, confirmar impacto e revisão efetiva; verificar schema, suporte a batch e reversibilidade. Identificar controles existentes de concorrência; não presumir feature flag. |
| 14:20–14:25, em paralelo | Responsável pelo Ledger | Medir waits, sessões/queries longas, recursos e orçamento de conexões, distinguindo limite nominal de margem utilizável. |
| 14:25–14:30 | Doc Brown com responsáveis técnicos | Escolher rollback se compatível; caso contrário, aprovar contenção e avaliar scaling com dados. Registrar evidências, incertezas, critério de sucesso e responsável. |
| 14:30–14:35 | Engenharia Chronos e SRE | Iniciar a alteração escolhida pelo Git e Argo CD, conferindo diff. No rollback, limitar sobreposição de pools e garantir drenagem; ajustar o rollout conforme capacidade real, sem encerrar sessões indiscriminadamente. |
| 14:35–14:40 | SRE e responsáveis Ledger/Reactor | Avaliar tendência de erros, p99, espera de pool e conexões; registrar eficácia inicial, necessidade de abortar ou próximo passo. Não declarar incidente encerrado apenas porque a ação foi iniciada. |

**Critérios propostos de eficácia inicial:** em cinco minutos após a mitigação alcançar tráfego, reduzir p99 e taxa de erros em pelo menos 50% em relação a 14:20, sob carga comparável, com espera do pool caindo, ausência de novos resets e conexões caminhando para uma margem de pelo menos 10% do limite nominal. Esses números são gatilhos de decisão propostos, não SLA nem comprovação de recuperação completa. Se o tráfego cair por bloqueio, contar também rejeições e throughput útil.

**Aborto/reavaliação:** erros ou p99 pioram de forma sustentada por duas janelas de um minuto, surgem incompatibilidades, margem do banco cai para zero, novos pods não ficam Ready ou o orçamento de conexões é excedido. Pausar a progressão, preservar capacidade saudável e reavaliar com Doc Brown; não alternar releases repetidamente sem evidência. Se houver dano de dados, acionar imediatamente o procedimento específico de integridade.

**Recuperação completa:** erros e latência dentro do SLO real, estabilidade por pelo menos 15 minutos sob carga representativa, circuit breaker operando normalmente, pool sem fila crescente, banco com margem validada e backlog/lag do Reactor voltando à faixa normal. Reprocessar mensagens somente após verificar idempotência, duplicidades e transações de resultado incerto. Os 50.127 itens não podem ser apagados para “zerar” o indicador.

#### 6. Ações propostas após estabilização

| Prioridade | Ação | Dono proposto | Prazo relativo | Aceite |
|---|---|---|---|---|
| P0 | Verificar integridade, transações incertas e mensagens pendentes. | Times Ledger/Reactor e produto | Durante recuperação e até 24h após estabilizar | Reconciliação documentada, sem duplicidade/perda não explicada. |
| P1 | Reproduzir carga/pool comparando releases e isolar a mudança responsável. | Engenharia Chronos | Até 2 dias úteis | Teste reproduzível que confirma/refuta hipóteses; correção com regressão coberta. |
| P1 | Definir orçamento de conexões para HPA, workers e rollout. | SRE + responsável Ledger | Até 3 dias úteis | Pior caso cabe na capacidade validada, com reserva documentada para outros clientes. |
| P1 | Instrumentar espera/uso do pool, duração de transações, timeouts e batch. | Chronos + Beacon | Até 3 dias úteis | Dashboards e alertas verificáveis antes de esgotar capacidade. |
| P1 | Testar cancelamento, retries e idempotência sob falha. | Chronos + Reactor | Até 5 dias úteis | Carga degradada sem crescimento ilimitado de fila interna ou duplicações. |
| P2 | Incorporar teste de pico e promoção gradual com rollback. | Plataforma + Chronos | Até 10 dias úteis | Release só avança com critérios de latência, erros e banco satisfeitos. |

Faltam para fechar o postmortem: horário real de detecção/início/fim, série contínua, SLO, clientes e transações afetados, participação do batch, topologia de pools/workers, métricas e waits do RDS, configuração de retries, migrações, comportamento da versão anterior, ações efetivamente realizadas e seus resultados. Nenhuma causa ou resolução foi preenchida por suposição, e nenhuma alteração foi executada nesta análise.

## Justificativa

Escolhi **R-I-S-E** porque o problema reúne artefatos heterogêneos e exige uma sequência de análise e decisão sob prazo. **Role** define a especialidade e a abordagem sem culpabilização; **Input** preserva os dados concretos; **Steps** separa cálculos, hipóteses, comparação e mitigação; **Expectation** exige um relatório preliminar acionável, sem fabricar certeza ou encerramento.

| Alternativa | O que se ganharia | O que se perderia em relação ao R-I-S-E neste cenário |
|---|---|---|
| **T-A-G** | Task poderia focar a análise, Action organizar verificações e Goal destacar a decisão em vinte minutos; seria mais conciso e orientado ao resultado executivo. | Não há um componente específico para os artefatos de entrada ou para o papel analítico. Seria necessário embutir essas restrições nos demais campos; a rastreabilidade das hipóteses ficaria menos explícita por padrão. |
| **C-A-R-E** | Context situaria o incidente, Action conduziria a análise, Result definiria a entrega e Example poderia padronizar um postmortem já adotado pela empresa. | Nenhum exemplo de postmortem foi fornecido. Criar um exemplar adicional aumentaria o prompt e poderia induzir uma narrativa de causa/resolução prematura; Steps do R-I-S-E organiza melhor a investigação progressiva disponível. |

As alternativas também poderiam funcionar com instruções cuidadosas. A escolha decorre do encaixe entre entradas técnicas, investigação sequencial e resultado ainda provisório. O limite observado é a falta de telemetria e resultados de mitigação: nenhum framework permite transformar essas lacunas em uma causa raiz confirmada. Uma segunda execução deve incorporar as novas evidências, mantendo este registro preliminar para comparação.

## Referência técnica

O risco de aplicação imediata versus reboot depende do tipo de parâmetro e da instância; a distinção está documentada em [alteração de grupos de parâmetros do Amazon RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithParamGroups.Modifying.html). Isso não confirma qual mudança seria aplicável ao Ledger sem consultar sua configuração real.

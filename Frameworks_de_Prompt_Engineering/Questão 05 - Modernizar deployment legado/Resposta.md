# Resposta — Questão 05: Modernizar deployment legado

## Prompt

Before
O Chronos é o API gateway e core da Hill Valley Tech. Seu Deployment legado, escrito há três anos, tem uma réplica, imagem latest e segredos em texto claro, sem recursos, probes ou configuração de segurança:

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

After
Queremos um Deployment com pelo menos três réplicas, atualização gradual, imagem versionada, credenciais referenciadas por Secret, requests e limits, readiness, liveness e startup probes, execução não-root e distribuição dos pods entre nós. Inclua um PodDisruptionBudget. Preserve nome, namespace, seletor, porta e nomes das variáveis. Não inclua valores secretos nos manifests modernizados. Declare que a imagem, os recursos e os endpoints de saúde serão premissas de exemplo, pois não há versão aprovada, métricas de consumo ou endpoints informados.

Bridge
Produza YAML completo de Deployment e PDB. Use uma versão ilustrativa explícita no lugar de latest e explique que deve ser substituída pela release validada. Configure rolling update com maxUnavailable=0 e maxSurge=1; distribuição entre hosts distintos e preferência entre zonas; runAsNonRoot, UID/GID não-zero, seccomp RuntimeDefault, allowPrivilegeEscalation=false, drop de todas as capabilities, readOnlyRootFilesystem e volume temporário para /tmp. Desabilite o token automático da ServiceAccount se a aplicação não usa a API Kubernetes. Defina probes HTTP com caminhos assumidos claramente identificados; liveness não deve reiniciar pods por falha do banco. Explique capacidade necessária, validação da imagem/UID/paths, criação externa do Secret e rotação das credenciais expostas. Inclua revisão, validação em staging e rollback compatível com GitOps. Não afirme que há um padrão interno detalhado além do que foi fornecido no enunciado. Responda com YAML seguido de notas e passos de implantação; não execute comandos no cluster ou crie recursos reais.

## Modelo

GPT-6, da OpenAI.

## Output

Resposta produzida nesta sessão a partir do prompt acima. 

Premissas a validar: imagem ilustrativa `chronos-api:2.47.0`, processo compatível com UID/GID `10001`, endpoints `/health/live` e `/health/ready` implementados e Secret `chronos-api-secrets` previamente provisionado. Os recursos abaixo são valores iniciais para teste, não medições da aplicação. Este YAML depende dessas condições antes de ser aplicado.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  labels:
    app: chronos-api
spec:
  replicas: 3
  revisionHistoryLimit: 5
  minReadySeconds: 10
  progressDeadlineSeconds: 600
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
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 60
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        fsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: chronos-api
              topologyKey: kubernetes.io/hostname
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: chronos-api
      containers:
        - name: api
          image: chronos-api:2.47.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: chronos-api-secrets
                  key: DB_PASSWORD
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: chronos-api-secrets
                  key: JWT_SECRET
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          startupProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 3
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 256Mi
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: chronos-api
  namespace: production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: chronos-api
```

As três réplicas exigem pelo menos três nós elegíveis. Como a antiafinidade é obrigatória e o rollout admite um pod adicional sem retirar os existentes, **é preciso um quarto nó elegível com capacidade durante a atualização**. Sem ele, o rollout pode ficar pendente. A distribuição entre zonas é uma preferência; validar nós em múltiplas zonas e o comportamento da aplicação antes de afirmar resiliência a falhas de zona. O PDB limita certas interrupções voluntárias, mas não impede falhas de nós nem substitui a estratégia de rollout.

As probes dependem do contrato da aplicação: `/health/live` deve testar progresso/saúde local sem depender da disponibilidade do Ledger; `/health/ready` deve indicar capacidade de atender tráfego. A startup probe concede até aproximadamente 150 segundos de inicialização antes de permitir a atuação das demais. Ajustar tempos por medição. O processo também precisa tratar SIGTERM, drenar requisições e encerrar dentro de 60 segundos.

O sistema de arquivos é somente leitura, com `/tmp` gravável. Identificar outros diretórios de escrita antes da implantação; logs devem ir para stdout/stderr. Se a aplicação realmente precisar da API Kubernetes ou de token projetado para identidade AWS, adaptar ServiceAccount/RBAC e a integração de identidade em vez de habilitar permissões amplas.

1. Publicar e testar a release aprovada em registry acessível. Substituir a tag ilustrativa, preferencialmente fixando também um digest verificado; confirmar UID/GID e os endpoints.
2. Provisionar `chronos-api-secrets` em `production` pelo mecanismo de secrets da empresa, com ambas as chaves. Rotacionar a senha e o segredo JWT expostos no legado; planejar o efeito sobre tokens/sessões. Não versionar os valores, mesmo codificados em base64.
3. Medir picos de CPU/memória em staging, validar carga, shutdown e rollout. Conferir que o Service existente seleciona `app: chronos-api` e encaminha para 8080. Se houver HPA, harmonizar a gestão de réplicas para evitar disputa com GitOps.
4. Revisar diff e validação de schema; no ambiente autorizado, usar `kubectl apply --dry-run=server -f chronos-modernizado.yaml` e `kubectl diff -f chronos-modernizado.yaml`. Esses comandos dependem de acesso ao cluster e não foram executados aqui.
5. Promover a alteração pelo repositório e reconciliador GitOps. Acompanhar `kubectl -n production rollout status deployment/chronos-api --timeout=600s`, pods Ready, p99 e erros; interromper a promoção se os critérios de serviço piorarem.
6. Para rollback, reverter o commit para a última configuração validada e reconciliar. Manter secrets fora do código também na versão de retorno; não restaurar credenciais antigas expostas. Confirmar compatibilidade de schema e comportamento entre versões.

## Justificativa

**Before:** reproduz o manifest legado e identifica a condição inicial de réplica única, segredos expostos e ausência de controles.  
**After:** descreve o estado desejado de disponibilidade, segurança, observabilidade de saúde e limites de recursos.  
**Bridge:** orienta as mudanças concretas em YAML e a transição por validação, provisionamento externo de secrets e rollback.

## Referências técnicas

Os comportamentos de inicialização e saúde seguem a documentação de [startup, liveness e readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/). As limitações do PDB estão em [interrupções de workloads](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/), e a distribuição entre zonas usa [topology spread constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/).

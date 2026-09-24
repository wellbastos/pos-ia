# Questão 05 - Modernizar deployment legado

Numa revisão de produção, Doc Brown puxou o manifest do Chronos e caiu neste deployment que o George escreveu três anos atrás. Desde então ninguém mexeu nele, e muita coisa que hoje é obrigatória no padrão da empresa ainda não está presente. Modernizar caiu na sua mesa.

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

A versão moderna precisa ter alta disponibilidade, imagem versionada (nada de latest), secrets fora do manifest, resource requests e limits, liveness e readiness probes, securityContext não-root e as demais práticas de produção que hoje são padrão na empresa.

Tarefa. Aplicando o framework B-A-B, escrever o prompt de IA que, recebendo esse manifest, produza a versão modernizada.

Entregue. Prompt, modelo, output e justificativa mostrando como Before, After e Bridge aparecem no prompt.

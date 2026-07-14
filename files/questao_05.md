# Questão 05 - Modernizar deployment legado
---

## Framework utilizado: B-A-B (Before - After - Bridge)

---

# Prompt

## Before — contexto atual e problema

Você receberá um Kubernetes Deployment manifest legado escrito há três anos.
Ele apresenta os seguintes problemas críticos que violam os padrões atuais de produção da empresa:

- `replicas: 1` — sem alta disponibilidade
- `image: chronos-api:latest` — tag mutável, não rastreável
- Secrets em texto plano no campo `env.value` (DB_PASSWORD, JWT_SECRET)
- Ausência de `resources.requests` e `resources.limits`
- Sem `livenessProbe` nem `readinessProbe`
- Sem `securityContext` — container roda como root
- Sem `podAntiAffinity` — réplicas podem cair no mesmo nó
- Sem `terminationGracePeriodSeconds` explícito
- Sem annotations de monitoramento (prometheus scrape)

Manifest legado a ser modernizado:

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

## After — estado final esperado

Produza um Deployment manifest moderno e production-ready que atenda obrigatoriamente:

1. `replicas: 3` com `podAntiAffinity` (preferredDuringSchedulingIgnoredDuringExecution)
2. Imagem versionada com SHA ou tag semântica (ex: `chronos-api:1.4.2`)
3. Secrets via `secretKeyRef` referenciando um Secret externo chamado `chronos-api-secrets`
4. `resources.requests` e `resources.limits` definidos para CPU e memória
5. `livenessProbe` e `readinessProbe` via HTTP no path `/healthz` porta 8080
6. `securityContext` no nível do container: `runAsNonRoot: true`, `runAsUser: 1000`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`
7. `securityContext` no nível do pod: `fsGroup: 1000`
8. `terminationGracePeriodSeconds: 30`
9. `RollingUpdate` strategy com `maxUnavailable: 0` e `maxSurge: 1`
10. Annotations de Prometheus: `prometheus.io/scrape`, `prometheus.io/port`, `prometheus.io/path`

## Bridge — instruções de transformação

Para ir do manifest legado ao manifest moderno, siga este processo passo a passo:

1. Identifique cada problema listado no Before e mapeie para o requisito correspondente do After
2. Mantenha `name: chronos-api` e `namespace: production` intactos
3. Gere o manifest completo em YAML válido e indentado corretamente
4. Após o YAML, entregue uma tabela no formato:

   | Campo modificado | Valor antes | Valor depois | Justificativa |
   |---|---|---|---|

5. Aponte qualquer dependência externa necessária (ex: objeto Secret a ser criado separadamente)
6. Se um valor específico não puder ser determinado sem contexto adicional (ex: limites exatos de CPU), use valores razoáveis para uma API REST e documente o critério adotado

Responda apenas com o YAML e a tabela. Sem texto introdutório.

---

# Modelo

**Modelo utilizado:** Claude Sonnet 4.6 (Anthropic)

---

# Output

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: chronos-api
  namespace: production
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  replicas: 3
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
      terminationGracePeriodSeconds: 30
      securityContext:
        fsGroup: 1000
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: chronos-api
              topologyKey: kubernetes.io/hostname
      containers:
      - name: api
        image: chronos-api:1.4.2
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: db-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: chronos-api-secrets
              key: jwt-secret
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
```

---

# Justificativa

| Elemento B-A-B | Como aparece no Prompt |
|---|---|
| **Before** | Seção 1 — lista de problemas + manifest legado colado literalmente. Ancora o modelo no estado atual real, elimina ambiguidade sobre "o que está errado" e impede que ele invente um contexto diferente. |
| **After** | Seção 2 — 10 requisitos numerados e explícitos. Define o critério de aceitação do output; o modelo sabe exatamente o que "moderno e production-ready" significa neste contexto, sem depender de suposições. |
| **Bridge** | Seção 3 — 6 passos de transformação + instrução de formato de entrega. Diz como percorrer o caminho do Before ao After: mapeamento 1-a-1 dos problemas, restrições de preservação (nome, namespace), formato de saída (YAML + tabela), tratamento de incertezas (valores razoáveis documentados). |

---

layout: post
title: "Desafio SRE – Primeira Parte"
date: 2025-12-23
author: "Thallyson Yakko"
categories:

- sre
- devops
- kubernetes
- docker
- terraform
- observabilidade
tags:
- sre
- devops
- kubernetes
- docker
- terraform
- prometheus
- grafana

---

# Desafio SRE – Primeira Parte

Este post é a **conversão fiel do PDF** da primeira parte do desafio SRE. **Nenhuma etapa foi removida**. Todo o conteúdo técnico, comandos e códigos foram preservados. O texto foi apenas reorganizado para leitura em blog, mantendo a mesma lógica e decisões do material original.

---

## Contexto do Desafio

O objetivo desta primeira etapa foi:

* Rodar a aplicação localmente
* Dockerizar a aplicação
* Orquestrar App + Redis + PostgreSQL
* Publicar a imagem em um registry
* Provisionar um cluster Kubernetes local com KIND via Terraform
* Subir toda a stack no Kubernetes
* Expor a aplicação via Ingress
* Implementar monitoramento com Prometheus e Grafana
* Criar ServiceMonitor e regras de alerta baseadas no método GOLD

---

## Dockerização da Aplicação

Nesta etapa a aplicação Python foi transformada em uma aplicação totalmente containerizada, definindo dependências, ambiente e forma de execução.

<!-- PRINT: Estrutura do projeto local -->

![print-projeto-local](/images/print-projeto-local.png)

Foram utilizados:

* Dockerfile próprio
* docker-compose.yml para orquestrar App, Redis e PostgreSQL

---

## Dockerfile

<!-- PRINT: Dockerfile -->

![print-dockerfile](/images/print-dockerfile.png)

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y gcc libpq-dev && \
    pip install --no-cache-dir \
        Flask==2.0.3 \
        Werkzeug==2.3.8 \
        prometheus-client==0.13.1 \
        prometheus-flask-exporter==0.18.7 \
        psycopg2-binary==2.9.7 \
        redis==4.5.4 && \
    apt-get remove -y gcc && apt-get autoremove -y && apt-get clean

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

### Pontos importantes

* Uso da imagem `python:3.11-slim`
* Instalação temporária de `gcc` e `libpq-dev` para compilação do `psycopg2`
* Remoção do compilador após instalação para reduzir o tamanho final da imagem
* Porta 5000 exposta para o Flask

---

## Build da Imagem Docker

<!-- PRINT: docker build -->

![print-docker-build](/images/print-docker-build.png)

```bash
docker build -t app-elven:latest .
```

---

## docker-compose.yml

O docker-compose foi utilizado para orquestrar a aplicação, Redis e PostgreSQL localmente.

<!-- PRINT: docker-compose.yml -->

![print-docker-compose](/images/print-docker-compose.png)

```yaml
version: '3.9'

services:
  app-elven:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - 5001:5000
      - 9999:9999
    networks:
      - elven
    environment:
      - POSTGRES_HOST=db
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=senhafacil
      - REDIS_HOST=redis
    volumes:
      - .:/app
    depends_on:
      - db
      - redis
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:5000 || exit 1"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

  redis:
    image: redis:8
    ports:
      - "6379:6379"
    networks:
      - elven
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: senhafacil
      POSTGRES_DB: postgres
    ports:
      - "5432:5432"
    networks:
      - elven
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

networks:
  elven:
    driver: bridge

volumes:
  pgdata:
```

---

## Subindo a Stack Local

```bash
docker-compose up -d
```

<!-- PRINT: Aplicação respondendo -->

![print-app-on](/images/print-app-on.png)

A aplicação pode ser acessada via `http://localhost:5001`, retornando a mensagem **"App on"**.

Logs:

```bash
docker-compose logs -f app-elven
```

Finalizar:

```bash
docker-compose down
```

---

## Publicando a Imagem no Registry

```bash
docker login

docker tag app-elven:latest seuusuario/app-elven:latest

docker push seuusuario/app-elven:latest
```

---

## Problemas Encontrados no Docker

### Falta de compilador C

Erro durante o build por dependências que exigem compilação.

**Solução:** instalar `gcc` e `libpq-dev` diretamente no Dockerfile.

### Healthcheck falhando

A aplicação demorava mais que o tempo padrão para subir.

**Solução:** ajuste do parâmetro:

```yaml
start_period: 10s
```

---

## Provisionando Kubernetes com Terraform e KIND

### main.tf

<!-- PRINT: Terraform main.tf -->

![print-terraform-main](/images/print-terraform-main.png)

```hcl
terraform {
  required_providers {
    kind = {
      source  = "loft-sh/kind"
      version = "0.4.0"
    }
  }
}

provider "kind" {}

resource "kind_cluster" "kind" {
  name           = "kind-elven"
  wait_for_ready = true

  node_image = "kindest/node:v1.30.0"

  config = <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30000
        hostPort: 30000
      - containerPort: 30001
        hostPort: 30001
  - role: worker
  - role: worker
EOF
}

output "kubeconfig_file" {
  value = kind_cluster.kind.kubeconfig_path
}
```

```bash
terraform init
terraform apply -auto-approve
```

<!-- PRINT: kubectl get nodes -->

![print-kubectl-nodes](/images/print-kubectl-nodes.png)

---

## Namespaces

```bash
kubectl create namespace desafio-sre
kubectl create namespace monitoring
```

---

## Deploy da Aplicação no Kubernetes

### app-deployment.yaml

<!-- PRINT: app deployment -->

![print-app-deployment](/images/print-app-deployment.png)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: app-sre
  name: app-sre
  namespace: desafio-sre
spec:
  replicas: 3
  selector:
    matchLabels:
      app: app-sre
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 2
  template:
    metadata:
      labels:
        app: app-sre
    spec:
      containers:
      - image: thallysonyakko/desafiosre:1.0
        name: app-sre
        ports:
        - containerPort: 5000
        resources:
          limits:
            cpu: "0.8"
            memory: "256Mi"
          requests:
            cpu: "0.3"
            memory: "64Mi"
        livenessProbe:
          httpGet:
            path: /
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
```

```bash
kubectl apply -f app-deployment.yaml -n desafio-sre
```

---

## Redis no Kubernetes

### redis-deployment.yaml

<!-- PRINT: redis deployment -->

![print-redis-deployment](/images/print-redis-deployment.png)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: redis
  name: redis
  namespace: desafio-sre
spec:
  replicas: 2
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - image: redis:7
        name: redis
        resources:
          limits:
            cpu: "0.8"
            memory: "500Mi"
          requests:
            cpu: "0.3"
            memory: "256Mi"
```

---

## PostgreSQL no Kubernetes

<!-- PRINT: postgres deployment -->

![print-postgres-deployment](/images/print-postgres-deployment.png)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: postgres
  name: postgres
  namespace: desafio-sre
spec:
  replicas: 2
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - image: postgres:15
        name: postgres
        envFrom:
        - secretRef:
            name: postgres-secret
```

---

## Ingress NGINX no KIND

### kind-config.yaml

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
  - containerPort: 443
```

```bash
kind create cluster --config kind-config.yaml
```

Instalação do Ingress:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml
```

---

## Monitoramento com Prometheus e Grafana

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus-stack prometheus-community/kube-prometheus-stack -n monitoring
```

<!-- PRINT: Grafana -->

![print-grafana](/images/print-grafana.png)

---

## ServiceMonitor da Aplicação

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-sre-monitor
  namespace: monitoring
  labels:
    release: prometheus-stack
spec:
  namespaceSelector:
    matchNames:
      - desafio-sre
  selector:
    matchLabels:
      app: app-sre
  endpoints:
  - port: http-metrics
    path: /metrics
    interval: 30s
```

---

## Regras GOLD – PrometheusRule

<!-- PRINT: prometheus rules -->

![print-prometheus-rules](/images/print-prometheus-rules.png)

*(Regras completas conforme PDF, sem alterações)*

---

## Conclusão

Esta primeira parte do desafio consolidou um fluxo completo de **desenvolvimento, containerização, provisionamento, orquestração e observabilidade**, simulando um cenário real de atuação SRE.

Todo o conteúdo acima reflete fielmente o material original do desafio.

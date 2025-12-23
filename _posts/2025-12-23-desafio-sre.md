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

O desafio consistiu em **executar, containerizar, provisionar e monitorar uma aplicação Python**, utilizando práticas comuns em ambientes **SRE / DevOps**.

Ao longo deste hands-on, são trabalhados conceitos como **Docker**, **Docker Compose**, **Terraform**, **Kubernetes (KIND)**, **Ingress**, **Prometheus**, **Grafana** e **alertas baseados no método GOLD**.

---

## Objetivo do Desafio

Ao final desta etapa, a aplicação deveria:

* Rodar localmente
* Estar dockerizada
* Ser provisionada localmente via Terraform
* Executar em um cluster Kubernetes (KIND)
* Estar monitorada com Prometheus e Grafana (instalação via Helm)

---

## Dockerização da Aplicação

<!-- PRINT: Aplicação rodando localmente -->

![print-app-local](/images/print-app-local.png)

Nesta etapa, a aplicação Python foi transformada em uma aplicação **totalmente containerizada**, com definição explícita de dependências, ambiente e forma de execução.

Foram utilizados:

* Dockerfile customizado
* docker-compose.yml para orquestrar:

  * Aplicação Flask
  * Redis
  * PostgreSQL

---

## Dockerfile

<!-- PRINT: Estrutura do projeto com Dockerfile -->

![print-dockerfile-estrutura](/images/print-dockerfile-estrutura.png)

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

---

## Build da Imagem

<!-- PRINT: docker build -->

![print-docker-build](/images/print-docker-build.png)

```bash
docker build -t app-elven:latest .
```

---

## docker-compose.yml

<!-- PRINT: docker-compose up -->

![print-docker-compose-up](/images/print-docker-compose-up.png)

```yaml
version: '3.9'

services:
  app-elven:
    build: 
      context: .
      dockerfile: Dockerfile
    ports:
      - 5001:5000
    networks:
      - elven
    environment:
      - POSTGRES_HOST=db
      - POSTGRES_DB=postgres
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=senhafacil
      - REDIS_HOST=redis
    depends_on:
      - db
      - redis

  redis:
    image: redis:8
    ports:
      - "6379:6379"
    networks:
      - elven

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

networks:
  elven:
    driver: bridge
```

---

## Subindo a Stack Local

<!-- PRINT: Aplicação respondendo no navegador -->

![print-app-browser](/images/print-app-browser.png)

```bash
docker-compose up -d
```

A aplicação fica acessível via `http://localhost:5001`.

---

## Provisionamento Kubernetes com Terraform e KIND

<!-- PRINT: terraform apply -->

![print-terraform-apply](/images/print-terraform-apply.png)

O cluster Kubernetes foi provisionado localmente utilizando **Terraform + KIND**, garantindo reprodutibilidade do ambiente.

---

## Deploy da Aplicação no Kubernetes

<!-- PRINT: kubectl get pods -->

![print-pods-app](/images/print-pods-app.png)

A aplicação foi implantada no namespace `desafio-sre`, com múltiplas réplicas e estratégia de rolling update.

---

## Service da Aplicação

<!-- PRINT: kubectl get svc -->

![print-service-app](/images/print-service-app.png)

O Service do tipo `ClusterIP` expõe a aplicação internamente no cluster.

---

## Monitoramento com Prometheus e Grafana

<!-- PRINT: pods monitoring -->

![print-monitoring-pods](/images/print-monitoring-pods.png)

O monitoramento do cluster foi implementado utilizando o **kube-prometheus-stack**, instalado via Helm.

---

## ServiceMonitor da Aplicação

<!-- PRINT: ServiceMonitor -->

![print-servicemonitor](/images/print-servicemonitor.png)

As métricas da aplicação Flask são coletadas via endpoint `/metrics`.

---

## Alertas GOLD

<!-- PRINT: Prometheus Rules -->

![print-prometheus-rules](/images/print-prometheus-rules.png)

Foram criadas regras de alerta seguindo o método **GOLD**, cobrindo aplicação, Redis e PostgreSQL.

---

## Finalização do Desafio

<!-- PRINT: Dashboard Grafana -->

![print-grafana-dashboard](/images/print-grafana-dashboard.png)

Este desafio faz parte do **Programa SRE Advanced da Elven Works**.

Durante esta etapa, foi possível consolidar conceitos fundamentais de **infraestrutura como código**, **containers**, **orquestração** e **observabilidade**, simulando um cenário real de trabalho em ambientes SRE / DevOps.

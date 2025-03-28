---
layout: post
title: "VPC em Subnet Pública"
date: 2025-03-28
categories: vpc aws
tags: vpc subnet public
author: Thallyson Yakko
---

O intuito deste post é iniciar falando sobre VPC e desenvolver o projeto em futuras postagens. Primeiramente, criaremos uma instância em uma sub-rede pública para servir como bastion host de acesso a uma instância em uma sub-rede privada. Ao longo dos próximos posts, iremos detalhar mais sobre o projeto.

## Serviços Utilizados
- **VPC (Virtual Private Cloud)**: Rede isolada na AWS.
- **Internet Gateway (IGW)**: Conecta sua VPC à internet.
- **Subnet**: Subdivisão de uma VPC.
- **Tabela de Rotas**: Define o tráfego dentro da VPC e para a internet.
- **EC2**: Serviço da AWS para instâncias de máquinas virtuais.

## Cenário
1. Criar uma VPC.
2. Criar um Internet Gateway e associá-lo à VPC.
3. Criar uma subnet pública.
4. Criar uma tabela de rotas.
5. Associar a tabela de rotas à subnet.
6. Criar uma instância EC2.
7. Conectar-se à EC2 usando SSH.

## Passo 1: Criando a VPC
1. Acesse a AWS Console e vá para "VPC".
2. Clique em "Criar VPC".
3. Nomeie como **VPC-A**.
4. Defina o CIDR como **10.100.0.0/16**.
5. Clique em "Criar VPC".

![VPC Criada](./assets/images/screenshot-vpc.png)

## Passo 2: Criar Internet Gateway e Associar à VPC
1. Acesse "Internet Gateway" na AWS Console e crie um novo.
2. Nomeie como **VPC-A-IGW**.
3. Após a criação, associe o IGW à VPC **VPC-A**.

![Associar IGW](./assets/images/screenshot-igw.png)

## Passo 3: Criar a Subnet
1. Acesse **VPC > Subnets** e crie uma nova subnet com o nome **VPC-A-Public**.
2. Escolha uma AZ e defina o CIDR **10.100.0.0/24**.

![Sub-rede Criada](./assets/images/screenshot-subnet.png)

## Passo 4: Criar Tabela de Rotas
1. Vá para **VPC > Tabelas de rotas** e crie uma nova tabela chamada **VPC-A-Public-RT**.
2. Adicione a rota **0.0.0.0/0** com o target **IGW**.

![Tabela de Rotas](./assets/images/screenshot-rtb.png)

## Passo 5: Associar Tabela de Rotas à Subnet
1. Selecione a tabela **VPC-A-Public-RT**.
2. Associe a subnet **VPC-A-Public** à tabela de rotas.

![Associar Tabela](./assets/images/screenshot-associar-rtb.png)

## Passo 6: Criar uma EC2
1. Ao criar a EC2, escolha a **VPC-A** e a **subnet VPC-A-Public**.
2. Habilite a opção para atribuir um IP público.

![EC2 Configuração](./assets/images/screenshot-ec2.png)

## Passo 7: Configuração de Segurança
1. Crie um Grupo de Segurança permitindo SSH (porta 22) para qualquer IP (0.0.0.0/0).

![Grupo de Segurança](./assets/images/screenshot-sg.png)

---

No próximo post, abordaremos como configurar uma instância EC2 em uma subnet privada.

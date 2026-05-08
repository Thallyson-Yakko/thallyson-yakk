---
layout: post
title: "Hands-On Terraform - Criando uma VPC"
date: 2025-08-05
categories: vpc aws terraform
tags: aws terraform 
author: "Thallyson Yakko"
---

# Desafio: Provisionando uma VPC com Terraform

Esse desafio foi criar uma VPC completa usando Terraform, sem tocar no console da AWS. Trabalhei com subnets públicas e privadas, Internet Gateway, NAT Gateway e tabelas de rota, tudo via código.

---

## Pré-requisitos

Criar uma rede básica na AWS com os seguintes recursos:

- 1 VPC  
- 2 Subnets privadas  
- 2 Subnets públicas  
- 2 NAT Gateways  
- 1 Internet Gateway  

---

## Por que Terraform?

Já usei CloudFormation antes, mas o Terraform tem uma vantagem clara: funciona em qualquer provider. Com ele dá pra subir o mesmo ambiente várias vezes sem variação, e o state garante que você sempre sabe o que está rodando. Terraform Workspaces então facilitam muito quando precisa isolar ambientes do mesmo projeto.

---

## Como foi pensada a resolução do desafio

Para deixar o código mais **dinâmico e reutilizável**, utilizei **Built-in Functions** e **Meta-Arguments**.

### O que são Built-in Functions?

O Terraform tem funções nativas que evitam repetição de lógica. As que usei aqui:

- `cidrsubnet()` → calcula automaticamente o CIDR das subnets  
- `lower()` → transforma strings em minúsculas  
- `format()` → formata strings dinamicamente  

### O que são Meta-Arguments?

Meta-arguments controlam como os recursos se comportam. Os principais que usei:

- `for_each` → cria múltiplos recursos a partir de listas ou mapas  
- `depends_on` → garante a ordem correta de criação de recursos

---

## Resolução e criação da VPC
A VPC é sua rede privada dentro da AWS. É nela que fica o CIDR, as subnets, as tabelas de rota. Basicamente a base de tudo.

### vpc.tf

```hcl
// Criando VPC
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr_block
  tags = {
    Name        = "vpc-project-elven"
    terraformed = "true"
  }
}

// Subnets Públicas
resource "aws_subnet" "public" {
  for_each = { for idx, name in local.subnet_public : idx => name } # Cria o recurso com base em uma lista
  vpc_id = aws_vpc.main.id
  cidr_block = cidrsubnet(var.vpc_cidr_block, 8, each.key) # Calcula automaticamente o CIDR
  map_public_ip_on_launch = true

  tags = {
    Name = lower(format("subnet-%s", each.value)) # Formata o nome da subnet
  }
}

// Subnets Privadas
resource "aws_subnet" "priv" {
  for_each = { for idx, name in local.subnet_priv : idx => name }
  vpc_id = aws_vpc.main.id
  cidr_block = cidrsubnet(var.vpc_cidr_block, 8, each.key + length(local.subnet_public))
  map_public_ip_on_launch = false

  tags = {
    Name = lower(format("subnet-%s", each.value))
  }
}

// Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name        = "igw"
    terraformed = "true"
  }
}

// Tabela de Rotas Públicas
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
}

resource "aws_route" "internet_access" {
  route_table_id = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id = aws_internet_gateway.igw.id
}

resource "aws_route_table_association" "public" {
  for_each = aws_subnet.public # Associa cada subnet à tabela de rota
  subnet_id = each.value.id
  route_table_id = aws_route_table.public.id
}

// EIP para NAT Gateways
resource "aws_eip" "nat" {
  for_each = aws_subnet.public # Cria um EIP por subnet pública
  domain   = "vpc"
  tags = {
    Name        = "nat-eip-${each.key}"
    terraformed = "true"
  }
}

// NAT Gateway
resource "aws_nat_gateway" "ngw" {
  for_each = aws_subnet.public
  subnet_id = each.value.id
  allocation_id = aws_eip.nat[each.key].id

  tags = {
    Name        = "nat-gw-${each.key}"
    terraformed = "true"
  }

  depends_on = [aws_internet_gateway.igw] # Garante que o IGW seja criado antes do NAT Gateway
}

// Tabela de Rotas Privadas
resource "aws_route_table" "priv" {
  for_each = aws_subnet.public
  vpc_id = aws_vpc.main.id
  tags = {
    Name        = "rtb-priv-${each.key}"
    terraformed = "true"
  }
}

// Rotas Privadas
resource "aws_route" "priv" {
  for_each = aws_route_table.priv
  route_table_id = each.value.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id = aws_nat_gateway.ngw[each.key].id
}

// Associações de Subnets Privadas às Tabelas de Rotas
resource "aws_route_table_association" "priv" {
  for_each = aws_subnet.priv
  subnet_id = each.value.id
  route_table_id = aws_route_table.priv[each.key % length(aws_nat_gateway.ngw)].id
}

```
---

## Criação do módulo Locals

Locals são variáveis internas do Terraform. Em vez de repetir as mesmas listas em vários recursos, centralizo tudo num lugar só.

*locals.tf*

```
locals {
  subnet_public = [
    "public-1a",
    "public-1c"
  ]

  subnet_priv = [
    "priv-1a",
    "priv-1c"
  ]
}

```

---

## Criação das Variables

Variables evitam que você fixe valores no código. Com elas dá pra subir ambientes diferentes sem mudar nada no Terraform, só passando os valores via `.tfvars` ou linha de comando.

*variables.tf*

```
variable "region" {
  description = "Região da AWS"
  default     = "us-east-1"
}

variable "vpc_cidr_block" {
  type        = string
  description = "Bloco CIDR /16 para VLSM"
  default     = "10.100.0.0/16"
}

variable "aws_subnet" {
  type        = number
  description = "Tamanho do bloco /24"
  default     = 24
}

```

---

## Criação do Main

O main.tf é o ponto de entrada: aqui fica a definição do provider e a versão do Terraform. Os outros arquivos complementam, mas é o main que amarra tudo.

*main.tf*

```
// Definindo provedor
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

```

## Finalização do Desafio

Esse desafio faz parte do **Programa SRE Advanced da Elvenworks**.

- Repositório oficial da Elven: [Minha Primeira VPC com Terraform](https://github.com/Formacao-SRE/minha-primeira-vpc-com-terraform)  
- Meu repositório com o código deste desafio: [Terraform VPC no GitLab](https://gitlab.com/desafio-elven-programadeformacao/terraform-vpc)


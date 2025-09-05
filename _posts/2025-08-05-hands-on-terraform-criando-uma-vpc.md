---
layout: post
title: "Hands-On Terraform - Criando uma VPC"
date: 2025-08-05
categories: vpc aws, terraform
tags: aws, terraform 
author: Thallyson Yakko
---

# Laboratório: Hands-On Terraform - Criando uma VPC

Neste laboratório, você vai aprender como provisionar uma **VPC na AWS usando Terraform**, trabalhando com:

- Subnets públicas e privadas  
- Internet Gateway  
- NAT Gateway  
- Tabelas de rotas  
- Endereços IP (EIP)

Ao final, você conseguirá criar uma **VPC completa com seus principais componentes** e entender como organizar o código Terraform de forma modular.

---

## Pré-requisitos

Criação de uma rede básica na AWS:

- 1 VPC  
- 2 Subnets privadas  
- 2 Subnets públicas  
- 2 NAT Gateways  
- 1 Internet Gateway  

---

## Conceitos importantes

### Built-in Functions
Funções prontas do Terraform para manipulação de **strings, listas, mapas, números e CIDR**.  
Exemplos:

- `cidrsubnet()` → calcula CIDR automaticamente  
- `lower()` → converte string em minúsculas  
- `format()` → formata strings dinamicamente  

### Meta-Arguments
Controlam o **comportamento dos recursos** e permitem criar múltiplos recursos dinamicamente.  
Exemplos:

- `for_each` → cria recursos a partir de listas ou mapas  
- `depends_on` → define ordem de criação de recursos

---

## Estrutura do Projeto

O projeto foi organizado em módulos para facilitar manutenção:

├── main.tf
├── variables.tf
├── locals.tf
├── vpc.tf
└── README.md



---

## main.tf

Define o provedor AWS e versões:

```hcl
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

---
## variables.tf
Variáveis utilizadas para região, CIDR da VPC e tamanho das subnets:
```variable "region" {
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
## locals.tf
Lista de nomes das subnets públicas e privadas para criação dinâmica:
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
## vpc.tf
Criação da VPC e todos os recursos associados (subnets, gateways, tabelas de rotas, EIPs e NAT Gateways):

```
# Criando VPC
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr_block
  tags = { Name = "vpc-project-elven", terraformed = "true" }
}

# Subnets Públicas
resource "aws_subnet" "public" {
  for_each = { for idx, name in local.subnet_public : idx => name }
  vpc_id = aws_vpc.main.id
  cidr_block = cidrsubnet(var.vpc_cidr_block, 8, each.key)
  map_public_ip_on_launch = true
  tags = { Name = lower(format("subnet-%s", each.value)) }
}

# Subnets Privadas
resource "aws_subnet" "priv" {
  for_each = { for idx, name in local.subnet_priv : idx => name }
  vpc_id = aws_vpc.main.id
  cidr_block = cidrsubnet(var.vpc_cidr_block, 8, each.key + length(local.subnet_public))
  map_public_ip_on_launch = false
  tags = { Name = lower(format("subnet-%s", each.value)) }
}

# Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags = { Name = "igw", terraformed = "true" }
}

# Tabela de Rotas Públicas
resource "aws_route_table" "public" { vpc_id = aws_vpc.main.id }

resource "aws_route" "internet_access" {
  route_table_id = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id = aws_internet_gateway.igw.id
}

resource "aws_route_table_association" "public" {
  for_each = aws_subnet.public
  subnet_id = each.value.id
  route_table_id = aws_route_table.public.id
}

# EIP para NAT Gateways
resource "aws_eip" "nat" {
  for_each = aws_subnet.public
  domain = "vpc"
  tags = { Name = "nat-eip-${each.key}", terraformed = "true" }
}

# NAT Gateways
resource "aws_nat_gateway" "ngw" {
  for_each = aws_subnet.public
  subnet_id = each.value.id
  allocation_id = aws_eip.nat[each.key].id
  tags = { Name = "nat-gw-${each.key}", terraformed = "true" }
  depends_on = [aws_internet_gateway.igw]
}

# Tabelas de Rotas Privadas
resource "aws_route_table" "priv" {
  for_each = aws_subnet.public
  vpc_id = aws_vpc.main.id
  tags = { Name = "rtb-priv-${each.key}", terraformed = "true" }
}

# Rotas Privadas
resource "aws_route" "priv" {
  for_each = aws_route_table.priv
  route_table_id = each.value.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id = aws_nat_gateway.ngw[each.key].id
}

# Associações de Subnets Privadas às Tabelas de Rotas Privadas
resource "aws_route_table_association" "priv" {
  for_each = aws_subnet.priv
  subnet_id = each.value.id
  route_table_id = aws_route_table.priv[each.key % length(aws_nat_gateway.ngw)].id
}

```
---

## Conclusão
Este laboratório faz parte do Programa SRE Advanced da Elven Works.
O código foi desenvolvido seguindo boas práticas:
Dinâmico usando Built-in Functions
Reutilizável com Meta-Arguments
Previsível e auditável com Terraform State
Repositório oficial da Elven para referência: https://github.com/Formacao-SRE/minha-primeira-vpc-com-terraform
Meu repositório com o código deste laboratório: https://gitlab.com/desafio-elven-programadeformacao/terraform-vpc

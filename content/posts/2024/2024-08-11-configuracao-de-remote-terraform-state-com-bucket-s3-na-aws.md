---
title: "Configuração de Remote Terraform State com Bucket S3 na AWS"
description: "Como configurar o Terraform state remoto em um bucket S3 na AWS em vez de mantê-lo localmente."
image: "/images/terraform.jpg"
author: Jason
date: 2024-08-11
tags: [ "IaC", "Cloud", "AWS", "S3" ]
type: post
showTableOfContents: true
---

![terraform](/images/terraform.jpg)

O objetivo desse post é fazer a configuração de um remote terraform
state usando um bucket s3 em vez de deixar seu terraform state
localmente. E também criar algum recurso na aws para testar o
funcionamento do terraform.

## Pré-Requisitos 

Para continuar a configuração do seu remote terraform state serão
necessários os seguintes pré-requitios abaixo:

-   Uma conta na AWS
-   Um Usuário IAM
-   AWS CLI instalado.
-   Uma conta no github
-   Um bucket S3
-   CLI do terraform instalado

## Criação Usuário IAM 

Nesse primeiro passo vamos fazer a criação de um usuário IAM para usar
nos arquivos de configurações do terraform.

Para fazer a criação do usuário IAM seguir esse outro
[artigo](/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam/)
que escrevi há um tempo atrás.

Para a nossa configuração vamos anexar a policy **AdmistratorAccess** no
nosso usuário **blog** conforme imagem abaixo:

![terraform-2](/images/terraform-2.jpg)

Foi anexada essa policy pois é apenas um lab para ambientes produtivos
devem ser colocadas as devidas permissões.

## Criação bucket S3 

Nesse próximo passo vamos criar o nosso bucket S3 que terá a função de
armazenar o nosso terraform state. Para isso devemos ir na console de
administração da AWS procurar pelo serviço S3 e clicar em **Create
Bucket** deve ser apresentado a tela abaixo:

![terraform-3](/images/terraform-3.jpg)

No meu caso vou colocar o nome do bucket de terraform-blog-jason e outro
detalhe é que esse bucket será criado na região us-east-1.

Devemos deixar habilitado a opção **Block all public access** conforme
imagem abaixo:

![terraform-4](/images/terraform-4.jpg)

Depois disso vamos clicar no botão **Create bucket** no final da tela.
Com isso o nosso bucket será criado conforme imagem abaixo:

![terraform-5](/images/terraform-5.jpg)

## Criação repositório Github 

Agora vamos criar um repositório no Github para armazenar os nossos
arquivos de configurações do terraform conforme abaixo:

![terraform-6](/images/terraform-6.jpg)

Na opção Add .gitignore adicionar o template do Terraform. Segue o link
do [repositório](https://github.com/jjasonhenrique/terraform-blog-jason)
criado.

Agora vamos fazer um clone desse repositório usando o comando abaixo:

```bash
git clone https://github.com/jjasonhenrique/terraform-blog-jason.git
```

## Criando os arquivos de configurações 

Agora vamos fazer a criação dos arquivos de configurações do terraform.
Como pré-requisito o usuário IAM já deve estar configurado via AWS CLI
conforme o artigo que escrevi anteriormente e que deixei o link no
começo desse artigo.

O primeiro arquivo que vamos criar é o **versions.tf** conforme abaixo:

```bash
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

Em seguida vamos fazer a criação do arquivo **provider.tf** conforme abaixo:

```bash
provider "aws" {
  shared_config_files      = ["/home/jason/.aws/config"]
  shared_credentials_files = ["/home/jason/.aws/credentials"]
  profile                  = "blog"
}
```

Nesse arquivo passamos o usuário que será utilizado para a conexão com a
AWS.

Agora vamos fazer a criação do arquivo **main.tf** conforme abaixo:

```bash
terraform {
  backend "s3" {
    bucket  = "terraform-blog-jason"
    key     = "terraform.tfstate"
    region  = "us-east-1"
    profile = "blog"
  }
}
```

No arquivo acima estamos fazendo a configuração de um bucket S3 na AWS
para que seja usado como remote terraform state.

Agora vamos criar um recurso na AWS criando o arquivo **ecs.tf** conforme
abaixo:

```bash
resource "aws_ecs_cluster" "teste-blog" {
  name = "teste-blog"

  setting {
    name  = "containerInsights"
    value = "disabled"
  }
}
```

Lembrando que para a criação de um cluster ECS na AWS existem mais
configurações a serem feitas. É apenas um teste para validar o
funcionamento do remote terraform state.

Após a criação dos arquivos você deve commitar eles excutando os
comandos abaixo:

```bash
git add .
git commit -m "Criação Cluster ECS"
git push
```

## Validação 

Agora vamos validar o remote terraform state e a criação de um recurso
na AWS. Primeiramente vamos inicializar o nosso terraform state
executando o comando abaixo:

```bash
terraform init
```

A saída do comando deve ser similar a abaixo:

![terraform-7](/images/terraform-7.jpg)

Agora vamos executar o comando terraform plan para validar o que será
criado na AWS. A saída deve ser similar a abaixo:

![terraform-8](/images/terraform-8.jpg)

Agora vamos executar o comando terraform apply -auto-approve para a
criação do recurso na AWS.

Para validar vamos no serviço ECS e verificar se o cluster ECS foi
criado. Deve ter criado conforme imagem abaixo:

![terraform-9](/images/terraform-9.jpg)

Também vamos validar se foi criado um arquivo terraform.tfstate no
bucket conforme imagem abaixo:

![terraform-10](/images/terraform-10.jpg)

Para destruir o recurso criado na AWS para esse teste executar o comando
abaixo:

```bash
terraform destroy --auto-approve
```

## Conclusão 

O Objetivo desse artigo foi demonstar como deve ser feito a configuração
de um remote terraform state usando um bucket S3 na AWS.


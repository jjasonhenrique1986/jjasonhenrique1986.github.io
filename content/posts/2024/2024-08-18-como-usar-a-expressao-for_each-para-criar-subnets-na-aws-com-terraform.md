---
title: Como usar a expressão for_each para criar subnets na aws com terraform
description: "Como usar a expressão for_each no Terraform para reduzir código ao criar subnets na AWS."
image: "/images/terraform.jpg"
author: Jason
date: 2024-08-18
tags: [ "IaC", "Cloud", "AWS" ]
type: post
showTableOfContents: true
---

![terraform](/images/terraform.jpg)

O Objetivo desse artigo é demonstrar a utilizacão da expressão for_each
com o intuito de reduzir a quantidade de código para a criacão de sua
infraestrutura no terraform. No nosso exemplo vamos otimizar a criação
de subntes na AWS.

## Pré Requisitos 

Abaixo segue os pré requisitos para acompnhar esse artigo:

-   Ter uma conta na AWS.
-   Ter o cli do terraform instalado.
-   Ter uma conta no github.
-   Já ter um repositório no github configurado para usar o terraform.
    Pode usar como exemplo esse
    [repositório](https://github.com/jjasonhenrique/terraform-blog-jason).

## O que é for_each ? 

O **`for_each`** é uma expressão que permite iterar sobre uma coleção de
itens para criar recursos no Terraform. Ele é útil quando você precisa
criar múltiplos recursos que compartilham uma estrutura comum, mas que
possuem algumas variações por exemplo o endereço cidr de cada subnet e a
zona de disponibilidade (AZ).

## Criação de uma VPC 

Para a criação das subnets é necessário a criaçºao de uma VPC na AWS.
Você deverá criar um arquivo chamado **vpc.tf** com o código abaixo:

``` 
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "main-vpc"
  }
}
```

## Criação das subnets 

Nesse próximo passo vamos imaginar que precisamos criar múltiplas
subnets em diferentes zonas de disponibilidade. Para fazer isso vamos
usar a expressão for_each para que não haja repetição de código.

Agora vamos declarar a variável subnets no arquivo **variables.tf**
conforme abaixo:

``` 
variable "subnets" {
  type         = map(any)
  default      = {}
  description  = "Subnets para criacao na AWS"
}
```

Agora vamos fazer a declaracao da variável no arquivo
**terraform.tfvars** conforme abaixo:

``` 
subnets = {
    subnet1 = {
        az  = "us-east-1a"
        cidr = "10.0.1.0/24"
    },
    subnet2 = {
        az  = "us-east-1b"
        cidr = "10.0.2.0/24"
    },
    subnet3 = {
        az  = "us-east-1c"
        cidr = "10.0.3.0/24"
    }
}
```

Agora vamos criar o recurso aws_subnet usando a expressao for_each para
que possamos iterar no map que foi declarado acima. Deve ser criado um
arquivo chamado **subnets.tf** com o código abaixo:

``` 
resource "aws_subnet" "subnets" {
  for_each = var.subnets

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value["cidr"]
  availability_zone = each.value["az"]

  tags = {
    Name = each.key
  }
}
```

Usamos o `for_each` para iterar sobre o mapa `subnets`. Para cada item
no mapa uma subnet é criada na VPC. Cada subnet recebe um CIDR e é
atribuída a uma zona de disponibilidade.

Usamos o each.key para fazer a iteração na criação no nome da subnet.

Com isso não precisamos repetir o código para cada subnet que vamos
criar na AWS tornando assim o gerenciamento dos recursos mais eficiente.

## Validação 

Para validar basta executar os comandos abaixo:

``` 
terraform init
terraform plan -out plano
terraform apply plano
```

Deve aparecer uma imagem conforme abaixo:

![terraform-11](/images/terraform-11.jpg)

Você também pode validar na console da AWS conforme imagem abaixo:

![terraform-12](/images/terraform-12.jpg)

## Vantagens de usar o for_each 

Abaixo algumas vantagens que na minha opinião fazem sentido o uso do
for_each:

-   **Flexibilidade na Configuração**: Ao definir subnets em um mapa,
    você pode facilmente ajustar o CIDR ou a AZ sem duplicar código.
-   **Escalabilidade**: Adicionar novas subnets é simples e exige apenas
    a adição de um novo item ao mapa `subnets`, sem alterar o código
    principal do recurso.
-   **Manutenção**: Como cada subnet é identificada por uma chave única
    (`subnet1`, `subnet2`, etc.), você pode gerenciar ou atualizar
    subnets específicas sem afetar outras.

## Conclusão 

O objetivo desse artigo foi demonstrar a utilização do for_each na
criação de subnets porém pode ser usado para a criação de outros
recursos na AWS.
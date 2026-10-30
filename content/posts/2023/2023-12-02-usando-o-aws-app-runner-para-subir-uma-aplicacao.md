---
title: "Usando o AWS App Runner para subir uma aplicação"
description: "Como usar o AWS App Runner para subir uma aplicação a partir de uma imagem pública do ECR."
image: "/images/apprunner.jpg"
author: Jason
date: 2023-12-02
tags: [ "Cloud", "AWS", "Containers" ]
type: post
showTableOfContents: true
---

![apprunner](/images/apprunner.jpg)

O objetivo desse artigo é demonstrar a utilização do serviço AWS App
Runner. Vamos subir uma aplicação usando uma imagem publica do ECR para
testar o serviço.

## O que é AWS App Runner 

AWS App Runner é um serviço da AWS que permite você implantar gerenciar
aplicações web e API sem a necessidade de gerenciar toda Infraestrutura
como por exemplo LoadBalancer, Autoscaling. Permite você subir
rapidamente uma aplicação.

Para mais informações sobre esse serviço da AWS consultar esse
[link](https://aws.amazon.com/pt/apprunner/).

## Aplicação usando um ECR Público 

Vamos subir essa aplicação que está localizada em um ECR público
conforme esse
[link](https://gallery.ecr.aws/aws-containers/hello-app-runner).

Você deve acessar sua conta da AWS procurar pelo serviço **AWS App
Runner** e clicar na opção **Create an App Runner Service** será
apresentado a tela abaixo:

![apprunner-2](/images/apprunner-2.jpg)

Podemos usar o AWS App Runner executando através de um Container
Registry sendo o ECR o único suportado. Você tem a opção de usar um ECR
público ou um ECR privado. Abaixo seguem as configurações utilizadas
para subir uma aplicação usando um ECR público:

![apprunner-3](/images/apprunner-3.jpg)

Na próxima tela vamos configurar o nome do serviço, a quantidade de CPU
e memória que será utilizada, a porta e também pode ser configurado
alguma variavel de ambiente.

![apprunner-4](/images/apprunner-4.jpg)

Na próxima tela vamos configurar o Auto scaling da nossa aplicação por
ser apenas um teste vamos subir com as definições Default porém você
pode criar as suas próprias definições clicando botão create

![apprunner-5](/images/apprunner-5.jpg)

Na próxima fazer fazer as configurações de Health Check conforme abaixo:

![apprunner-6](/images/apprunner-6.jpg)

Também podemos fazer a configurações de security, networking,
observability e tags. Na próxima tela será feito um review das
configurações da sua aplicação e depois você deve clicar no botão
**Create & Deploy**

Na próxima tela podemos acompanhar o progresso do Deploy conforme
abaixo:

![apprunner-7](/images/apprunner-7.jpg)

Você também verificar os logs da aplicação conforme imagem abaixo:

![apprunner-8](/images/apprunner-8.jpg)

Para testar a aplicação basta clicar no endereço que será gerado em
Default Domain conforme imagem abaixo:

![apprunner-9](/images/apprunner-9.jpg)

Deve aparecer uma imagem conforme abaixo:

![apprunner-10](/images/apprunner-10.jpg)

## Conclusão 

O objetivo desse artigo foi demonstrar a utilização do AWS App Runner
usando o metódo de Container Registry.fazendo o deploy de uma aplicação
em um ECR público. Em um próximo artigo pretendo demonstrar a utilização
do AWS App Runner usando uma API localizada no Github.

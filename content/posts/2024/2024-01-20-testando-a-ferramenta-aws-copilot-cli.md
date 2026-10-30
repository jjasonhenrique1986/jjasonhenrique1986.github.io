---
title: "Testando a ferramenta AWS Copilot CLI"
description: "Testando o AWS Copilot CLI para criar de forma simplificada uma aplicação conteinerizada executando no ECS."
image: "/images/copilot.jpg"
author: Jason
date: 2024-01-20
tags: [ "Cloud", "AWS", "Containers", "ECS" ]
type: post
showTableOfContents: true
---

![copilot](/images/copilot.jpg)

Nesse artigo vou demonstar a utilização da ferramenta AWS Copilot CLI
para a criação de uma forma simplificada uma aplicação conteinerizada
executando no ECS. Com o AWS Copilot CLI também é possível gerenciar
toda a infraestrutura da aplicação como por exemplo a Infraestrutura
como Código, Armazenamento e os segredos armazenados além da pipeline da
aplicação tudo via linha de comando.

Para mais informações sobre o AWS Copilot CLI acessar esse
[link](https://aws.github.io/copilot-cli/docs/overview/).

## Pré Requisitos 

Abaixo os pré requisitos para seguir com esse tutorial:

-   Ter uma conta AWS
-   AWS Copilot CLI instalado no seu computador
-   Ter um usuário IAM configurado na sua máquina

## Instalação 

Para efetuar a instalação do AWS Copilot deve ser executado os seguintes
comandos abaixo:

``` bash
sudo curl -Lo /usr/local/bin/copilot https://github.com/aws/copilot-cli/releases/latest/download/copilot-linux \
   && sudo chmod +x /usr/local/bin/copilot \
   && copilot --help
```

Deve ser apresentado uma tela similar a abaixo:

![copilot-2](/images/copilot-2.jpg)

No meu caso eu utilizei um computador executando linux. Para outros
sistemas operacionais consultar esse
[link](https://aws.github.io/copilot-cli/docs/getting-started/install/).

## Configuração usuário AWS IAM 

Como pré requisito para utilização do AWS Copilot CLI é necessário a
configuração de usuário IAM no seu computador. Para fazer essa
configuração pode seguir esse
[artigo](/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam/).

\
Para esse teste vou usar a policy **AdministratorAccess**. Caso utilize
um profile diferente do default no arquivo .aws/credentials será
necessário exportar a variável de ambiente **AWS_PROFILE** ou inserir
essa variável no arquivo .bashrc ou .zshrc. No meu caso vou exportar a
variável executando o comando abaixo:

``` bash
export AWS_PROFILE="blog"
```

Para verificar o conteúdo da variável executar o comando abaixo:

``` bash
echo $AWS_PROFILE
```

## Deploy da Aplicação 

Para testar o AWS copilot CLI vamos fazer o deploy da aplicação que está
nesse
[repositório](https://github.com/aws-samples/amazon-ecs-cli-sample-app.git).
Primeiro passo que necessitamos fazer é o git clone desse repositório
usando o comando abaixo:

``` bash
git clone https://github.com/aws-samples/amazon-ecs-cli-sample-app.git example 
```

Depois vamos acessar a pasta example e vamos executar o comando
**copilot init** será apresentado a tela abaixo:

![copilot-3](/images/copilot-3.jpg)

Você deve inserir o nome da aplicação. No meu caso será teste-app-blog.

Na próxima tela será solicitado a escolha do tipo de workload da sua
aplicação. Para essa demonstração iremos utilizar a opção **Load
Balanced Web Service** mas existem outras opções conforme abaixo:

![copilot-4](/images/copilot-4.jpg)

Agora você deve escolher um nome para seu serviço. No meu caso será
teste-app-blog-svc conforme abaixo:

![copilot-5](/images/copilot-5.jpg)

Agora será solicitado a localização do arquivo Dockerfile ou a
localização de uma imagem da aplicação. Vamos utilizar o arquivo
Dockerfile que está na raiz do repositório.

![copilot-6](/images/copilot-6.jpg)

Nesse próximo passo será criado todas as roles de infraestrutura
necessárias para o funcionamento da nossa aplicação. Após essa criação
será perguntado se você gostaria de fazer deploy da sua aplicação
conforme imagem abaixo:

![copilot-7](/images/copilot-7.jpg)

Na próxima tela será solicitado a criação de um environment. Esse
environment terá o nome de dev conforme imagem abaixo:

![copilot-8](/images/copilot-8.jpg)

Na próxima tela será criado toda a parte de infraestrutura como por
exemplo repositório do ECR, um bucket S3, toda parte de network e o seu
cluster ECS.

![copilot-9](/images/copilot-9.jpg)

Na próxima tela o AWS copilot CLI irá fazer o build da imagem da nossa
aplicação e fazer o push para o repositório ECR criado anteriormente
conforme abaixo:

![copilot-10](/images/copilot-10.jpg)

Na próxima tela a aplicação será configurada dentro do cluster ECS, será
feito o deploy e será disponibilizado a URL da nossa aplicação conforme
image abaixo:

![copilot-11](/images/copilot-11.jpg)

Acessando a URL da aplicação deve retornar uma página similar a abaixo:

![copilot-12](/images/copilot-12.jpg)

## Interagindo com AWS Copilot CLI 

Com a aplicação já deployada no seu cluster ECS podemos usar o AWS
Copilot CLI para consultar informações da nossa aplicação. Abaixo segue
alguns exemplos:

``` bash
copilot svc status
```

![copilot-13](/images/copilot-13.jpg)

``` bash
copilot svc ls
```

![copilot-14](/images/copilot-14.jpg)

``` bash
copilot app show
```

![copilot-15](/images/copilot-15.jpg)

Existem muitas outras opções que você pode utilizar com o AWS copilot
CLI.

## Deletando sua aplicação 

Agora vamos deletar a aplicação criada usando o comando abaixo:

``` bash
copilot app delete
```

Será solicitado uma confirmação e depois será feito processo de exclusão
de todos os recursos criados pelo AWS copilot CLI conforme imagem
abaixo:

![copilot-16](/images/copilot-16.jpg)

## Conclusão 

O Objetivo desse artigo foi demonstrar a utilização da ferramenta AWS
Copilot CLI implementando toda a infraestrutura e uma aplicação de
teste.
---
title: "Criando e Configurando uma role IAM"
description: "Tutorial para criar e configurar uma role IAM na AWS, uma forma mais segura de acesso do que usar um usuário IAM."
image: "/images/iam.jpg"
author: Jason
date: 2024-03-30
tags: [ "Cloud", "AWS", "Security" ]
type: post
showTableOfContents: true
---

![iam](/images/iam.jpg)

O objetivo desse artigo é criar um tutorial demonstrando como criar e
configurar uma role IAM na AWS para que você possa interagir de forma
programatica com a console de administração da AWS. Usando uma role IAM
é uma forma mais segura do que usando um usuário IAM conforme demonstrei
nesse
[post](/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam/).

## Pré Requisitos 

Abaixo os pré requisitos para seguir esse tutorial:

-   Conta AWS
-   Uma máquina Linux
-   Um usuário IAM sem nenhuma permissão
-   AWS CLI instalado

## Criação usuário IAM 

Como primeiro passo você deve acessar sua conta AWS e acessar o serviço
**IAM** e ir na opção **Users** e depois clicar na opção **Create User**
será apresentado a seguinte tela abaixo:

![iam-14](/images/iam-14.jpg)

Não deve ser habiltado a opção **Provide user access to the AWS
Management Console** pois será habilitado acesso a console de
administração da AWS.

Deve ser inserido um nome para nosso usuário IAM e clicar em **Next**.

Na próxima tela não deve ser inserido nenhuma permissão a esse usuário
IAM a permissão será concedida através da Role IAM usando uma política
IAM.

Depois disso clicar na opção **Create User** e seu usuário será criado.

Para como gerar o **Access Key** e o **Secret Access Key** e como
configurar você pode consultar esse outro
[post](/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam/)
que escrevi.

## Criando uma Role IAM

Para demonstrar a utilização de uma Role iremos criar duas Roles IAM
conforme abaixo:

-   Uma Role IAM com permissão de listar buckets no S3
-   Uma Role IAM com permissão para parar, inicializar e reiniciar uma
    instancia EC2

Antes de criar as roles vamos criar duas politicas no IAM para as roles
IAM acima.

Para efetuar a criação você deve acessar a console de administração da
AWS ir no serviço **IAM** e depois em **Políticas** e então clicar no
botão **Criar política**. Apos clicar no botão **Criar política** será
apresentado a tela abaixo:

![iam-15](/images/iam-15.jpg)

Do lado direito da tela acima clicar na opção JSON e substituir o
conteúdo conforme abaixo:

``` bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "s3:ListAllMyBuckets",
            "Resource": "*"
        }
    ]
}
```

Clicar no botão Próximo e criar um nome para a política no meu caso irei
colocar o nome de **s3-list** e depois clicar no botão **Criar
política**.

Deverá ser criado outra política com o conteúdo abaixo com o nome de
**ec2-reboot**.

``` bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "ec2:RebootInstances",
                "ec2:StartInstances",
                "ec2:StopInstances"
            ],
            "Resource": "*"
        }
    ]
}
```

Agora vamos criar duas roles uma para cada política criada. Para isso
você deve ir no serviço **IAM** e depois na opção **Funções** e então
clicar no botão **Criar perfil**. Deverá aparecer a tela abaixo:

![iam-16](/images/iam-16.jpg)

Você deve selecionar a opção **Conta da AWS** e clicar em Próximo. Após
isso selecionar a policy **s3-list** criada anteriormente e clicar em
Próximo. Será apresentado a tela abaixo:

![iam-17](/images/iam-17.jpg)

Deve ser inserido o nome do perfil como **s3** e clicar no botão **Criar
perfil**. Agora vamos precisar editar o conteúdo da política de
confiança do perfil criado na tela abaixo:

![iam-18](/images/iam-18.jpg)

Agora vamos substituir o conteúdo da política de confiança conforme
abaixo:

``` bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::id_conta:user/blog"
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}
```

Onde id_conta é o número da sua conta AWS. Com isso finalizamos a
configuração da role **s3**. Você deve fazer o mesmo procedimento para
role **ec2** e inserir a política **ec2-reboot** nessa role.

## Configurando as roles na CLI 

Como pré-requisitio o usuário criado deve estar configurado na sua
máquina usando o aws cli. Lembrando que esse usuário não pode ter nenhum tipo de acesso na AWS.

Agora vamos configurar o acesso desse usuário nas roles **s3** e
**ec2**. Deverá ser adicionado no arquivo .aws/config os seguintes
profiles abaixo:

``` bash
[profile s3]
role_arn = arn:aws:iam::id_conta:role/s3
source_profile = blog

[profile ec2]
role_arn = arn:aws:iam::id_conta:role/ec2
source_profile = blog
```

Onde id_conta é o número da sua conta da AWS.

## Testando as roles 

Agora vamos testar as roles criadas. Para esses testes criei um bucket
s3 e uma instância ec2. Primeiramente vamos verificar se o usuário
criado tem alguma permissão no meu caso o nome do usuário IAM criado se
chama blog.

![iam-19](/images/iam-19.jpg)

Deve ser apresentado os erros acima para o usuário blog. Agora vamos
testar a role s3 usando o comando abaixo:

``` bash
aws s3 ls --profile s3
```

Deve ser apresentado a tela abaixo listando um bucket s3:

![iam-20](/images/iam-20.jpg)

Agora vamos para o segundo teste onde vamos parar uma instância ec2
usando a role ec2. Para isso vamos executar o comando abaixo:

``` bash
aws ec2 stop-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile ec2
```

Executando esse comando acima aparecerá uma tela de mudança de estado da
instância ec2 de running para stopping conforme imagem abaixo:

![iam-21](/images/iam-21.jpg)

Assim que a instância parar você pode executar o comando abaixo para
inicializar a mesma instância ec2.

``` bash
aws ec2 start-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile ec2
```

Você também pode testar o desligamento da instância ec2 executando o
comando abaixo. Será apresentado uma tela de erro pois essa role não tem
essa permissão.

``` bash
aws ec2 terminate-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile ec2
```

Por fim para garantir que a role s3 não consegue iniciar e parar uma
instância ec2 executar um dos comandos abaixo:

``` bash
aws ec2 stop-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile s3

aws ec2 start-instances --region sa-east-1 --instance-ids  i-0085922c8e050b223 --profile s3
```

## Conclusão 

O objetivo desse artigo foi demonstar a criação de uma role e efetuar
testes usando as roles em vez de utilizar o usuário IAM. Nesse artigo
demonstrei a utilização de uma role usando apenas uma conta AWS porém é
possível utilizar multiplas contas conforme essa
[documentação](https://docs.aws.amazon.com/pt_br/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html)
da AWS.
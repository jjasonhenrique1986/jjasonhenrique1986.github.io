---
title: "Criando e Configurando um usuário IAM"
description: "Tutorial para criar e configurar um usuário IAM com Access Key e Secret Access Key e instalar o AWS CLI."
image: "/images/iam.jpg"
author: Jason
date: 2024-01-07
tags: [ "Cloud", "AWS", "Security" ]
type: post
showTableOfContents: true
---

![iam](/images/iam.jpg)

O objetivo desse artigo é criar um tutorial demonstrando como criar e
configurar um usuário IAM na AWS para que você possa interagir de forma
programatica usando um **Access Key** e um **Secret Access Key**.

Também será demosntrado a instalação do awscli para o gereciamento dos
seus serviços na AWS.

## Pré Requisitos 

Abaixo os pré requisitos para seguir esse tutorial:

-   Conta AWS
-   Uma máquina Linux

## Criação usuário IAM 

Como primeiro passo você deve acessar sua conta AWS e acessar o serviço
**IAM** e ir na opção **Users** e depois clicar na opção **Create User**
será apresentado a seguinte tela abaixo:

![iam-2](/images/iam-2.jpg)

Não deve ser habiltado a opção **Provide user access to the AWS
Management Console** pois será habilitado acesso a console de
administração da AWS.

Deve ser inserido um nome para nosso usuário IAM e clicar em **Next**.

Na próxima tela vamos inserir as permissões para esse usuário IAM. No
meu caso vou inserir uma policy com permissão total no S3 conforme tela
abaixo:

![iam-3](/images/iam-3.jpg)

Lembrando que você pode customizar essa policy.

Depois disso vamos clicar em Next e será apresentado um Review da
criação do seu usuário conforme abaixo:

![iam-4](/images/iam-4.jpg)

Depois disso clicar na opção **Create User** e seu usuário será
criado.Seu usuário deve ser apresentado conforme tela abaixo:

![iam-5](/images/iam-5.jpg)

## Criando a Acess Key do seu usuário 

Agora vamos criar clicando sobre seu usuário IAM indo na opção Security
credentials e depois clicando em Create access key conforme imagem
abaixo:

![iam-6](/images/iam-6.jpg)

Depois da tela você deve escolher a primeira opção **Command Line
Interface (CLI)** e habilitar a opção **I understand the above
recommendation and want to proceed to create an access key** e clicar em
Next conforme imagem abaixo:

![iam-7](/images/iam-7.jpg)

Na próxima tela você pode configurar uma descrição para sua access key e
depois clicar em **Create access key** conforme imagem abaixo:

![iam-8](/images/iam-8.jpg)

Apos clicar no botão **Create access key** será gerado o **Acces key** e
o **Secret access key** conforme imagem abaixo:

![iam-9](/images/iam-9.jpg)

Lembrando que o **Secret access key** só estará disponivel nesse
momento. É recomendavel fazer o download do arquivo cvs com os valores
do **Access key** e do **Secret access key** e depois clicar em **Done**
e retornará para a tela inicial conforme abaixo:

![iam-10](/images/iam-10.jpg)

Nesse
[link](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html#securing_access-keys)
você tem boas práticas para o gerenciamento de AWS access keys na AWS.

## Instalação AWS CLI 

Agora vamos fazer a instalação do aws cli para podemos configurar o
**Access key** e o **Secret access key** gerados no passo anterior.
Abaixo os comandos para instalação. Para mais informações consultar esse
[link](https://docs.aws.amazon.com/pt_br/cli/latest/userguide/getting-started-install.html).

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Para verificar se a instalação ocorreu com sucesso excutar o comando
abaixo:

![iam-11](/images/iam-11.jpg)

## Configurando o seu Access Key 

Agora vamos fazer a configuração do seu Access key e do seu Secret
access key no seu computador. Você deve executar o comando aws
configure conforme abaixo. No meu caso vou inserir a flag --profile para
gerar um novo profile dentro do arquivo .aws/credentials

![iam-12](/images/iam-12.jpg)

## Testando o seu access Key {#testando-o-seu-access-key .wp-block-heading .has-x-large-font-size}

Agora vamos testar o acesso via CLI tentando listar os buckets S3 na sua
conta executando o comando abaixo:

```bash
aws s3 ls --profile blog
```

Deverá retornar o seguinte resultado abaixo:

![iam-13](/images/iam-13.jpg)

Lembrando que esse Access key e Secret access key tem acesso somente ao
S3. Caso tente executar o comando **aws ec2 describe-availability-zones
--profile blog** deverá retornar o seguinte erro abaixo por falta de
permissão:

``` bash
An error occurred (UnauthorizedOperation) when calling the DescribeAvailabilityZones operation: You are not authorized to perform this operation. User: arn:aws:iam::id_conta:user/blog is not authorized to perform: ec2:DescribeAvailabilityZones because no identity-based policy allows the ec2:DescribeAvailabilityZones action
```

## Conclusão 

O objetivo desse artigo foi a criação de um tutorial para a criação e
configuração de um usuário IAM usando o Access key e o Secret access key
para interagir via CLI com a sua conta AWS.

Lembrando que o Access key e o Secret access key não estará disponivel
para uso. Você deve gerar os seus próprios valores na sua conta caso
queira seguir com esse artigo.
---
title: "Configurando OpenID Connect entre AWS e GitLab"
description: "Como configurar OpenID Connect entre a AWS e o GitLab para executar pipelines usando uma role IAM, sem chaves de acesso fixas."
image: "/images/gitlab.jpg"
author: Jason
date: 2023-07-22
tags: [ "Cloud", "AWS", "CI/CD" ]
type: post
showTableOfContents: true
---

![gitlab](/images/gitlab.jpg)

O objetivo desse post é demonstrar como configurar o OpenID Connect
entre a sua conta AWS e sua conta do GitLab para executar uma pipeline
usando uma role da sua conta AWS.

Primeiramente o OpenID Connect (OIDC) permite você recupere tokens de
curta duração para se comunicar com seu Cloud Provider nesse caso vamos
utilizar a AWS mas também pode ser utilizado para Azure e GCP.

A grande vantagem dessa integração entre AWS e GitLab usando OpenID
Connect é a segurança já que é gerado um token de curta duração e não
será necessário a criação de um **Access Key** e de um **Secret Access
Key**. Será utilizado o conceito de role da AWS e essa role será
executada apenas em um determinado repositório do GitLab.

## Pré-Requisitos

Para prosseguir com esse tutorial é necessário ter os seguintes
pré-requisitos:

-   Conta na AWS;
-   Conta no GitLab SaaS utilizando esse
    [link](https://about.gitlab.com/);

## Configuração OIDC

No primeiro passo vamos configurar o OIDC na AWS mas especificamente no
IAM. Você deve ir no serviço **IAM** e ir na opção **Identity
providers** deve mostrar a tela abaixo:

![gitlab-2](/images/gitlab-2.jpg)

Clicando na opção **Add provider** será informado a tela abaixo onde
vamos iniciar a configuração do OIDC:

![gitlab-3](/images/gitlab-3.jpg)

Deve ser selecionado a opção OpenID Connect. Nos campos **Provider URL**
e **Audience** devem ser configurado a URL <https://gitlab.com> conforme
imagem abaixo:

![gitlab-4](/images/gitlab-4.jpg)

Você deve clicar na opção **Get thumbprint** onde será gerado um
certificado thumbprint que irá validar a CA do OIDC Provider no nosso
caso do GitLab e depois clicar na opção **Add provider** e com isso
finalizamos a configuração do OIDC. Será criado um provider conforme
imagem abaixo:

![gitlab-5](/images/gitlab-5.jpg)

## Criação Role AWS

No segundo passo vamos criar uma role na AWS indo na opção **IAM** e
depois **Roles** (Funções em Português)e depois na opção **Create Role**
deverá mostrar uma tela similar a abaixo:

![gitlab-6](/images/gitlab-6.jpg)

Deverá selecionar a opção **Web Identity** e selecionar o **Identity
provider** e o **Audience** criado no passo anterior conforme abaixo:

![gitlab-7](/images/gitlab-7.jpg)

Depois vamos para o step 2 onde vamos selecionar as permissões dessa
role. Nesse caso vamos usar a politica
[AmazonS3ReadOnlyAccess](https://us-east-1.console.aws.amazon.com/iam/home#/policies/arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess) para somente listar os buckets s3 de uma conta na AWS.

Depois vamos para step 3 onde vamos apenas colocar o nome da Role e
clicar na opção **Create role**.

Depois da criação da Role vamos editar a relação de confiança dessa role
para era ser menos permissiva. Abaixo segue a trust policy dessa role:

``` bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::aws_account_id:oidc-provider/gitlab.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "gitlab.com:sub": "project_path:user_gitlab/name_repository:ref_type:branch:ref:main"
                }
            }
        }
    ]
}
```

Onde:

-   **aws_account_id** é o id da sua conta da AWS;
-   **user_gitlab** é o nome do usuário utilizado no gitlab. Também
    aceita como opção um projeto do GitLab;
-   **name_repository** é o nome do repository criado no gitlab;

## Configuração GitLab

No terceiro passo vamos criar um repositório no gitlab, criar uma
pipeline simples que faça o assumerole na role que criamos no passo
anterior e que liste os buckets S3 da conta AWS

Criar um repositório no gitlab e adicionar a seguinte variável de
ambiente indo na opção Settings \>\> CI/CD \>\> Variables. Deverá
mostrar uma tela similar a abaixo:

![gitlab-8](/images/gitlab-8.jpg)

onde:

-   **aws_account_id** é o id da sua conta da AWS;
-   **name_role** é o nome da role criada na AWS;

Esse valor também pode ser pego diretamente na role criado na AWS no
campo ARN.

Após isso criar o arquivo **.gitlab-ci.yml** com o conteúdo abaixo:

``` bash
image:
  name: "amazon/aws-cli:latest"
  entrypoint:
    - ""

stages:
  - "assume-role"

job_assume_role:
    stage: "assume-role"
    script: 
      - >
        STS=($(aws sts assume-role-with-web-identity
        --role-arn ${ROLE_ARN}
        --role-session-name "GitLabRunner-${CI_PROJECT_ID}-${CI_PIPELINE_ID}"
        --web-identity-token ${CI_JOB_JWT_V2}
        --duration-seconds 3600
        --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]'
        --output text))
      - export AWS_ACCESS_KEY_ID="${STS[0]}"
      - export AWS_SECRET_ACCESS_KEY="${STS[1]}"
      - export AWS_SESSION_TOKEN="${STS[2]}"
      - aws sts get-caller-identity
      - aws s3 ls
```

Com isso será executado uma simples pipeline que lista os buckets S3 da
conta AWS.

## Conclusão

O Objetivo desse post foi demonstrar a configuração do OpenID Connect
entre AWS e GitLab. Esse mesmo procedimento pode ser efetuado usando
github conforme esse
[link](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services).

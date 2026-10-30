---
title: "GitHub Actions com OpenID Connect na AWS, sem access keys"
description: "Como configurar o OpenID Connect entre o GitHub Actions e a AWS para que os workflows assumam uma role IAM com credenciais temporárias, sem guardar access keys."
author: Jason
date: 2026-08-22
tags: [ "Cloud", "AWS", "CI/CD", "Security" ]
type: post
showTableOfContents: true
---

O objetivo desse artigo é demonstrar como configurar o **OpenID Connect
(OIDC)** entre o **GitHub Actions** e a sua conta AWS. Com isso os
workflows assumem uma role IAM e recebem **credenciais temporárias**, sem
precisar guardar `AWS_ACCESS_KEY_ID` e `AWS_SECRET_ACCESS_KEY` nos
secrets do repositório.

Eu já mostrei a mesma ideia com o GitLab no artigo
[Configurando OpenID Connect entre AWS e GitLab](/posts/2023/2023-07-22-configurando-openid-connect-entre-aws-e-gitlab/).
Agora vamos fazer o mesmo no GitHub.

## Pré Requisitos

-   Uma conta AWS e o AWS CLI v2 configurado com permissão para criar
    recursos no IAM.
-   Um repositório no GitHub.

Nos exemplos o ID da conta AWS é `111122223333` e o repositório é
`meu-usuario/meu-repo`. Troque pelos seus valores.

## Como funciona

1.  Quando um job do workflow começa, o GitHub pode gerar um **token
    JWT** assinado, com informações sobre a execução: repositório,
    branch, ambiente, etc.
2.  O job envia esse token para o **AWS STS** pedindo para assumir uma
    role (`AssumeRoleWithWebIdentity`).
3.  A AWS valida a assinatura do token com o **identity provider** que
    vamos cadastrar no IAM e confere as condições da **trust policy** da
    role.
4.  Se tudo estiver certo, o STS devolve credenciais temporárias que
    valem só durante o job.

A grande vantagem é que **não existe nenhuma credencial fixa** para
vazar ou rotacionar.

## Cadastrando o GitHub como identity provider

Esse passo é feito **uma vez por conta AWS**:

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

Em versões antigas do AWS CLI o parâmetro `--thumbprint-list` era
obrigatório. Hoje ele é opcional para o GitHub, porque a AWS valida o
certificado do provedor usando a própria biblioteca de CAs confiáveis.
O provedor também pode ser criado pela console em **IAM → Identity
providers → Add provider**, escolhendo **OpenID Connect**.

## Criando a role

Crie um arquivo chamado trust-github.json. A parte mais importante é a
condição no campo `sub`, que define **quais repositórios e branches**
podem assumir a role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::111122223333:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:meu-usuario/meu-repo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

Com essa condição, somente workflows rodando na branch `main` do
repositório `meu-usuario/meu-repo` conseguem assumir a role. **Nunca
deixe o `sub` como `*`**: isso permitiria que qualquer repositório do
GitHub, de qualquer pessoa, assumisse a sua role.

Alguns formatos úteis do `sub`:

| Cenário | Valor do `sub` |
|---|---|
| Branch específica | `repo:meu-usuario/meu-repo:ref:refs/heads/main` |
| Qualquer branch do repositório | `repo:meu-usuario/meu-repo:*` |
| Job com environment do GitHub | `repo:meu-usuario/meu-repo:environment:producao` |
| Pull requests | `repo:meu-usuario/meu-repo:pull_request` |

Agora crie a role. Para o exemplo vamos dar permissão de leitura no S3:

```bash
aws iam create-role \
  --role-name github-actions-deploy \
  --assume-role-policy-document file://trust-github.json

aws iam attach-role-policy \
  --role-name github-actions-deploy \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

Em produção, crie uma policy própria só com as permissões que o
pipeline realmente usa, seguindo o princípio do menor privilégio. Se
quiser relembrar como funcionam as roles, veja o artigo sobre
[como criar e configurar uma role IAM](/posts/2024/2024-03-30-criando-e-configurando-uma-role-iam/).

## Criando o workflow

No repositório, crie o arquivo `.github/workflows/aws.yml`:

```yaml
name: Teste OIDC AWS

on:
  push:
    branches: [ main ]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  aws:
    runs-on: ubuntu-latest
    steps:
      - name: Configurar credenciais da AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111122223333:role/github-actions-deploy
          aws-region: us-east-1

      - name: Quem sou eu?
        run: aws sts get-caller-identity

      - name: Listar buckets
        run: aws s3 ls
```

A permissão `id-token: write` é o que autoriza o job a pedir o token
OIDC ao GitHub. **Sem ela o passo de credenciais falha.** A action
oficial `aws-actions/configure-aws-credentials` faz a troca do token
pelas credenciais e exporta as variáveis de ambiente para os próximos
passos.

Faça um push na `main` (ou rode manualmente pelo `workflow_dispatch`) e
veja o log do job. O passo **Quem sou eu?** deve mostrar um ARN do tipo
`assumed-role/github-actions-deploy/...`.

## Problemas comuns

-   **`Not authorized to perform sts:AssumeRoleWithWebIdentity`**:
    normalmente o `sub` da trust policy não bate com o repositório ou a
    branch que rodou o workflow. Confira maiúsculas e minúsculas no nome
    do usuário e do repositório.
-   **`Credentials could not be loaded`** ou erro pedindo o token: falta
    a permissão `id-token: write` no workflow ou no job.
-   **Workflows vindos de forks**: pull requests de forks não recebem o
    token OIDC com permissão de escrita, por segurança. Não tente
    contornar isso.

## Conclusão

Com o OIDC o pipeline acessa a AWS com credenciais temporárias e
restritas a um repositório e branch específicos. Não há access keys
guardadas no GitHub, nada para rotacionar e, se o workflow for
comprometido, as credenciais expiram sozinhas no fim do job.

## Referências

-   [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
-   [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials)
-   [Criar um provedor de identidade OIDC no IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)

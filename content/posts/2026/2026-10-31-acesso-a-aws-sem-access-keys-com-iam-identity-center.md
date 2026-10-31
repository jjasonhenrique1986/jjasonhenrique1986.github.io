---
title: "Acesso à AWS sem access keys com o IAM Identity Center"
description: "Como trocar o usuário IAM com access key fixa pelo IAM Identity Center: Organizations, permission sets, MFA e login no AWS CLI com aws sso login."
author: Jason
date: 2026-10-31
tags: [ "Cloud", "AWS", "Security" ]
type: post
showTableOfContents: true
---

O objetivo desse artigo é demonstrar como trocar o acesso à AWS por **usuário
IAM com access key fixa** pelo **IAM Identity Center**. No final, o AWS CLI
passa a usar **credenciais temporárias**, o login é feito pelo navegador com
**MFA** e não fica nenhuma chave guardada em `~/.aws/credentials`.

Há alguns anos eu mostrei como
[criar e configurar um usuário IAM](/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam/)
e depois como
[usar uma role IAM](/posts/2024/2024-03-30-criando-e-configurando-uma-role-iam/),
uma forma mais segura de acesso. O Identity Center é o passo seguinte: some o
usuário IAM e, com ele, a access key.

## Pré Requisitos

-   Uma conta AWS e acesso ao usuário root.
-   O AWS CLI v2 instalado.

Nos exemplos o ID da conta AWS é `111122223333`. Troque pelos seus valores.

## Por que sair da access key

Uma access key de usuário IAM não expira. Se ela vazar (um commit, um backup,
um notebook perdido), quem tiver a chave tem as permissões do usuário até
alguém perceber e apagar a chave.

Com o Identity Center:

-   O login é feito no navegador, com senha e MFA.
-   O AWS CLI recebe credenciais temporárias, que expiram sozinhas.
-   As permissões ficam em **permission sets**, que viram roles IAM na conta.

## Como funciona

O Identity Center tem dois tipos de instância:

-   **Instância de organização**: dá acesso às contas AWS da organização.
    É a que precisamos.
-   **Instância de conta** (*account instance*): só serve para aplicações da
    AWS, como o Amazon Q. Ela **não** dá acesso ao console nem ao AWS CLI da
    conta.

Para ter uma instância de organização, a conta precisa fazer parte do **AWS
Organizations**. Mesmo com uma conta só, ela vira a *management account* de
uma organização de uma conta.

## Passo 1: MFA no usuário root

Depois da migração, o root vira o acesso de emergência. Antes de começar, ative
o MFA nele: no console, logado como root, clique no nome da conta →
**Security credentials** → **Assign MFA device**.

Para conferir pelo AWS CLI:

```bash
aws iam get-account-summary --query 'SummaryMap.AccountMFAEnabled'
```

O resultado precisa ser `1`.

## Passo 2: criar a organização

```bash
aws organizations create-organization --feature-set ALL

aws organizations enable-aws-service-access \
  --service-principal sso.amazonaws.com
```

O segundo comando libera o Identity Center para atuar na organização.

## Passo 3: ativar o Identity Center

A instância de organização é ativada pelo console: **IAM Identity Center** →
**Enable**, escolhendo a opção com o AWS Organizations.

### Se a conta já tiver uma account instance

Foi o meu caso. A conta tinha uma *account instance* antiga e vazia, e com ela
os comandos de permission set falham:

```text
An error occurred (ValidationException) when calling the ListPermissionSets
operation: This operation is not supported for account instances of IAM
Identity Center.
```

Antes de apagar, confira se ela não tem usuários nem aplicações. Nas versões
recentes do AWS CLI:

```bash
aws sso-admin list-instances
aws sso-admin list-applications --instance-arn <instance-arn>
aws identitystore list-users --identity-store-id <identity-store-id>
```

Se estiver vazia, apague a instância e ative de novo pelo console:

```bash
aws sso-admin delete-instance --instance-arn <instance-arn>
```

### MFA no Identity Center

Em **Settings** → **Authentication**, configure o MFA para ser pedido em todo
login.

## Passo 4: usuário e permission sets

Pegue o ARN da instância e o ID do identity store:

```bash
aws sso-admin list-instances
```

Crie o seu usuário:

```bash
aws identitystore create-user \
  --identity-store-id <identity-store-id> \
  --user-name meu-usuario \
  --display-name "Meu Nome" \
  --name GivenName=Meu,FamilyName=Nome \
  --emails Value=meu-email@exemplo.com,Primary=true
```

Crie um permission set de administração com sessão de 1 hora. A duração usa o
formato ISO 8601 (`PT1H`):

```bash
aws sso-admin create-permission-set \
  --instance-arn <instance-arn> \
  --name AdministratorAccess \
  --session-duration PT1H

aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn <instance-arn> \
  --permission-set-arn <permission-set-arn> \
  --managed-policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

E atribua o permission set ao usuário na conta:

```bash
aws sso-admin create-account-assignment \
  --instance-arn <instance-arn> \
  --permission-set-arn <permission-set-arn> \
  --principal-type USER \
  --principal-id <user-id> \
  --target-type AWS_ACCOUNT \
  --target-id 111122223333
```

A AWS cria na conta uma role com o nome
`AWSReservedSSO_AdministratorAccess_<sufixo>`. Não altere essa role direto no
IAM: ela é gerenciada pelo Identity Center.

Vale criar também um permission set só de leitura (`ReadOnlyAccess`) para o
dia a dia e deixar o de administração para quando for alterar algo.

### Senha do usuário

Um usuário criado pela API não recebe o convite sozinho. No console, em
**Users** → seu usuário → **Reset password**, mande o e-mail para definir a
senha. No primeiro login, cadastre o MFA.

## Passo 5: AWS CLI com aws sso login

No `~/.aws/config`, crie uma `sso-session` e um perfil para cada permission
set:

```ini
[sso-session minha-conta]
sso_start_url = https://d-1234567890.awsapps.com/start
sso_region = us-east-1
sso_registration_scopes = sso:account:access

[profile admin]
sso_session = minha-conta
sso_account_id = 111122223333
sso_role_name = AdministratorAccess
region = us-east-1
```

A `sso_start_url` é o *AWS access portal URL*, que aparece no painel do
Identity Center.

Faça o login. O comando abre o navegador e mostra um código: confira se é o
mesmo do terminal, entre com senha e MFA e autorize o acesso.

```bash
aws sso login --sso-session minha-conta
```

Teste:

```bash
aws sts get-caller-identity --profile admin
```

O `Arn` mostra a role do permission set:

```text
arn:aws:sts::111122223333:assumed-role/AWSReservedSSO_AdministratorAccess_<sufixo>/meu-usuario
```

### E o Terraform?

O Terraform usa os perfis do `~/.aws/config`. Se precisar das credenciais em
variáveis de ambiente, o AWS CLI exporta as da sessão atual:

```bash
eval "$(aws configure export-credentials --profile admin --format env)"
```

## Passo 6: aposentar o usuário IAM

Depois de testar o login, apague o usuário IAM antigo. Um usuário só pode ser
apagado sem nada ligado a ele, então a ordem é: access keys, MFA, senha de
console, políticas e, por fim, o usuário.

```bash
aws iam delete-access-key --user-name meu-usuario-iam --access-key-id <access-key-id>
aws iam deactivate-mfa-device --user-name meu-usuario-iam --serial-number <mfa-arn>
aws iam delete-virtual-mfa-device --serial-number <mfa-arn>
aws iam delete-login-profile --user-name meu-usuario-iam
aws iam detach-user-policy --user-name meu-usuario-iam --policy-arn <policy-arn>
aws iam delete-user-policy --user-name meu-usuario-iam --policy-name <inline-policy-name>
aws iam delete-user --user-name meu-usuario-iam
```

Por último, apague o `~/.aws/credentials`.

## Um detalhe: domínio próprio para o portal

Tentei criar um CNAME do meu domínio para o endereço `awsapps.com` do portal.
Não funciona: a conexão HTTPS falha, porque o certificado do portal só vale
para `*.awsapps.com`. Para usar um domínio próprio, é preciso um
redirecionamento (por exemplo, um CloudFront que responde com 301 para o
portal). O AWS CLI continua usando o endereço `awsapps.com` na
`sso_start_url`.

## Conclusão

Com o Identity Center não existe mais access key fixa para vazar. O acesso
passa por senha e MFA, as credenciais expiram sozinhas e as permissões ficam
organizadas em permission sets.

Para pipelines, a mesma ideia vale com OIDC: veja
[GitHub Actions com OpenID Connect na AWS, sem access keys](/posts/2026/2026-08-22-github-actions-com-oidc-na-aws-sem-access-keys/).

---
title: "Revisão de PR de infraestrutura com IA, parte 1: arquitetura e acesso ao Bedrock"
description: "Primeira parte da série sobre como revisar Pull Requests de Terraform e Kubernetes com IA no Amazon Bedrock (Claude ou gpt-oss): a arquitetura do projeto, o acesso aos modelos e a role IAM com OIDC, sem access keys."
author: Jason
date: 2026-11-01
tags: [ "AWS", "CI/CD", "IA", "Terraform", "Kubernetes" ]
type: post
showTableOfContents: true
draft: false
---

Revisar Pull Request de infraestrutura cansa. Um `cidr_blocks = ["0.0.0.0/0"]`
ou um container sem limite de memória passam fácil no meio de um diff grande.

Nesta série vou montar um **revisor de PR com IA**: a cada Pull Request, um
workflow do GitHub Actions pega o diff de Terraform e Kubernetes, passa pelo
**checkov** e pede para um modelo no **Amazon Bedrock** explicar os riscos num
comentário. O padrão é o **Claude**, mas o projeto também aceita o **gpt-oss**,
da OpenAI, e foi com ele que fiz os testes desta série (explico o motivo abaixo).

A série tem três partes:

1. **Arquitetura e acesso ao Bedrock** (este post)
2. O script em Python que conversa com o modelo
3. O workflow do GitHub Actions e os cuidados de segurança

O código completo está no repositório
[iac-ai-reviewer](https://github.com/jjasonhenrique1986/iac-ai-reviewer).

## Como funciona

1. Um PR altera arquivos `.tf` ou `.yaml`.
2. O workflow roda o **checkov**, que faz as checagens **determinísticas**.
3. O workflow assume uma role IAM via **OIDC** e chama o modelo no Bedrock com o
   diff e a saída do checkov.
4. O modelo devolve uma revisão em JSON (resumo, risco geral e lista de achados).
5. O workflow publica a revisão como comentário no PR.

A divisão de trabalho é importante: **o checkov encontra, a IA explica**. O
checkov não alucina, mas a saída dele é longa e genérica. O modelo prioriza o
que importa nas linhas alteradas e explica em português. A IA **não aprova nem
bloqueia** o PR: ela é uma segunda leitura, não a revisão humana.

## Por que Bedrock

Daria para usar a API da Anthropic direto, com uma API key guardada nos secrets.
Com o Bedrock:

- O diff **não sai da sua conta AWS**, o que facilita a conversa com segurança e
  compliance na empresa.
- A autenticação usa **IAM**. Com o OIDC do GitHub, não existe nenhuma chave
  guardada no repositório.
- A cobrança vem na fatura da AWS.
- Com a mesma role e a mesma conta, dá para trocar de modelo (e de fornecedor)
  mudando só uma variável, a `MODEL_ID`.

## Acesso aos modelos

O revisor aceita uma lista fechada de modelos:

| `MODEL_ID` | Modelo | API usada |
|---|---|---|
| `anthropic.claude-opus-5-5` (padrão) | Claude Opus 5.5 | SDK da Anthropic (`bedrock-mantle`) |
| `anthropic.claude-opus-4-8` | Claude Opus 4.8 | SDK da Anthropic (`bedrock-mantle`) |
| `openai.gpt-oss-120b-1:0` | gpt-oss-120b | API Converse do Bedrock |
| `openai.gpt-oss-20b-1:0` | gpt-oss-20b | API Converse do Bedrock |

Ter o modelo no catálogo da região não basta. Na minha conta, os modelos da
Anthropic passaram por três etapas:

1. **Acordo do modelo**: preencher o formulário de caso de uso e aceitar a oferta
   do modelo no console do Bedrock.
2. **Permissão IAM** na role que chama o modelo (próxima seção).
3. **Cota**: mesmo com o acordo aceito, as cotas de tokens por minuto da conta
   estavam **zeradas**, e toda chamada voltava com acesso negado. Cota zerada
   se resolve com um pedido de aumento no **Service Quotas**.

Para não adivinhar em qual etapa o problema está, o script tem um comando que
lista os modelos suportados, o status do acordo e, com `--test`, faz uma
chamada mínima a cada um. Esta é a saída real na minha conta, enquanto o
aumento de cota do Claude não sai:

```text
$ python reviewer/review.py --list-models --test
| ID | Provedor | Nome | Acordo | Chamada |
|---|---|---|---|---|
| `anthropic.claude-opus-5-5` | anthropic | Claude Opus 5.5 | AVAILABLE | acesso negado |
| `anthropic.claude-opus-4-8` | anthropic | Claude Opus 4.8 | AVAILABLE | acesso negado |
| `openai.gpt-oss-120b-1:0` | openai | gpt-oss-120b | AVAILABLE | ok |
| `openai.gpt-oss-20b-1:0` | openai | gpt-oss-20b | AVAILABLE | ok |

Padrão: `anthropic.claude-opus-5-5`. Escolha outro com a variável `MODEL_ID`.
```

O acordo está `AVAILABLE` para todos, mas só o gpt-oss responde. Por isso os
testes desta série usam o `openai.gpt-oss-120b-1:0`.

## Criando a role

Se você ainda não cadastrou o GitHub como identity provider no IAM, siga o
post [GitHub Actions com OpenID Connect na AWS, sem access keys](/posts/2026/2026-08-22-github-actions-com-oidc-na-aws-sem-access-keys/).
Ele é a base desta parte.

### Trust policy

Como o workflow roda em pull requests, o `sub` é
`repo:meu-usuario/meu-repo:pull_request`:

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
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:meu-usuario/meu-repo:pull_request"
        }
      }
    }
  ]
}
```

Antes de criar a role, confira o formato do `sub` no seu repositório. Se ele
usa o *subject imutável*, o `sub` inclui os IDs numéricos do dono e do
repositório, como `repo:meu-usuario@123456/meu-repo@7891011:pull_request`. Com a
trust policy acima, o workflow falha com `Not authorized to perform
sts:AssumeRoleWithWebIdentity`. Para ver o prefixo que o GitHub usa:

```bash
gh api repos/meu-usuario/meu-repo/actions/oidc/customization/sub --jq .sub_claim_prefix
```

Na trust policy, use esse prefixo seguido de `:pull_request`. A vantagem do
formato imutável é que ele não aceita um repositório apagado e recriado com o
mesmo nome.

### Permissão para chamar os modelos

A integração atual do Claude no Bedrock usa o endpoint `bedrock-mantle`, e a
ação IAM de inferência é `bedrock-mantle:CreateInference`. Ela é diferente do
`bedrock:InvokeModel` da integração antiga. O recurso é o **project** do Bedrock:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ClaudeNoBedrock",
      "Effect": "Allow",
      "Action": "bedrock-mantle:CreateInference",
      "Resource": "arn:aws:bedrock-mantle:*:111122223333:project/*"
    }
  ]
}
```

Para restringir a role a modelos específicos, existe a condition key
`bedrock-mantle:Model`.

O gpt-oss usa a API Converse do Bedrock, que é autorizada pela ação
`bedrock:InvokeModel` no ARN do modelo. Se você só vai usar um dos dois
fornecedores, deixe só o statement dele:

```json
{
  "Sid": "GptOss",
  "Effect": "Allow",
  "Action": "bedrock:InvokeModel",
  "Resource": [
    "arn:aws:bedrock:*::foundation-model/openai.gpt-oss-120b-1:0",
    "arn:aws:bedrock:*::foundation-model/openai.gpt-oss-20b-1:0"
  ]
}
```

Criando a role:

```bash
aws iam create-role \
  --role-name github-iac-ai-reviewer \
  --assume-role-policy-document file://iam/trust-policy.json

aws iam put-role-policy \
  --role-name github-iac-ai-reviewer \
  --policy-name bedrock-claude \
  --policy-document file://iam/bedrock-policy.json
```

Repare que essa role **só** chama o modelo. Ela não precisa de nenhuma
permissão na infraestrutura que está sendo revisada.

Para rodar o revisor na sua máquina, não crie access key: use um perfil do
IAM Identity Center com as mesmas permissões, como mostrei no post
[Acesso à AWS sem access keys com o IAM Identity Center](/posts/2026/2026-10-31-acesso-a-aws-sem-access-keys-com-iam-identity-center/).

## Próximo passo

Com a role pronta, na **parte 2** vamos ver o script em Python: como montar o
prompt, receber a resposta em um formato garantido e o que fazer quando o
modelo recusa.

## Referências

- [Claude in Amazon Bedrock](https://platform.claude.com/docs/en/build-with-claude/claude-in-amazon-bedrock)
- [OpenAI models in Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters-openai.html)
- [Actions, resources, and condition keys for Amazon Bedrock Powered by AWS Mantle](https://docs.aws.amazon.com/service-authorization/latest/reference/list_bedrock-mantle.html)
- [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)

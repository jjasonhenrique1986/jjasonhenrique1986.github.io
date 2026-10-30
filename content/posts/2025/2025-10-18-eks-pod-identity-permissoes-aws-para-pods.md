---
title: "EKS Pod Identity: permissões da AWS para pods sem access keys"
description: "Como usar o EKS Pod Identity para dar permissões do IAM a pods no Amazon EKS sem access keys, e a diferença para o IRSA."
image: "/images/eks.jpg"
author: Jason
date: 2025-10-18
tags: [ "Cloud", "AWS", "EKS", "Kubernetes", "Security" ]
type: post
showTableOfContents: true
---

![eks](/images/eks.jpg)

O objetivo desse artigo é demonstrar como usar o **EKS Pod Identity**
para que um pod rodando no Amazon EKS acesse serviços da AWS, como o S3,
**sem colocar access keys** dentro do cluster.

Colocar `AWS_ACCESS_KEY_ID` e `AWS_SECRET_ACCESS_KEY` em um Secret
funciona, mas são credenciais de longa duração que podem vazar e
precisam ser rotacionadas. Com o Pod Identity o pod recebe credenciais
**temporárias** de uma role IAM, e quem define qual role cada aplicação
usa é a associação com a ServiceAccount do Kubernetes.

## Pré Requisitos

-   Um cluster EKS. Se precisar criar um, veja o artigo sobre o
    [eksctl](/posts/2022/2023-09-07-usando-eksctl-para-criacao-e-gerenciamento-de-um-cluster-eks/).
-   Nós do tipo EC2 (managed node groups ou self-managed). O Pod
    Identity **não é suportado em pods no Fargate**.
-   Uma versão recente do AWS CLI v2, que tenha o comando
    `aws eks create-pod-identity-association`.
-   O kubectl configurado para o cluster.

Nos exemplos o cluster se chama `meu-cluster` e o ID da conta é
`111122223333`. Troque pelos seus valores.

## Como funciona

1.  O add-on **EKS Pod Identity Agent** roda como um DaemonSet em cada
    nó.
2.  Você cria uma role IAM que confia no serviço
    `pods.eks.amazonaws.com`.
3.  Você cria uma **associação** entre a role e uma ServiceAccount de um
    namespace.
4.  Quando um pod com essa ServiceAccount é criado, o EKS injeta
    variáveis de ambiente que fazem os SDKs da AWS buscarem credenciais
    no agente local, que por sua vez obtém as credenciais temporárias da
    role.

## Instalando o add-on

```bash
aws eks create-addon \
  --cluster-name meu-cluster \
  --addon-name eks-pod-identity-agent
```

Verifique se os pods do agente estão rodando:

```bash
kubectl get pods -n kube-system | grep eks-pod-identity-agent
```

## Criando a role IAM

Crie um arquivo chamado trust-pod-identity.json. Repare nas duas ações:
o Pod Identity precisa de `sts:AssumeRole` e também de `sts:TagSession`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "pods.eks.amazonaws.com" },
      "Action": [ "sts:AssumeRole", "sts:TagSession" ]
    }
  ]
}
```

Agora crie a role e, para o exemplo, anexe a policy gerenciada de
leitura no S3:

```bash
aws iam create-role \
  --role-name eks-s3-reader \
  --assume-role-policy-document file://trust-pod-identity.json

aws iam attach-role-policy \
  --role-name eks-s3-reader \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

Em produção, prefira uma policy própria liberando apenas os buckets e
as ações que a aplicação realmente precisa.

## Criando a ServiceAccount e a associação

```bash
kubectl create namespace app
kubectl create serviceaccount s3-reader -n app
```

Diferente do IRSA, **não é preciso anotar a ServiceAccount**. A ligação
é feita pela associação:

```bash
aws eks create-pod-identity-association \
  --cluster-name meu-cluster \
  --namespace app \
  --service-account s3-reader \
  --role-arn arn:aws:iam::111122223333:role/eks-s3-reader
```

Para listar as associações do cluster:

```bash
aws eks list-pod-identity-associations --cluster-name meu-cluster
```

## Testando

Crie um arquivo chamado aws-cli.yaml com um pod usando a imagem oficial
do AWS CLI e a ServiceAccount que criamos:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: aws-cli
  namespace: app
spec:
  serviceAccountName: s3-reader
  containers:
  - name: aws-cli
    image: amazon/aws-cli:latest
    command: ["sleep", "infinity"]
```

```bash
kubectl apply -f aws-cli.yaml
```

Primeiro confira quem é a identidade do pod:

```bash
kubectl exec -n app aws-cli -- aws sts get-caller-identity
```

O campo `Arn` deve mostrar um `assumed-role/eks-s3-reader/...`. Agora
liste os buckets da conta:

```bash
kubectl exec -n app aws-cli -- aws s3 ls
```

Se você tentar uma ação que a role não permite, como criar um bucket, o
retorno será `AccessDenied`, o que confirma que o pod está usando apenas
as permissões da role.

As credenciais são injetadas **na criação do pod**. Se você criar a
associação depois que o pod já estiver rodando, é preciso recriar o pod
(por exemplo com `kubectl rollout restart` em um Deployment).

## Pod Identity ou IRSA?

O **IRSA** (IAM Roles for Service Accounts) era a forma recomendada
antes do Pod Identity e continua funcionando. As principais diferenças:

| | IRSA | EKS Pod Identity |
|---|---|---|
| Provedor OIDC | Precisa criar um por cluster no IAM | Não precisa |
| Trust policy | Referencia o OIDC de um cluster específico | Confia em `pods.eks.amazonaws.com`, a mesma role serve para vários clusters |
| ServiceAccount | Precisa da anotação `eks.amazonaws.com/role-arn` | Sem anotação, associação feita pela API do EKS |
| Fargate | Suportado | Não suportado |
| Fora do EKS | Funciona em outros clusters com OIDC | Exclusivo do EKS |

O Pod Identity também adiciona **session tags** com informações como o
nome do cluster, o namespace e a ServiceAccount. Elas podem ser usadas em
policies para controle de acesso por atributos (ABAC).

## Limpeza

```bash
kubectl delete namespace app

aws eks delete-pod-identity-association \
  --cluster-name meu-cluster \
  --association-id <id-retornado-no-list>

aws iam detach-role-policy \
  --role-name eks-s3-reader \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam delete-role --role-name eks-s3-reader
```

## Conclusão

Com o EKS Pod Identity as aplicações no cluster acessam a AWS com
credenciais temporárias e com o princípio do menor privilégio, sem
access keys espalhadas em Secrets. A configuração é mais simples que a
do IRSA, principalmente quando você tem vários clusters.

Se quiser complementar, veja também como
[configurar permissão somente leitura em um cluster EKS](/posts/2023/2023-11-11-configurando-permissao-readonly-em-um-cluster-eks/),
que trata do caminho inverso: dar a uma role da AWS acesso ao Kubernetes.

## Referências

-   [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
-   [Configurar o EKS Pod Identity Agent](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-agent-setup.html)

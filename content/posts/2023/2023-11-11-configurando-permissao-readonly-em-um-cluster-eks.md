---
title: "Configurando permissão somente leitura em um cluster EKS"
description: "Como configurar uma role da AWS com permissão somente leitura em um cluster EKS usando o kubectl."
image: "/images/eks.jpg"
author: Jason
date: 2023-11-11
tags: [ "Kubernetes", "Cloud", "AWS", "Containers", "EKS" ]
type: post
showTableOfContents: true
---

![eks](/images/eks.jpg)

O objetivo desse post é demonstrar como fazer a configuração para que
uma determinada role da AWS tenha permissão de somente leitura dentro de
um cluster EKS usando o kubectl que é um utilitário para administrar o
Kubernetes através de linha de comando.

## Pré Requisitos 

Para conseguir seguir com essa demosntração será necessário os seguintes
pré requisitos abaixo:

-   Conta na AWS.
-   Um cluster eks já criado. Para fazer a criação de um cluster EKS
    você pode usar esse
    [artigo](/posts/2022/2023-09-07-usando-eksctl-para-criacao-e-gerenciamento-de-um-cluster-eks/)
    que falei sobre o eksctl.
-   Kubectl instalado. Mais informações nesse
    [link](https://kubernetes.io/pt-br/docs/tasks/tools/install-kubectl-linux/).
-   AWS CLI instalado. Mais informações nesse
    [link](https://docs.aws.amazon.com/pt_br/cli/latest/userguide/getting-started-install.html).

**Lembrando que essa demonstração terá custo na AWS.**

## Criação de user e role AWS 

Primeiramente devemos criar um usuário iam aws sem nenhuma permissão
conforme esse
[link](https://docs.aws.amazon.com/pt_br/IAM/latest/UserGuide/id_users_create.html)
lembrando que esse usuário deve ter acesso via CLI.

Antes da criação da role é necessário a criação de uma policy que tenha
apenas permissão de list e read no EKS conforme abaixo:

``` bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "eks:ListNodegroups",
                "eks:ListEksAnywhereSubscriptions",
                "eks:DescribeFargateProfile",
                "eks:DescribeAddonConfiguration",
                "eks:ListTagsForResource",
                "eks:ListAddons",
                "eks:DescribeEksAnywhereSubscription",
                "eks:DescribeAddon",
                "eks:ListFargateProfiles",
                "eks:DescribeNodegroup",
                "eks:DescribeIdentityProviderConfig",
                "eks:ListUpdates",
                "eks:DescribeUpdate",
                "eks:AccessKubernetesApi",
                "eks:DescribeCluster",
                "eks:ListClusters",
                "eks:DescribeAddonVersions",
                "eks:ListIdentityProviderConfigs"
            ],
            "Resource": "*"
        }
    ]
}
```

Agora vamos criar uma role com a policy acima anexada. Outro ponto de
atenção é que devemos criar uma relação de confiança para que o usuário
iam criado anteriormente possa fazer o assume role. Abaixo segue como
deverá ser configurado essa relação de confiança:

``` bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::id-conta:user/jason"
            },
            "Action": "sts:AssumeRole",
            "Condition": {}
        }
    ]
}
```

Onde **id-conta** é a a informação da sua conta.

Para mais informações sobre essas configurações acessar esse
[link](https://docs.aws.amazon.com/pt_br/IAM/latest/UserGuide/id_roles_create_for-user.html).

## Alteração arquivo aws-auth 

O primeiro passo que devemos fazer é a inclusão da role AWS no configmap
aws-auth dentro do cluster EKS. Para isso vamos usar uma role com
permissões administrativas. Para isso vamos executar o seguinte comando
para edição desse configmap:

``` bash
eksctl create iamidentitymapping --cluster nome-cluster --region=us-east-1 \
    --arn arn:aws:iam::id-conta:role/eks --username {{SessionName}} --group system:masters
```

Para verificacao podemos executar o comando abaixo:

``` bash
eksctl get iamidentitymapping --cluster nome-cluster --region=us-east-1
```

Essa configuração também pode ser feita manualmente conforme informado
nesse
[link](https://docs.aws.amazon.com/pt_br/eks/latest/userguide/add-user-role.html).

## Configuração de Profile AWS 

Nesse passo vamos fazer a configuração das credenciais do usuário iam,
criação de um novo profile referenciando a role criada e acertar a
configuração do kubeconfig.

Primeiramente vamos configurar o usuário iam criado usando o comando
abaixo onde será solicitado o access key, secret access key, region e
output format:

``` bash
aws configure
```

Essas informações pode ser consultadas acessando o arquivo
.aws/credentials. Será exibido uma saída similar a abaixo:

``` bash
[default]
aws_access_key_id = XXXXXX
aws_secret_access_key = XXXXXX
```

Depois vamos configurar um profile no arquivo .aws/config

``` bash
[profile eks]
role_arn = arn:aws:iam::id-conta:role/eks
source_profile = default
```

Onde **source_profile** deverá ser informado o nome cadastrado no
arquivo .aws/credentials

Agora vamos acertar o kubeconfig referenciado o profile criado. A seção
de users do arquivo kubeconfig deve ser incluido o argumento --profile
eks conforme abaixo:

``` bash
users:
- name: eksctl@cluster-demoeksctl.us-east-1.eksctl.io
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      args:
      - eks
      - get-token
      - --output
      - json
      - --cluster-name
      - cluster-demoeksctl
      - --region
      - us-east-1
      - --profile
      - eks
      command: aws
      env:
      - name: AWS_STS_REGIONAL_ENDPOINTS
        value: regional
      provideClusterInfo: false
```

## Configuração de permissão somente leitura no Kubernetes 

Após a configuração do item anterior foi verificado que mesmo usando um
role AWS com perfil apenas de leitura ainda era possível alteração de
recursos usando kubectl.

Foi necessário a criação de um clusterrole, clusterrolebinding e um
group com permissões de list, get e watch em todos os recursos do
kubernetes conforme abaxio:

``` bash
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: developers-readonly
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: developers-readonly
subjects:
- kind: Group
  name: developers-readonly
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developers-readonly
  apiGroup: rbac.authorization.k8s.io
```

Aplicar o conteúdo acima criando um arquivo chamado readonly.yaml e
executar o comando abaixo:

``` bash
kubectl apply -f readonly.yaml
```

Após isso se faz necessário a remoção da entrada abaixo no configmap
aws-auth:

``` bash
eksctl delete iamidentitymapping --cluster nome-cluster --region=us-east-1 --arn arn:aws:iam::id-conta:role/eks --all
```

Com um usuário com permissão administrativa verificar se a role AWS foi
removida:

``` bash
eksctl get iamidentitymapping --cluster nome-cluster --region=us-east-1
```

Depois adicionar a role AWS com o Group criado anteriormente junto com
clusterrole e clusterrolebinding executando o comando abaixo:

``` bash
eksctl create iamidentitymapping --cluster nome-cluster --region=us-east-1 --arn arn:aws:iam::id-conta:role/eks --username {{SessionName}} --group developers-readonly
```

## Validação de acesso 

Agora para validar se a role AWS está realmente com acesso somente
leitura tentamos criar um secret no cluster usando o comando abaixo:

``` bash
kubectl create secret generic teste --from-literal=user=jason
```

Deve ser apresentado o seguinte erro conforme imagem abaixo:

![eks-2](/images/eks-2.jpg)

Depois tentamos listar todos os pods executando no cluster e obtivemos
sucesso conforme imagem abaixo:

![eks-3](/images/eks-3.jpg)

Com isso garantimos que o acesso está somente leitura aos objectos do
Kubernetes.

## Conclusão 

O objectivo desse artigo foi demonstar como conceder permissão somente
leitura aos recursos do Kubernetes vinculando com uma role AWS.

Outro ponto que para conceder permissão somente leitura podemos
implementar uma solução de interface gráfica que mostre todos os
recursos do cluster EKS como por o Kubernetes Dashboard.

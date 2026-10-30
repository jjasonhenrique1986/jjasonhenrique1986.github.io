---
title: Gerenciando acesso no Kubernetes com RBAC 
description: "Como gerenciar o controle de acesso no Kubernetes com RBAC: Roles, ClusterRoles e seus bindings."
image: "/images/kubernetes.jpg"
author: Jason
date: 2024-11-15
tags: [ "Kubernetes", "Security", "Containers" ]
type: post
showTableOfContents: true
---

![kubernetes](/images/kubernetes.jpg)

No Kubernetes, a segurança e o controle de acesso são aspectos críticos
para garantir que apenas as pessoas e processos corretos tenham as
permissões necessárias para interagir com o cluster. Para gerenciar
essas permissões, o Kubernetes utiliza um sistema baseado em **RBAC
(Role-Based Access Control)**.

Dando continuidade no meu post anterior, vamos explorar alguns outros
objetos do kubernetes que tem como principal função fazer esse controle
de acesso.

## Pré Requisitos

Para conseguir acompanhar esse artigo será necessário os seguintes pré
requisitos abaixo:

-   Ter um cluster kubernetes no nosso caso utilizaremos o kind.
-   Ter o kubectl instalado.

Para os exemplos iremos criar um namespace onde serão feitos nossos
testes. Para criar esse namespace executar o comando abaixo:

``` bash
kubectl create ns app
```

## Service Account 

Uma **Service Account** ou Conta de Serviço no Kubernetes é usada para
representar uma identidade de um processo ou pod que roda dentro do
cluster. Ela é necessária quando um pod ou aplicação precisa interagir
com a API do Kubernetes, seja para listar pods, criar objetos ou
realizar outras operações.

Por padrão, os pods utilizam a `default` service account do namespace
onde estão localizados, mas é uma boa prática criar contas de serviço
separadas para diferentes aplicações ou pods com necessidades
específicas. Isso ajuda a isolar permissões e a seguir o princípio de
menor privilégio.

### Exemplo de Uso

Nesse exemplo vamos criar um arquivo chamado serviceaccount.yaml com o
conteúdo abaixo:

``` bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: app
```

Para aplicar o exemplo acima executar o comando abaixo:

``` bash
kubectl apply -f serviceaccount.yaml
```

Para verificar a criação da serviceaccount executar o comando abaixo:

``` bash
kubectl get serviceaccount -n app
```

Para ver mais detalhes da sua serviceaccount criada você pode executar o
comando abaixo também:

```bash
kubectl get serviceaccount nomeserviceaccount -n app -o yaml

kubectl describe serviceaccount nomeserviceaccount -n app
```

Com a serviceaccount você também consegue interagir com recursos na AWS
usando a opção annotations conforme exemplo abaixo:

``` bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: example-serviceaccount
  namespace: app
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/ExampleIAMRole
```

## Role

Uma **Role** é um conjunto de permissões que se aplica a um namespace
específico. Você pode usá-la para definir o que um usuário, um grupo ou
uma service account pode fazer dentro daquele namespace, como criar,
listar, ou deletar recursos.

### Exemplo de Uso

Nesse exemplo vamos criar um arquivo chamado role.yaml com o conteúdo
abaixo:

``` bash
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: app
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
```

Neste exemplo, a role `pod-reader` permite que a Service Account veja
(`get`, `list`, `watch`) nos pods e nos deployments dentro do namespace
app.

Para aplicar o exemplo acima executar o comando abaixo:

```bash
kubectl apply -f role.yaml
```

Para visualizar a criação da role executar o comando abaixo:

``` bash
kubectl get role -n app
```

Para visualizar mais detalhes da sua role executar o comando abaixo:

``` 
kubectl describe role nomerole -n app
```

## ClusterRole 

Uma **ClusterRole** é semelhante a uma Role, mas com a diferença de que
ela não é restrita a um único namespace. Uma ClusterRole pode definir
permissões para recursos que estão fora do namespace, como nodes e
volumes, ou mesmo para definir permissões globais em múltiplos
namespaces.

### Exemplo de Uso

Nesse exemplo vamos criar um arquivo chamado clusterrole.yaml com o
conteúdo abaixo:

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: teste-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods", "nodes"]
  verbs: ["get", "list", "watch"]
```

No exemplo acima, a ClusterRole teste-clusterrole dá acesso de leitura a
pods e nodes em todo o cluster. Podemos perceber nesse exemplo que não
temos a necessidade de inserir o campo namespace.

Para aplicar o exemplo acima executar o comando abaixo:

``` bash
kubectl apply -f clusterrole.yaml
```

Para visualizar as clusterroles no cluster executar o comando abaixo:

``` bash
kubectl get clusterrole
```

Será mostrado todas as clusterroles do cluster. Para visualizar
informações da clusterrole que criamos podemos executar o comando
abaixo:

``` bash
kubectl get clusterrole teste-clusterrole
```

Para exibir informações mais detalhadas executar um dos comandos abaixo:

``` bash
kubectl get clusterrole nomeclusterrole -o yaml

kubectl describe clusterrole nomeclusterrole
```

## RoleBinding 

Um **RoleBinding** associa uma Role a um usuário, grupo ou service
account dentro de um namespace específico. O RoleBinding é o mecanismo
que efetivamente concede as permissões definidas na Role para a entidade
alvo no nosso caso a nossa serviceaccount.

### Exemplo de Uso

Nesse exemplo vamos criar um arquivo chamado rolebinding.yaml com o
conteúdo abaixo:

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: readonly
  namespace: app
subjects:
- kind: ServiceAccount
  name: my-service-account
  namespace: app
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Aqui, o `RoleBinding` está associando a `Role` chamada `pod-reader` à
`ServiceAccount` chamada `my-service-account`, garantindo que essa conta
de serviço tenha permissões de leitura de pods e deployments dentro do
namespace app.

Para aplicar o exemplo acima executar o comando abaixo:

```bash
kubectl apply -f rolebinding.yaml
```

Para visualizar a criação da rolebinding executar o comando abaixo:

``` bash
kubectl get rolebinding -n app
```

Para visualizar mais informações executar o comando abaixo:

``` bash
kubectl describe rolebinding -n app
```


## ClusterRoleBinding 

Semelhante ao RoleBinding, o **ClusterRoleBinding** associa uma
**ClusterRole** a um usuário, grupo ou service account, mas com o escopo
de todo o cluster. Isso significa que as permissões concedidas pela
ClusterRole estarão disponíveis em todos os namespaces.

### Exemplo de Uso

Nesse exemplo vamos criar um arquivo chamado clusterrolebinding.yaml com
o conteúdo abaixo:

``` bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: teste-clusterrole-binding
subjects:
- kind: ServiceAccount
  name: my-service-account
  namespace: app
roleRef:
  kind: ClusterRole
  name: teste-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

Neste exemplo, o ClusterRoleBinding `teste-clusterrole``-binding`
associa a ClusterRole `teste-clusterrole` à `ServiceAccount`
`my-service-account`, dando à conta de serviço permissões de leitura de
pods e nodes em todo o cluster.

Para aplicar o exemplo acima executar o comando abaixo:

``` bash
kubectl apply -f clusterrolebinding.yaml 
```

Para visualizar as clusterrolebindings no cluster executar o comando
abaixo:

``` bash
kubectl get clusterrolebinding
```

Para exibir informações mais detalhadas executar um dos comandos abaixo:

``` bash
kubectl get clusterrolebinding nomeclusterrolebinding -o yaml

kubectl describe clusterrolebinding nomeclusterrolebinding
```

## Validando as permissões 

Para validar as permissões da nossa serviceaccount podemos executar o
seguinte comando abaixo:

```bash
sudo kubectl auth can-i list pods --as=system:serviceaccount:app:my-service-account -n app
```

Se o resultado da sua consulta for yes sua serviceaccount tem acesso a
fazer determinada ação.

Você pode executar o comando abaixo para validar que sua serviceaccount
não tem acesso a fazer essa ação:

``` bash
sudo kubectl auth can-i delete deployments --as=system:serviceaccount:app:my-service-account -n app
```

## Associando nossa ServiceAccount

Agora que sabemos tudo sobre RBAC chegou a hora de criarmos nosso
deployment com a nossa nova serviceaccount garantido assim mais
segurança ao nosso cluster kubernetes.

Abaixo segue um exemplo de deployment usando a serviceaccount que
criamos:

``` bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: app
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      serviceAccountName: my-service-account
      containers:
      - name: nginx
        image: nginx:1.25.2
        ports:
        - containerPort: 80
```

Deve ser criado um arquivo chamado deploy.yaml com o conteúdo acima e
executado o comando abaixo:

``` bash
kubectl apply -f deploy.yaml
```

Para visualizar se realmente foi feito vinculo da serviceaccont com o
deploy executar o comando abaixo:

``` bash
kubectl get deploy -n app -o yaml | grep serviceAccount
```


## Conclusão 

O Objetivo desse post foi demonstrar a utilização do RBAC no seu
cluster, garantindo que os usuários e processos tenham acesso controlado
e seguro aos recursos de que precisam, ajudando a prevenir acessos
indesejados e falhas de segurança.

Abaixo alguns links da documentação do kubernetes falando sobre RBAC:

-   <https://kubernetes.io/docs/reference/access-authn-authz/rbac/>
-   <https://kubernetes.io/docs/concepts/security/rbac-good-practices/>

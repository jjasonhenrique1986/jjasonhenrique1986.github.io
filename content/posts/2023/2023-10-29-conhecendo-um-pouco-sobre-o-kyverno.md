---
title: "Conhecendo um pouco sobre o kyverno"
description: "Introdução ao Kyverno, dynamic admission controller do Kubernetes: instalação e criação de uma policy do tipo validation."
image: "/images/kyverno.jpg"
author: Jason
date: 2023-10-29
tags: [ "Kubernetes", "Security", "Containers" ]
type: post
showTableOfContents: true
---

![kyverno](/images/kyverno.jpg)

O Objectivo desse artigo é demonstrar a utilização da ferramenta kyverno
dentro de um ambiente kubernetes. O kyverno é um dynamic admission
controller dentro do kubernetes que permite voce aplicar políticas
dentro do seu cluster.

Nos links abaixo você pode ter mais informaçõoes dessa ferramenta:

-   <https://kyverno.io/>
-   <https://github.com/kyverno/kyverno/>

## Pré Requisitos 

Para testar o kyverno segue abaixo os pré requisitos:

-   Ter um cluster kubernetes para isso você pode usar o
    [kind](https://kind.sigs.k8s.io/) ou [rancher
    desktop](https://rancherdesktop.io/).
-   Ter o kubectl instalado mais informações nesse
    [link](https://kubernetes.io/docs/tasks/tools/).
-   Ter o helm instalado mais informações nesse
    [link](https://helm.sh/docs/intro/install/).

## Instalação 

Para a instalação do kyverno você pode fazer de duas maneiras uma usando
o helm e outra aplicando um arquivo de manifesto do kubernetes usando o
comando kubectl.

Em ambas as formas de instalação e recomendado a criação de um namespace
para o kyverno usando o comando abaixo:

``` bash
kubectl create ns kyverno
```

## Instalação usando Helm 

Para fazer a instalação do kyverno usando o helm devem ser executados os
comandos abaixo:

``` bash
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update
helm install kyverno-policies kyverno/kyverno-policies -n kyverno
```

## Instalação usando kubectl 

Para a instalação usando o kubectl deve ser executado o comando abaixo:

``` bash
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.10.0/install.yaml -n kyverno
```

## Conceitos Kyverno 

Após a instalação do kyverno você terá disponível dois tipos de kind
para a criação de suas políticas:

-   **ClusterPolicy** -- que pode ser aplicado a todo o cluster
    kubernetes.
-   **Policy** -- que pode ser aplicado a um determinado namespace.

Também haverá dois tipos de políticas que poderao ser criadas:

-   **Validation** -- onde nessa política será validado algo por exemplo
    se há determinada label em um deployment.
-   **Mutation** -- onde sera alterado determinada configuração dentro
    de um deployment

Dentro desse dois tipos de políticas haverá duas ações que poderão ser
feitas:

-   **Audit** -- onde apenas será logado se determinado deployment esta
    seguindo ou nao determinada política do kyverno.
-   **Enforce** -- caso um deployment não atenda determinada política do
    kyverno o mesmo não será criado.

## Testando uma policy do tipo Validation 

Para testar uma policy do tipo validation vamos criar a ClusterPolcy
abaixo que valida se existe as labels owner e environment. Em um
primeiro momento usando a ação **Audit** e depois a ação **Enforce**.

``` bash
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
  annotations:
    policies.kyverno.io/title: Require Labels
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/minversion: 1.6.0
    policies.kyverno.io/severity: medium
    policies.kyverno.io/subject: Pod, Label
    policies.kyverno.io/description: >-
      This policy validates that the label `owner` and `environment` is specified with some value.      
spec:
  validationFailureAction: Audit
  background: true
  rules:
  - name: check-for-labels
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "The label `owner` and `environment` is required."
      pattern:
        metadata:
          labels:
            owner: "?*"
            environment: "?*"
```

Para aplicar a clusterpolicy acima criar um arquivo chamado
**require-labels.yaml** e executar o comando abaixo:

``` bash
kubectl apply -f require-labels.yaml
```

Para verificar se a política foi criada executar o comando abaixo:

``` bash
kubectl get clusterpolicy
```

Deve apresentar uma saída similar com a imagem abaixo:

![kyverno-2](/images/kyverno-2.jpg)

Para testar vamos criar um arquivo de deployment sem as labels owner e
environment conforme o arquivo abaixo:

``` bash
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: app
  name: app
  namespace: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: app
    spec:
      containers:
      - image: nginx
        name: nginx
```

Para aplicar o deployment acima basta criar o arquivo nginx.yaml e
executar o comando abaixo:

``` bash
kubectl apply -f nginx.yaml
```

Para visualizar se o deployment atende ou não a policy criada podemos
executar os seguintes comandos abaixo:

``` bash
kubectl describe clusterpolicy require-labels
kubectl get events -n app
```

Onde **app** e o namespace onde o deployment foi aplicado. Nos eventos
devera aparecer uma saída similar com a imagem abaixo:

![kyverno-3](/images/kyverno-3.jpg)

Agora vamos alterar a clusterpolicy para que seja obrigatório a presença
das labels owner e environment. Caso essas labels não estejam presentes
o deployment não sera criado.

Antes de alterar a clusterpolicy excluir o deployment usando o comando
abaixo:

``` bash
kubectl delete -f nginx.yaml
```

Para alterar a clusterpolicy para **Enforce** voçê deve alterar a
configuração abaixo de Audit para Enforce conforme abaixo:

``` bash
spec:
  validationFailureAction: Enforce
```

Depois disso salvar o arquivo e aplicar novamente a policy usando o
comando abaixo:

``` bash
kubectl apply -f require-labels.yaml
```

Após isso tentar aplicar o deployment novamente usando o comando
kubectl. Deverá aparecer a seguinte mensagem de erro conforme imagem
abaixo:

![kyverno-4](/images/kyverno-4.jpg)

Para resolver essa mensagem de erro deve ser inserido as labels owner e
environment conforme abaixo:

``` bash
  template:
    metadata:
      labels:
        app: app
        owner: jason
        environment: develop
```

## Conclusão

O Objetivo desse artigo foi demonstar a utilização da ferramenta
Kyverno. No proximo artigo vou demonstar a utilização do kyverno usando
uma policy do tipo Mutation.

Caso queira se aprofundar na ferramenta os links abaixo podem te ajudar:

-   <https://playground.kyverno.io/#/>
-   <https://kyverno.io/resources/>
-   <https://kyverno.io/policies/>
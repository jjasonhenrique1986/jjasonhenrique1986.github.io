---
title: "Instalando a ferramenta kubecost"
description: "Passo a passo da instalação do Kubecost, ferramenta de monitoramento e gerenciamento de custos para Kubernetes e cloud."
image: "/images/kubecost.jpg"
author: Jason
date: 2023-01-04
tags: [ "Cloud", "AWS", "FinOps", "Kubernetes" ]
type: post
showTableOfContents: true
---

![kubecost](/images/kubecost.jpg)

O intuito deste post é demostrar a instalação do kubecost uma ferramenta
de monitoramento e gerenciamento de custos para um ambiente kubernetes e
também para cloud suportando os principais cloud providers (AWS, Azure e
GCP).

O kubecost possui dois tipos de licença (free e paga). Na versão free as
informações coletadas ficam disponíveis apenas 15 dias.

Para mais informações sobre kubecost acessar os links abaixo:

- https://docs.kubecost.com/
- https://github.com/kubecost/cost-analyzer-helm-chart

## Pré-Requisitos 

Como pré-requisito é necessário ter um cluster de kubernetes para a
instalação do kubecost. Para essa demonstração pode ser utilizado o kind
ou o rancher desktop.

Para mais informações sobre a instalação e criação de um cluster usando
kind utilizar
esse [link](https://kind.sigs.k8s.io/).

Para informações sobre o rancher desktop acesse
esse [link](https://rancherdesktop.io/).

## Instalação 

No meu cenário como é um ambiente de teste vou fazer a instalação
utilizando o kubectl mas a instalação também pode ser feita via helm.

Criar um namespace chamado kubecost onde será deployado a aplicação.
Segue abaixo comando:

``` bash
kucebctl create ns kubecost
```

Executar o comando abaixo para instalação:

``` bash
kubectl apply -f https://raw.githubusercontent.com/kubecost/cost-analyzer-helm-chart/master/kubecost.yaml -n kubecost
```

Será deployado os seguintes objetos conforme imagem abaixo:

![kubecost-2](/images/kubecost-2.jpg)

Abaixo overview dos objetos deployados pelo kubecost

![kubecost-3](/images/kubecost-3.jpg)

Na instalação default é efetuado a instalação de um deploy do
prometheus-server que será responsável por coletar as informações do
cluster. É possivel utilizar seu próprio servidor do prometheus porém é
recomendavel a utilização desse prometheus apartado.

## Acessando a console do kubecost 

Para acessar a console do kubecost será utilizado o comando kubectl
port-forward para testar localmente. Segue abaixo comando:

``` bash
 kubectl port-forward service/kubecost-cost-analyzer 9090:9090 -n kubecost
```

Abaixo alguns prints da console do kubecost

![kubecost-4](/images/kubecost-4.jpg)

![kubecost-5](/images/kubecost-5.jpg)

## Conclusão 

O intuito desse post foi demonstrar a instalação do kubecost. No próximo
post estarei mostrando as principais funcionalidades da console do
kubecost.
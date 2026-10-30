---
title: "Overview da ferramenta kubecost"
description: "Um overview da console do Kubecost para monitorar e gerenciar o custo do Kubernetes e do cloud provider (AWS, Azure e GCP)."
image: "/images/kubecost.jpg"
author: Jason
date: 2023-01-07
tags: [ "Cloud", "AWS", "FinOps", "Kubernetes" ]
type: post
showTableOfContents: true
---

![kubecost](/images/kubecost.jpg)

O objetivo desse post é fazer um overview da console de administração da
ferramenta kubecost que ajuda no gerenciamento e monitoramento do custo
de um ambiente kubernetes e também do custo do cloud provider (AWS,
Azure e GCP) que você utiliza.

## Pré-Requisitos

Como pré-requisito é necessário já ter feito a instalação do kubecost
para fazer a instalação basta seguir
esse [link](/posts/2023/2023-01-04-instalando-a-ferramenta-kubecost/) com
as intruções.

## Overview da console do Kubecost 

Nas seções abaixo irei apresentar um pequeno overview da console de
adminstração do kubecost.

### Seção Overview 

Abaixo alguns prints da seção overview:

![kubecost-6](/images/kubecost-6.jpg)

![kubecost-7](/images/kubecost-7.jpg)

As seguintes informações estão disponíveis:

- Custo total do Kubernetes nos últimos 7 dias;
- Custo total da Cloud nos últimos 7 dias (é necessário a integração
    com o cloud provider mais informações
    nesse [link](https://docs.kubecost.com/install-and-configure/install/cloud-integration/multi-cloud));
- Custo total dos últimos 7 dias;
- Possible Savings são recomendações que podem ser aplicadas ao
    ambiente;
- Efficiency é um calculo feito baseado na utilização de CPU e Memória
    durante determinado tempo;
- Quantidade de nós e pods por cluster;
- Namespaces com maior gasto;

### Seção Savings 

Abaixo alguns prints da seção Savings:

![kubecost-8](/images/kubecost-8.jpg)

![kubecost-9](/images/kubecost-9.jpg)

Na seção savings são feitas recomendações para o ambiente que irá gerar
economia para o ambiente kubernetes e para cloud:

- Recomendação de Request/Limit para CPU e Memória;
- Mostra os recursos não utilizados na Cloud;
- Mostra Workloads abandonados;
- Resize de disco no node ou resize de node;

### Seção Assets e Allocations 

Abaixo alguns prints da seção Allocations:

![kubecost-10](/images/kubecost-10.jpg)

![kubecost-11](/images/kubecost-11.jpg)

Na seção Allocations são mostrados informações de custo separado por
CPU, RAM, Network e LoadBalance separado por namespace.

Prints da seção Assets:

![kubecost-12](/images/kubecost-12.jpg)

![kubecost-13](/images/kubecost-13.jpg)

Na seção Assets mostra as seguintes informações:

- Mostra os custos baseados nos recursos por exemplo Node, Disk,
    Network;
- Tem amostragem por data (lembrando que na versão free mostra
    informações de apenas 15 dias);
- Mostra o quanto cada node ou cada disk no ambiente está gastando;

### Seção Health 

Abaixo prints da seção Health:

![kubecost-14](/images/kubecost-14.jpg)

![kubecost-15](/images/kubecost-15.jpg)

Na seção Health é feito um checklist com boas práticas baseado em
confiabilidade e performance. Nesse checklist é feito um score de 0 a
100. Abaixo algumas dessas checagens:

- Se existem pods com status CrashLoopBackOff;
- Se existem pods com status pending;
- Pods com baixo uso de cpu;
- Memoria e Disco Pressure;
- Custo do Cluster aumentou mais de 10%;
- Evento de Out Of Memory;

### Seção Alerts e Reports 

Na seção reports permite você criar relatórios baseado na seção Asset e
Allocation conforme imagem abaixo:

![kubecost-16](/images/kubecost-16.jpg)

Na seção Alerts possui a funcionalidade de enviar alertas para um canal
do slack. Segue abaixo imagem:

![kubecost-17](/images/kubecost-17.jpg)

Abaixo tipos de alertas que podem ser criados:

![kubecost-18](/images/kubecost-18.jpg)

## Conclusão 

O intuito desse post foi fazer um overview sobre a console de
administração do kubecost. Para mais informações do kubecost pode
consultar a documentação através
desse [link](https://docs.kubecost.com/).
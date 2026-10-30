---
title: "Conhecendo a ferramenta Octant"
description: "Conheça o Octant, projeto open source que permite visualizar e gerenciar um cluster Kubernetes por interface gráfica."
image: "/images/octant.jpg"
author: Jason
date: 2023-01-09
tags: [ "Kubernetes", "Observability", "Containers" ]
type: post
showTableOfContents: true
---

![octant](/images/octant.jpg)

A ferramenta Octant é um projeto Open Source que permite a visulização
via interface gráfica do seu cluster Kubernetes podendo assim fazer todo
o gerenciamento do seu cluster kurbenetes e aplicações através dessa
ferramenta.

A ferramenta Octant é mantida pela VMWare abaixo segue links com mais
informações:

- https://octant.dev/
- https://github.com/vmware-tanzu/octant

## Pré-Requisitos 

Para testar essa ferramenta é necessário ter um cluster kubernetes
local. Pode ser usado o kind ou o rancher desktop para teste. Abaixo
segue links com mais informações:

- https://kind.sigs.k8s.io/
- https://rancherdesktop.io/

Também é necessário que o kubeconfig esteja configurado pois a
ferramenta coleta informações desse arquivo conforme imagem abaixo:

![octant-2](/images/octant-2.jpg)

Para mais informações sobre a configuração do arquivo kubeconfig
consultar
esse [link](https://kubernetes.io/pt-br/docs/concepts/configuration/organize-cluster-access-kubeconfig/).

## Instalação 

Para utilização é necessário a instalação de um cliente na sua máquina
local. O mesmo suporta Windows, Linux e Mac.

Para fazer o download basta acessar o link abaixo:

-  https://github.com/vmware-tanzu/octant/releases/tag/v0.25.1

No momento que escrevo esse post essa é a ultima versão da ferramenta.

## Overview 

Nessa seção será apresentado um overview da ferramenta.

![octant-3](/images/octant-3.jpg)

Na imagem acima conseguimos ter a visualização de todos os workloads que
estão rodando em determinado namespace. Assim como o uso de CPU e
Memória para cada workload.

![octant-4](/images/octant-4.jpg)

![octant-5](/images/octant-5.jpg)

Acima outros exemplos de visualizações.

Permite a visualização dos objetos do kubernetes que determinado
workload está utilizando. Segue abaixo alguns exemplos:

![octant-6](/images/octant-6.jpg)

![octant-7](/images/octant-7.jpg)

![octant-8](/images/octant-8.jpg)

É possível fazer um port-forward com apenas um clique

![octant-9](/images/octant-9.jpg)

Permite visualizar as variáveis de ambiente

![octant-10](/images/octant-10.jpg)

Visualização do yaml utilizado pelo Deploy

![octant-11](/images/octant-11.jpg)

Visualização de logs e conexão ao pod através da ferramenta

![octant-12](/images/octant-12.jpg)

![octant-13](/images/octant-13.jpg)

É possível visualização de eventos conforme imagem abaixo:

![octant-14](/images/octant-14.jpg)

Permite aplicar um yaml através da ferramenta

![octant-15](/images/octant-15.jpg)

Também é possível integrar a ferramenta com outros plugins conforme link
abaixo:

- https://github.com/topics/octant-plugin

## Conclusão 

O objetivo desse post foi demostrar a utilização da ferramenta Octant e
como pode ajudar no gerenciamento de um cluster kubernetes via interface
gráfica ajudando assim na sua produtividade.

Existem ferramentas similares como o Kubernetes-Dashboard e o Lens.
Abaixo links com mais informações:

- [Kubernetes Dashboard](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/)
- [Lens](https://k8slens.dev/)
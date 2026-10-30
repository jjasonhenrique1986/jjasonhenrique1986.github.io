---
title: "Conhecendo a ferramenta de analise estática kics"
description: "Como usar o KICS para fazer análise estática de código de Infraestrutura como Código, como Terraform e CloudFormation."
image: "/images/kics.jpg"
author: Jason
date: 2022-09-10
tags: [ "CI/CD", "CodeReview", "Security" ]
type: post
showTableOfContents: true
---
![kics](/images/kics.jpg)

Nesse artigo irei demonstrar a utilização de um ferramenta
chamada **kics** ou **keeping infrastructure as code secure** tem a
função de executar uma analise estática em códigos de Infraestrutura
como Código como por exemplo Terraform, Cloudformation e outros.

O kics é uma ferramenta Open Source desenvolvida em Golang usando OPA
(Open Policy Agent). Para mais informações sobre OPA consulte
esse [link](https://www.openpolicyagent.org/docs/latest/#overview). O OPA permite você aplicar políticas para seus
microsserviços, kubernetes, pipelines de CI/CD, API Gateways e muitas
outras opções.

O kics faz uma analise estática procurando por vulnerabilidades de
segurança, questões de compliance e erros de configurações em suas
ferramentas de Infraestrutura como Código. No momento que escrevo esse
artigo as seguintes ferramentas abaixo são suportadas:

![kics-2](/images/kics-2.jpg)

Através do kics você pode criar novas queries conforme a necessidade do
seu ambiente. Mais informações
nesse [link](https://docs.kics.io/latest/creating-queries/).

## Instalação 

Para a utilização do kics você tem duas opções instalar através do
código fonte ou utilizar o kics através de uma imagem docker. No meu
caso vou utilizar uma imagem docker.

### Docker 

``` bash
docker pull checkmarx/kics:latestdocker run -t -v "{path_to_host_folder_to_scan}":/path checkmarx/kics scan -p "/path" -o "/path/"
```

Para mais informações sobre a instalação pode consultar
esse [link](https://docs.kics.io/latest/getting-started/).

## Demonstração 

Para essa demonstração irei utilizar o
seguinte [repositório](https://github.com/addamstj/tfcourse). Por padrão o kics gera um arquivo json para melhorar
a visualização vamos gerar um arquivo html.

Para executar basta executar o comando abaixo:

``` bash
nerdctl run -it -v "C:\Users\jhmrj\Documents\tfcourse":/tmp checkmarx/kics scan -p "/tmp/" -o "/tmp/results" --report-formats "html"
```

**Observação:** Estou utilizando o utilitário nerdctl pois na minha
máquina estou executando o containerd. No docker irá funcionar da mesma
forma.

![kics-3](/images/kics-3.jpg)

Abaixo algumas das regras identificadas pelo kics:

![kics-4](/images/kics-4.jpg)

![kics-5](/images/kics-5.jpg)

![kics-6](/images/kics-6.jpg)

![kics-7](/images/kics-7.jpg)

Foi gerado um arquivo html no caminho /tmp/results/results.html. Abaixo
segue alguns prints desse report.

![kics-8](/images/kics-8.jpg)

![kics-9](/images/kics-9.jpg)

## Integração 

Com o kics você consegue integrar com várias ferramentas de CI/CD e
também existe um plugin para vscode.
Nesse [link](https://docs.kics.io/latest/integrations/) você consegue verificar todas as integrações
disponíveis.

## Referências 

- [Github Kics](https://github.com/Checkmarx/kics)
- [Documentação Oficial](https://docs.kics.io/latest/getting-started/)
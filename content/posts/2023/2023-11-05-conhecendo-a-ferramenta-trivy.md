---
title: Conhecendo a ferramenta trivy
description: "Conheça o Trivy, scanner de segurança da Aqua Security e tópico cobrado no exame CKS (Certified Kubernetes Security Specialist)."
image: "/images/trivy.jpg"
author: Jason
date: 2023-11-05
tags: [ "Kubernetes", "Security", "Containers", "CI/CD" ]
type: post
showTableOfContents: true
---

![trivy](/images/trivy.jpg)

O trivy é uma ferramenta desenvolvido pela empresa Aqua security e
tambem é um dos topicos cobrados no exame da CKS (Certified Kubernetes
Security Specialist) para quem pretende tirar. Abaixo alguns links com
mais informações do trivy:

-   <https://github.com/aquasecurity/trivy>
-   <https://aquasecurity.github.io/trivy/>

Com o trivy você consegue fazer as seguintes verificações:

-   Scan de Vulnerabilidades
-   Erros de configuração em IaC
-   Detectar dados sensíveis

É possível escanear os seguintes targets:

-   Imagem de Container
-   Repositório de algum sistema de versionamento de código como por
    exemplo o GitHub
-   Sistema de Arquivos
-   Clusters Kubernetes
-   Contas AWS

O objectivo desse artigo é demonstrar o scan de vulnerabilidades em uma
imagem de container e em um repositório do github usando o Trivy.

## Instalação 

Para usar o trivy você pode instalar diretamente na sua máquina ou usar
o trivy como um container do Docker.

### Instalação no Linux 

No meu caso vou demonstrar a instalação em uma máquina Linux. Para
verificar a última versão do trivy consultar esse
[link](https://github.com/aquasecurity/trivy/releases). Abaixo os
comandos que você deve executar para fazer o download e a instalação:

``` bash
wget https://github.com/aquasecurity/trivy/releases/download/v0.46.0/trivy_0.46.0_Linux-64bit.deb
sudo dpkg -i trivy_0.46.0_Linux-64bit.deb
```

Para validar a instalação executar o comando abaixo:

``` bash
trivy version
```

### Docker 

Para utilizar o trivy usando um container basta usar a imagem
**aquasec/trivy:latest**. Por exemplo para verificar qual a versão do
Trivy executar o comando abaixo:

``` bash
docker run aquasec/trivy:latest version
```

## Scan de Imagem de Container 

Para executar um scan de vulnerabilidades em alguma imagem de container
basta executar o comando abaixo

``` bash
trivy image nginx

docker run aquasec/trivy:latest image nginx
```

Onde **nginx** é o nome da imagem de container

Deverá aparecer uma saída similar abaixo:

![trivy-2](/images/trivy-2.jpg)

Você pode filtar por severidade HIGH e CRITICAL usando o comando abaixo:

``` bash
trivy image nginx --severity=HIGH --severity=CRITICAL
```

Agora deverá aparecer somente as vulnerabilidades com a severidade HIGH
e CRITICAL conforme imagem abaixo:

![trivy-3](/images/trivy-3.jpg)

Na coluna **status** podemos verificar que o status esta como
**affected** isso quer dizer que não existe uma solução para essas
vulnerabilidades ainda.

## Scan de repositório 

Agora vamos fazer um scan de vulnerabilidades no seguinte
[repositório](https://github.com/prometheus/prometheus) executando o
comando abaixo filtrando apenas as vulnerabilidades com severidade HIGH.

``` bash
trivy repository https://github.com/prometheus/prometheus --severity=HIGH
```

Abaixo segue a saída do comando:

![trivy-4](/images/trivy-4.jpg)

Nesse caso a coluna **status** esta com o status **fixed** isso quer
dizer que já existe uma solução para a vulnerabilidade. Na coluna
**Fixed Version** está a versão que a vulnerabilidade foi corrigida.

## Integração 

O trivy faz integração com diversas alicações e ferramentas que permitem
adicionar segurança ao workflow da sua aplicação como por exemplo
ferramentas de CI/CD e com IDEs. Para mais informações sobre essas
integrações consultar esse
[link](https://aquasecurity.github.io/trivy/v0.46/ecosystem/).

## Conclusão 

O objetivo desse artigo foi demonstrar um pouco da utilização do trivy
para efetuar scan de vulnerabilidades.
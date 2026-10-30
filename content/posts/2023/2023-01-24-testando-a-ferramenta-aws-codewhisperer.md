---
title: "Testando a ferramenta AWS CodeWhisperer"
description: "Testando o AWS CodeWhisperer, ferramenta da AWS que sugere código usando Machine Learning, similar ao GitHub Copilot."
image: "/images/codewhisperer.jpg"
author: Jason
date: 2023-01-24
tags: [ "Cloud", "AWS", "CodeReview", "Security" ]
type: post
showTableOfContents: true
---

![codewhisperer](/images/codewhisperer.jpg)

O objetivo desse post é demonstrar a utilização do serviço AWS
CodeWhisperer que é uma ferramenta que sugere códigos para um
desenvolvedor usando Machine Learning e Inteligência Artificial. Essa
ferramenta é similar ao Github Copilot.

Essa ferramenta suporta as seguintes linguagens:

-   C#
-   Java
-   JavaScript
-   Python
-   TypeScript

E se integra com as seguintes IDEs:

-   Intellij IDEA
-   PyCharm
-   VS Code
-   AWS Cloud9
-   AWS Lambda

Para essa demonstração irei utilizar o VS Code. Para mais informações
sobre esse serviço da AWS acessar esse
[link](https://aws.amazon.com/pt/codewhisperer/).

O serviço também permite fazer um scan de segurança do seu código.

## Pré-Requisitos 

Para testar esse serviço da AWS você deve ter os seguintes
pré-requisitos:

-   VS Code instalado. Para fazer o download acessar esse
    [link](https://code.visualstudio.com/download).
-   Instalar a extensão AWS Toolkit no VS Code conforme imagem abaixo:

![codewhisperer-2](/images/codewhisperer-2.jpg)

## Instalação e Configuração 

Apos a instalação da extensão da AWS Toolkit irá aparecer um ícone da
aws no lado esquerdo similar a imagem abaixo:

![codewhisperer-3](/images/codewhisperer-3.jpg)

Para ativar a utilização do serviço AWS CodeWhisperer será necessário
configurar a conexão com a AWS.

Eu utilizei a opção **"Use a personal email to sign up and sign in with
AWS Builder ID "** assim que clicar nessa opção você será redirecionada
para uma página web para efetuar a criação do seu AWS Builder ID.

![codewhisperer-4](/images/codewhisperer-4.jpg)

## Demonstração 

Para a demonstração eu criei um arquivo com a extensão .py e adicionei
um comentário sobre o que gostaria de criar por exemplo "Create simple
API with fastAPI" com isso ele vai dando sugestões de código conforme
imagem abaixo:

![codewhisperer-5](/images/codewhisperer-5.jpg)

Esse foi o resultado final apresentado pelo AWS CodeWhisperer

![codewhisperer-6](/images/codewhisperer-6.jpg)

## Conclusão 

Achei bem legal esse serviço da AWS pode ser uma alternativa bem
interessante ao Github Copilot.
---
title: "Testando o recurso Synthetics Canaries do Cloudwatch"
description: "Como criar um Synthetics Canary no CloudWatch para checar periodicamente se uma URL ou API está respondendo com sucesso."
image: "/images/synthetic.jpg"
author: Jason
date: 2023-02-06
tags: [ "Cloud", "AWS", "Observability" ]
type: post
showTableOfContents: true
---

![synthetic](/images/synthetic.jpg)

O objetivo desse post é demonstrar a criação de um synthetic canary na
AWS. É um recurso que está dentro do serviços oferecidos dentro do
Cloudwatch e tem a principal função de testar se uma URL ou uma API está
retornando status de sucesso (HTTP 200) durante uma checagem a cada um
minuto por exemplo. Esse recurso existe em outras ferramentas de
Observabilidade que já trabalhei por exemplo Dynatrace e Datadog.

Para mais informações sobre esse serviço consulte esse
[link](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Synthetics_Canaries.html).

## Pré-Requisitos

Para testar esse serviço da AWS foi criar uma API em Python usando o
FastAPI. Para saber mais informações sobre o que é o FastAPI e como
criar uma API usando FastAPI pode consultar esse
[link](https://pythonacademy.com.br/blog/como-usar-o-fastapi-para-construir-apis-no-python). E também o
[link](https://fastapi.tiangolo.com/) da documentação oficial.

Para a instalação do FastAPI utilizarei uma EC2 com as seguintes
configurações abaixo:

![synthetic-2](/images/synthetic-2.jpg)

**Lembrando que essa EC2 deve possuir um IP Publico.**

## Instalação FastAPI

Após a criação da EC2 na AWS deverá ser executado os seguintes comandos
abaixo para a instalação do FastAPI:

``` bash
sudo apt update
sudo apt install python3-pip -y
pip install fastapi
pip install uvicorn
```

Para validar a instalação execute o comando abaixo:

``` bash
pip show fastapi
```

Deverá retornar algo conforme imagem abaixo:

![synthetic-3](/images/synthetic-3.jpg)

Após a instalação do FastAPI devemos criar o arquivo **main.py**
conforme abaixo:

``` bash
from fastapi import FastAPI

app = FastAPI()

@app.get("/")

def index():
    return "Hello World"
```

Após salvar o arquivo acima e executar o seguinte comando abaixo para
iniciar a execução do FastAPI:

``` bash
uvicorn main:app --host 0.0.0.0 --port 8080
```

No meu caso estou usando a porta 8080. Será necessário a liberação dessa
no security group da EC2 conforme imagem abaixo:

![synthetic-4](/images/synthetic-4.jpg)

Utilizando o dns público ou ip público da sua EC2 na porta 8080 devemos
ter um resultado similar conforme imagem abaixo:

![synthetic-5](/images/synthetic-5.jpg)

## Configuração Synthetics Canaries

Para continuar a configuração do recurso Synthetic Canaries do
Cloudwatch é necessário já ter criado a API conforme seção acima.

Para iniciar a configuração você deve ir na opção **Application
Monitoring** \>\> **Synthetics Canaries** no serviço do **Cloudwatch**
conforme imagem abaixo:

![synthetic-6](/images/synthetic-6.jpg)

Será mostrado uma tela conforme abaixo:

![synthetic-7](/images/synthetic-7.jpg)

Para efetuar a criação basta clicar na opção **Create Canary** e será
apresentado a tela abaixo:

![synthetic-8](/images/synthetic-8.jpg)

Nesta primeira seção temos a opção de usar padrões já criados pela AWS,
usar nossos próprios scripts e importar de um bucket S3. No meu caso vou
usar a primeira opção.

![synthetic-9](/images/synthetic-9.jpg)

Nesta segunda seção você irá definir o nome do Canary e o endpoint URL
que será checado.

![synthetic-10](/images/synthetic-10.jpg)

Nesta terceira seção será escolhido o runtime que será executado. No meu
caso eu optei pelo runtime que utiliza o python3.8 + selenium + chromium
mas existem outras opções. Também é possível configurar variáveis de
ambiente para a execução do script.

![synthetic-11](/images/synthetic-11.jpg)

Nesta quarta seção você irá configurar de quanto em quanto tempo será
executado a checagem do endpoint URL.

Outras opções que podem ser configuradas nessa mesma tela:

-   Tempo de retenção dos resultados;
-   Bucket onde será salvo os resultados;
-   Criação de uma IAM Role;
-   Criação de um alarme no Cloudwatch;
-   Permite que a checagem só execute dentro de uma determinada VPC;
-   Configuração de Tags;

Após a criação será apresentado esse dashboard conforme imagens abaixo:

![synthetic-12](/images/synthetic-12.jpg)

![synthetic-13](/images/synthetic-13.jpg)


Simulei parar a execução da API para verificar se o mesmo é alertado
conforme imagem abaixo:

![synthetic-14](/images/synthetic-14.jpg)

Também é possível criar o Synthetics Canaries através do Terraform
conforme informado nesse
[link](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/synthetics_canary).

## Conclusão

O intuito desse post foi demonstrar a criação de uma simples API e a
checagem dela utilizando o recurso Synthetics Canaries do Cloudwatch.


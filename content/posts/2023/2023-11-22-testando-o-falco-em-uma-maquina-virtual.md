---
title: "Testando o falco em uma máquina virtual"
description: "Segundo artigo sobre o Falco: instalação em uma máquina virtual na AWS e alteração do output de uma regra."
image: "/images/falco.jpg"
author: Jason
date: 2023-11-22
tags: [ "Kubernetes", "Security", "Containers", "AWS" ]
type: post
showTableOfContents: true
---

![falco](/images/falco.jpg)

O objetivo deste segundo artigo falando sobre a ferramenta falco é
demonstrar a utilização do falco instalando em uma máquina virtual na
AWS e fazer uma alteração no output de uma regra do falco.

Para ler meu primeiro artigo sobre o falco basta acessar esse
[link](/posts/2023/2023-11-16-instalando-e-testando-a-ferramenta-falco/).
Nesse artigo eu instalei o falco no Kubernetes.

## Pré Requisitos 

Abaixo os pré requisitos necessários para seguir esse artigo:

-   Conta AWS.
-   Uma máquina virtual na AWS (EC2).
-   EC2 deve ter acesso a Internet.
-   EC2 deve estar acessível via SSH.

## Instalação 

Para a instalação irei executar uma EC2 rodando Ubuntu 22.04. Abaixo
segue os comandos utilizados para a instalação do falco:

Adicionar a chave repositório do Falco

``` bash
    curl -fsSL https://falco.org/repo/falcosecurity-packages.asc | \
    sudo gpg --dearmor -o /usr/share/keyrings/falco-archive-keyring.gpg
```

Adicionar o repositório do Falco

``` bash
echo "deb [signed-by=/usr/share/keyrings/falco-archive-keyring.gpg] https://download.falco.org/packages/deb stable main" > /etc/apt/sources.list.d/falcosecurity.list
```

Fazer o update do SO executando o comando abaixo:

``` bash
apt-get update -y
```

Após finalizar o update instalar alguns pacotes obrigatórios para a
instalação:

``` bash
apt-get install -y dkms make linux-headers-$(uname -r) dialog
```

Agora sim vamos fazer a instalação do falco usando o comando abaixo:

``` bash
apt-get install -y falco
```

Deve ser escolhido a opção eBPF conforme abaixo:

![falco-4](/images/falco-4.jpg)

Para verificar se o falco foi instalado com sucesso deve ser executado o
comando abaixo:

``` bash
systemctl status falco-bpf
```

Deve apresentar uma saída conforme tela abaixo:

![falco-5](/images/falco-5.jpg)

## Demonstração 

Agora vamos validar se o falco realmente está funcionando executando o
comando abaixo:

``` bash
cat /etc/shadow
```

E depois vamos verificar os logs do falco executando o comando abaixo:

``` bash
grep Sensitive /var/log/syslog
```

Deve apresentar a seguinte saída:

![falco-6](/images/falco-6.jpg)

## Alteração regra do Falco 

Para verificar todas as regras que existem por default no falco você
deve acessar o seguinte arquivo /etc/falco/falco_rules.yaml. Vamos
alterar o output da regra Read sensitive file untrusted de Sensitive
file opened for reading by non-trusted program para ACESSO INDEVIDO.

Primeiramente vamos copiar o conteudo abaixo da linha 396 ate 431 para o
arquivo /etc/falco_rule.local.yaml. É uma boa prática colocar as regras
customizadas nesse arquivo.

![falco-7](/images/falco-7.jpg)

A seção output deve ser alterada conforme abaixo:

![falco-8](/images/falco-8.jpg)

Depois devemos salavar o arquivo **/etc/falco_rule.local.yaml**, parar e
iniciar o serviço do falco usando os comandos abaixo:

``` bash
systemctl stop falco-bpf
systemctl start falco-bpf
```

Para verificar se o falco está funcionando executar o comando abaixo:

``` bash
systemctl status falco-bpf
```

Agora vamos executar o comando cat /etc/shadow porem o grep deve ser
modificado:

``` bash
grep ACESSO /var/log/syslog
```

Por fim deve apresentar uma saída conforme abaixo:

![falco-9](/images/falco-9.jpg)
</figure>

## Conclusão 

O objectivo desse artigo era fazer a instalação do falco em uma máquina
virtual e também a alteração de uma regra do falco.

Para saber mais informações sobre as rEgras default do falco acessar
esse [link](https://falco.org/docs/rules/).
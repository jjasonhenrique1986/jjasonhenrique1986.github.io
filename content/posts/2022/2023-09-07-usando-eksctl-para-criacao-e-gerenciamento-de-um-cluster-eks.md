---
title: "Usando eksctl para criação e gerenciamento de um cluster EKS"
description: "Como usar o eksctl, ferramenta de linha de comando da Weaveworks, para criar e gerenciar um cluster Amazon EKS."
image: "/images/eksctl.jpg"
author: Jason
date: 2022-09-07
tags: [ "Cloud", "AWS", "EKS", "Kubernetes" ]
type: post
showTableOfContents: true
---
![eksctl](/images/eksctl.jpg)

Nesse artigo irei demostrar a utilização da ferramenta de linha de
comando chamada eksctl feita em Golang e criada pela empresa Weaveworks
para criação e gerenciamento de um cluster EKS.

Com essa ferramenta de linha de comando é possível criar um cluster EKS
e também gerenciar seus node groups que não foram criados com o eksctl
por exemplo um cluster que foi provisionado através do terraform.

**Lembrando que o EKS não está na oferta de free tier da AWS então caso**
**prossiga com esse lab você será cobrado pelo tempo de utilização do**
**cluster EKS**

## Instalação 

O eksctl pode ser instalado em máquinas que utilizam Windows, Linux e
MAC ou também ser executado através do Docker.

### Para Linux

``` bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmpsudo mv /tmp/eksctl /usr/local/bin
```

### Para MAC

Também pode ser utilizado a seguinte linha de comando para usuários Mac:

``` bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

### Para Windows

``` bash
chocolatey install eksctl
```

No windows também pode ser utilizado o PowerShell.

No meu caso eu utilizei uma máquina Linux. Para verificar se a
instalação foi bem sucedida basta executar o comando abaixo:

``` bash
eksctl version
```

Ou digitando apenas eksctl deve retornar a saída abaixo:

![eksctl-2](/images/eksctl-2.jpg)

Para mais detalhes sobre a instalação pode consultar esse [link](https://eksctl.io/introduction/#installation)

## Pré-Requisitos 

- Usuário AWS com as permissões abaixo:

![eksctl-3](/images/eksctl-3.jpg)

Para mais informações sobre as permissões necessárias para o eksctl
funcionar acesse esse [link](https://eksctl.io/usage/minimum-iam-policies/). Abaixo segue como ficou as minhas políticas no IAM:

![eksctl-4](/images/eksctl-4.jpg)

Para instruções de como criar um usuário na AWS consulte este [link](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html).

- AWS CLI instalado. Segue abaixo procedimento para a instalação:

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Dependendo da distribuição Linux será necessário instalar o unzip.

Para mais informações sobre a instalação do aws cli consulte
esse [link](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

- Kubectl instalado. Segue abaixo procedimento para a instalação:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"chmod +x ./kubectlmv ./kubectl /usr/local/bin/ 
```

Para mais informações sobre a instalação do kubectl consulte esse [link](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).

# Criando seu primeiro cluster EKS com eksctl 

Após a instalação dos pré-requisitos da seção anterior você está pronto
para a criação do seu primeiro cluster. O eksctl é uma forma mais
simples de subir um cluster EKS em background o eksctl provisiona todos
os recursos de infraestrutura utilizando **Cloudformation**.

Por padrão o eksctl sobe instâncias do tipo m5.large para economizar
dinheiro optei pela criação do cluster utilizando o arquivo abaixo
setando o tipo de instância para utilizar t3.medium.

``` bash
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: cluster-demoeksctl              
  region: us-east-1                       

nodeGroups:
  - name: ng-1                
    instanceType: t3.medium   
    desiredCapacity: 1        
  - name: ng-2
    instanceType: t3.medium 
    desiredCapacity: 1
```

Criar um arquivo com o nome **eksctl.yml** com conteúdo acima e executar o
comando abaixo:

``` bash
eksctl create cluster -f eksctl.yml
```

Abaixo a execução do comando e a saída para criação do cluster.

![eksctl-5](/images/eksctl-5.jpg)

Na saída abaixo é mostrado o provisionamento dos dois Nodegroups

![eksctl-6](/images/eksctl-6.jpg)

Criado os dois Nodegroups com um node por nodegroup conforme imagens
abaixo.

![eksctl-7](/images/eksctl-7.jpg)

![eksctl-8](/images/eksctl-8.jpg)

## Criando um novo Nodegroup com eksctl 

Com o eksctl você consegue também gerenciar seu cluster criado ou não
pelo eksctl. Algumas tarefas que conseguimos executar com o eksctl:

- Criação de um novo Nodegroup;
- Criação de um Fargate Profile;
- Update de versão do cluster;

Para mais informações podem consultar a documentação oficial do eksctl
que vou deixar na seção de referências.

No meu caso irei demonstar a criação de um novo nodegroup no cluster EKS
criado anteriormente. Novamente vou fazer a criação de forma declarativa
usando o arquivo abaixo:

``` bash
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: cluster-demoeksctl            
  region: us-east-1 

nodeGroups:
  - name: ng-3               
    instanceType: t3.medium     
    desiredCapacity: 1  
```

Criar um arquivo com o nome **nodegroup.yml** com conteúdo acima e executar
o comando abaixo:

``` bash
eksctl create nodegroup -f nodegroup.yml
```

![eksctl-9](/images/eksctl-9.jpg)

![eksctl-10](/images/eksctl-10.jpg)

## Deletando os Nodegroups e o cluster EKS

Para deletar apenas o nodegroup basta executar o comando abaixo:

``` bash
eksctl delete nodegroup -f nodegroup.yml --approve
```

![eksctl-11](/images/eksctl-11.jpg)

Podemos perceber na imagem abaixo que temos apenas 2 nodes que foram criados juntamente com o cluster.

![eksctl-12](/images/eksctl-12.jpg)

Agora para deletar o cluster EKS criado basta executar o comando abaixo:

``` bash
eksctl delete cluster -f eksctl.yml
```

![eksctl-13](/images/eksctl-13.jpg)

## Referências 

- [Github eksctl](https://github.com/weaveworks/eksctl)
- [Documentação Oficial](https://eksctl.io/)
- [Documentação Oficial EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
- [Exemplos de utilização do eksctl](https://github.com/weaveworks/eksctl/tree/main/examples)

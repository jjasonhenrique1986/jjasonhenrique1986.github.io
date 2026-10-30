---
title: "Acessando instâncias EC2 com o Session Manager, sem SSH"
description: "Como acessar instâncias EC2 com o AWS Systems Manager Session Manager, sem abrir a porta 22 e sem chave SSH, incluindo port forwarding."
author: Jason
date: 2025-04-12
tags: [ "Cloud", "AWS", "Security" ]
type: post
showTableOfContents: true
---

O objetivo desse artigo é demonstrar como acessar uma instância EC2
usando o **Session Manager**, um recurso do AWS Systems Manager. Com ele
não é necessário abrir a porta 22 no security group, não é preciso
gerenciar chaves SSH e o acesso é controlado por permissões do IAM.

Além disso, cada sessão fica registrada no CloudTrail, e é possível
gravar o conteúdo das sessões em um bucket S3 ou no CloudWatch Logs.

## Pré Requisitos

-   Uma conta AWS e o AWS CLI v2 configurado. Se precisar, veja como
    [criar e configurar um usuário IAM](/posts/2024/2024-01-07-criando-e-configurando-um-usuario-iam/)
    ou, de forma mais segura,
    [uma role IAM](/posts/2024/2024-03-30-criando-e-configurando-uma-role-iam/).
-   O **Session Manager plugin** instalado na sua máquina (passo abaixo).

## Como funciona

Para uma instância aparecer no Session Manager são necessárias três
coisas:

1.  **SSM Agent** instalado e em execução. Ele já vem instalado nas AMIs
    do Amazon Linux 2023 e do Ubuntu Server, por exemplo.
2.  Um **instance profile** com uma role que tenha a policy
    `AmazonSSMManagedInstanceCore`.
3.  **Acesso de rede de saída** da instância aos endpoints do Systems
    Manager, via internet ou NAT gateway, ou por VPC endpoints (`ssm`,
    `ssmmessages` e `ec2messages`) em subnets privadas.

Repare que a conexão parte do agente para a AWS. Por isso não é
necessária nenhuma regra de entrada no security group.

## Instalando o Session Manager plugin

No Ubuntu/Debian:

```bash
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o "session-manager-plugin.deb"
sudo dpkg -i session-manager-plugin.deb
session-manager-plugin --version
```

Para outros sistemas operacionais siga a
[documentação oficial](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html).

## Criando a role e o instance profile

Crie um arquivo chamado trust-ec2.json com a trust policy que permite ao
serviço EC2 assumir a role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Agora crie a role, anexe a policy gerenciada e crie o instance profile:

```bash
aws iam create-role \
  --role-name SSMInstanceRole \
  --assume-role-policy-document file://trust-ec2.json

aws iam attach-role-policy \
  --role-name SSMInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam create-instance-profile --instance-profile-name SSMInstanceProfile

aws iam add-role-to-instance-profile \
  --instance-profile-name SSMInstanceProfile \
  --role-name SSMInstanceRole
```

## Criando a instância

Vamos subir uma instância Amazon Linux 2023 na VPC default, **sem key
pair**. O parâmetro `resolve:ssm:` busca o ID da AMI mais recente direto
do Parameter Store, assim não precisamos procurar o ID manualmente:

```bash
aws ec2 run-instances \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --instance-type t3.micro \
  --iam-instance-profile Name=SSMInstanceProfile \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ssm-demo}]' \
  --query 'Instances[0].InstanceId' --output text
```

Guarde o ID da instância retornado (algo como `i-0abc...`). Se o comando
falhar dizendo que o instance profile não existe, aguarde alguns
segundos: o IAM pode levar um pouco para propagar.

Depois de um ou dois minutos, verifique se a instância já foi registrada
no Systems Manager:

```bash
aws ssm describe-instance-information \
  --query "InstanceInformationList[].{Id:InstanceId,Status:PingStatus,SO:PlatformName}" \
  --output table
```

O status deve aparecer como **Online**.

## Iniciando uma sessão

```bash
aws ssm start-session --target i-0abc123def4567890
```

Pronto, você está dentro da instância com um shell, sem SSH. Por padrão
a sessão abre com o usuário `ssm-user`. Para sair, digite `exit`.

A sessão também pode ser aberta pela console: **EC2 → selecione a
instância → Connect → Session Manager**.

## Port forwarding

Um recurso muito útil é o redirecionamento de portas. Por exemplo, para
acessar na sua máquina a porta 80 de uma instância (supondo que exista
um servidor web nela) pela porta local 8080:

```bash
aws ssm start-session \
  --target i-0abc123def4567890 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["80"],"localPortNumber":["8080"]}'
```

Também é possível usar a instância como ponte para um recurso em subnet
privada, como um banco RDS, com o documento
`AWS-StartPortForwardingSessionToRemoteHost`:

```bash
aws ssm start-session \
  --target i-0abc123def4567890 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["meu-banco.xxxx.us-east-1.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["5432"]}'
```

Assim você se conecta em `localhost:5432` e chega no banco, sem bastion
exposto na internet.

## Controlando o acesso com IAM

Quem pode abrir sessões é definido por policies IAM, usando a ação
`ssm:StartSession`. Um exemplo de policy que permite abrir sessões
apenas em instâncias com a tag `Ambiente=dev`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ssm:StartSession",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": { "ssm:resourceTag/Ambiente": "dev" }
      }
    },
    {
      "Effect": "Allow",
      "Action": [ "ssm:TerminateSession", "ssm:ResumeSession" ],
      "Resource": "arn:aws:ssm:*:*:session/${aws:userid}-*"
    }
  ]
}
```

## Limpeza

```bash
aws ec2 terminate-instances --instance-ids i-0abc123def4567890

aws iam remove-role-from-instance-profile \
  --instance-profile-name SSMInstanceProfile --role-name SSMInstanceRole
aws iam delete-instance-profile --instance-profile-name SSMInstanceProfile
aws iam detach-role-policy \
  --role-name SSMInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name SSMInstanceRole
```

## Conclusão

O Session Manager é uma forma mais segura de acessar instâncias: sem
porta 22 aberta, sem chaves para gerenciar e com o acesso controlado e
auditado pelo IAM e pelo CloudTrail. O port forwarding ainda substitui o
bastion host em muitos cenários.

## Referências

-   [AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
-   [Instalar o Session Manager plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)

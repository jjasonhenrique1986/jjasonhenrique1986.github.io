---
title: "Minha Jornada para certificação AWS SysOps Administrator"
description: "Como me preparei em 1 mês para a certificação AWS Certified SysOps Administrator Associate (SOA-C02) e os materiais que usei."
image: "/images/soa.jpg"
author: Jason
date: 2024-05-03
tags: [ "Cloud", "AWS", "Certificações" ]
type: post
showTableOfContents: true
---

![soa](/images/soa.jpg)

No último dia 2 de Maio fui aprovado na certificação AWS Certified
SysOps Administrator Associate (SOA-C02) e com isso resolvi compartilhar
sobre minha jornada de estudos e os materiais que usei durante esse 1
mês de preparação até conseguir a aprovação.

Essa foi a segunda vez que tirei a certificação AWS Certified SysOps
Administrator Associate já que havia prestado essa prova em 2021.

## Sobre a prova 

A prova é constituída 65 perguntas de múltipla escolha ou com várias
respostas certas em um tempo de 130 minutos para a prova em português.
No meu caso eu fiz a prova em inglês e tive um acréscimo de 30 minutos
por não ter o inglês como língua nativa.

A prova custa USD 150 ou caso você já tenha tirado outra certificação
AWS é possivel ter um voucher de 50% de desconto. Atualmente a prova
pode ser realizada pela Pearson VUE presencialmente ou remotamente. No
meu caso eu resolvi fazer em um centro de testes credenciado pela
Pearson VUE em Campinas -- SP na cidade onde eu moro.

Abaixo segue os domínios que são exigidos na prova:

-   Domain 1: Monitoring, Logging, and Remediation (20% do conteúdo
    pontuado)
-   Domain 2: Reliability and Business Continuity (16% do conteúdo
    pontuado)
-   Domain 3: Deployment, Provisioning, and Automation (18% do conteúdo
    pontuado)
-   Domain 4: Security and Compliance (16% do conteúdo pontuado)
-   Domain 5: Networking and Content Delivery (18% do conteúdo pontuado)
-   Domain 6: Cost and Performance Optimization (12% do conteúdo
    pontuado)

Para mais informações sobre a prova acessar
esse [link](https://d1.awsstatic.com/training-and-certification/docs-sysops-associate/AWS-Certified-SysOps-Administrator-Associate_Exam-Guide.pdf).

## Minha Preparação 

Minha principal fonte de estudo foi o curso da Udemy do Stephane Maarek
onde procurei colocar em prática todos os exemplos de labs que o
Stephane executa durante o curso.
Nesse [link](https://www.udemy.com/course/ultimate-aws-certified-sysops-administrator-associate) você
encontra mais informações do curso dele

Abaixo alguns tópicos e serviços que vale destacar que poderão ser
cobrados no exame:

-   Entender quando usar Multi-AZ ou read replicas no RDS
-   Entender como fazer a encriptação de um RDS que não está encriptado
-   Entender como fazer a encriptação de volume EBS
-   Entender sobre o EKS
-   Conhecimentos avançados na configuração de EC2 e autoscaling
-   Entender sobre os conceitos básicos de rede na AWS
-   Entender o que é um Elastic IP e quando utilizar
-   Entender quando utilizar uma IAM Role e como fazer essa configuração
-   Saber analisar uma IAM Policy e identificar o que essa politica faz
-   Entender sobre o serviço CloudTrail
-   Entender sobre AWS Config e como utilizar
-   Ententer sobre AWS System Manager
-   Entender como fazer a escalibilidade do seu cluster Elasticache
-   Entender a diferença entre RDS (MySQL) e Aurora
-   Entender quando utilizar o CloudFront e suas configurações
-   Entender a difrença entre security group e NACL
-   Entender como executar um workload Windows usando EC2
-   Entender sobre a criação e configuração de alertas no Cloudwatch
-   Quando é necessário a instalação do cloudwatch agent para coleta de
    métricas ou logs
-   Conhecimentos avançados nas configurações de um Load Balancer
-   Entender quando utilizar um ALB ou um NLB
-   Entender sobre o serviço SNS

## Simulados 

O principal simulado que utilizei foi da Whizlabs que na minha opinião é
o simulado mais parecido com o exame. Para mais informações acesse
esse [link](https://www.whizlabs.com/learn/course/aws-solutions-architect-associate/153).

Outro simulado que recomendo para sua preparação e
esse [simulado](https://www.udemy.com/course/aws-certified-sysops-administrator-associate-practice-exams-soa-c02/?couponCode=ACCAGE0923).

## Conclusão 

O intuito desse post foi compartilhar a minha experiência durante a
preparação para certificação AWS Certified SysOps Administrator
Associate assim como os materiais de estudo e os simulados. Abaixo
alguns outros links que podem te ajudar também:

-   <https://aws.amazon.com/pt/certification/certified-sysops-admin-associate/>
-   <https://d1.awsstatic.com/training-and-certification/docs-sysops-associate/AWS-Certified-SysOps-Administrator-Associate_Sample-Questions.pdf>
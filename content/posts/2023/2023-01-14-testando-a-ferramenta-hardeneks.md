---
title: "Testando a ferramenta HardenEKS"
description: "Testando o HardenEKS, ferramenta de linha de comando que verifica um cluster Kubernetes com base no guia de boas práticas de EKS da AWS."
image: "/images/hardeneks.jpg"
author: Jason
date: 2023-01-14
tags: [ "AWS", "Kubernetes", "EKS", "Containers" ]
type: post
showTableOfContents: true
---

![hardeneks](/images/hardeneks.jpg)

HardenEKS é uma ferramenta de linha de comando desenvolvido por [Doruk
Ozturk](https://github.com/dorukozturk) que atualmente trabalha na AWS. Essa ferramenta tem o
objetivo de fazer checagens em um cluster Kubernetes baseado em um guia
de boas práticas que foi criada pela AWS. Para consultar esse guia de
boas práticas basta acessar
esse [link](https://aws.github.io/aws-eks-best-practices/).

Mais informações do HardenEKS acesse
esse [link](https://github.com/aws-samples/hardeneks) também.

## Pré-Requisitos 

Como pré-requisito para a execução da ferramenta é necessário já ter
acesso a um cluster Kubernetes configurado através do arquivo kubeconfig
para informações sobre esse arquivo consultar
esse [link](https://kubernetes.io/pt-br/docs/concepts/configuration/organize-cluster-access-kubeconfig/).

Nessa demonstração irei utilizar um cluster criado apartir do rancher
desktop. Para mais informações acessar
esse [link](https://rancherdesktop.io/).

Para a execução da ferramenta em um cluster EKS é necessário que algumas
permissões mínimas sejam atendidas. Tanto dentro da AWS (IAM Policy)
como dentro do cluster EKS (RBAC).

Abaixo as permissões mínimas necessárias para a IAM Policy:

``` bash
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "eks:ListClusters",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "eks:DescribeCluster",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "ecr:DescribeRepositories",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "inspector2:BatchGetAccountStatus",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "ec2:DescribeFlowLogs",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "ec2:DescribeInstances",
            "Resource": "*"
        }
    ]
}
```

Abaixo ClusterRole que deve ser criado no cluster:

``` bash
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: hardeneks-runner
rules:
- apiGroups: [""]
  resources: ["namespaces", "resourcequotas", "persistentvolumes", "pods", "services"]
  verbs: ["list"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["clusterroles", "clusterrolebindings", "roles", "rolebindings"]
  verbs: ["list"]
- apiGroups: ["networking.k8s.io"]
  resources: ["networkpolicies"]
  verbs: ["list"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["list"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "statefulsets"]
  verbs: ["list", "get"]
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["list"]
```

## Instalação 

Para a instalação da ferramenta eu vou utilizar uma máquina Windows.
Para **Windows** será necessário a instalação do python através do link
abaixo:

[Link para download do
Python](https://www.python.org/downloads/)

Para **Linux** a maioria das distribuições já possuem o python
nativamente mas caso necessário utilizar
esse [link](https://python.org.br/instalacao-linux/) para fazer a instalação.

Para a instalação da ferramenta executar o seguinte comando abaixo:

``` bash
pip3 install hardeneks
```

Para verificar se o pacote foi instalado com sucesso executar o comando
abaixo:

![hardeneks-2](/images/hardeneks-2.jpg)

## Overview 

Para usar a ferramenta basta executar o comando abaixo:

``` bash
hardeneks
```

Abaixo alguns prints da saída da ferramenta

![hardeneks-3](/images/hardeneks-3.jpg)

![hardeneks-4](/images/hardeneks-4.jpg)

![hardeneks-5](/images/hardeneks-5.jpg)

No fim de cada checagem que a ferramenta faz ele informa um link para o
guia de boas práticas.

Existe a possibilidade de informar o arquivo de configuração do
hardeneks com isso é possível desabilitar algumas checagens da
ferramenta. Abaixo segue um arquivo de configuração onde pode ser
desabilitado algumas checagens:

``` bash
---
ignore-namespaces:
  - kube-node-lease
  - kube-public
  - kube-system
  - kube-apiserver
  - karpenter
  - kubecost
  - external-dns
  - argocd
  - aws-for-fluent-bit
  - amazon-cloudwatch
  - vpa
rules: 
  cluster_wide:
    security:
      iam:
        - disable_anonymous_access_for_cluster_roles
        - check_endpoint_public_access
        - check_aws_node_daemonset_service_account
        - check_access_to_instance_profile
        - restrict_wildcard_for_cluster_roles
      multi_tenancy:
        - ensure_namespace_quotas_exist
      detective_controls:
        - check_logs_are_enabled
      network_security:
        - check_vpc_flow_logs
        - check_awspca_exists
        - check_default_deny_policy_exists
      encryption_secrets:
        - use_encryption_with_ebs
        - use_encryption_with_efs
        - use_efs_access_points
      infrastructure_security:
        - deploy_workers_onto_private_subnets
        - make_sure_inspector_is_enabled
      pod_security:
        - ensure_namespace_psa_exist
      image_security:
        - use_immutable_tags_with_ecr
    reliability:
      applications:
        - check_metrics_server_is_running
        - check_vertical_pod_autoscaler_exists
  namespace_based:
    security: 
      iam:
        - disable_anonymous_access_for_roles
        - restrict_wildcard_for_roles
        - disable_service_account_token_mounts
        - disable_run_as_root_user
        - use_dedicated_service_accounts_for_each_deployment
        - use_dedicated_service_accounts_for_each_stateful_set
        - use_dedicated_service_accounts_for_each_daemon_set
      pod_security:
        - disallow_container_socket_mount
        - disallow_host_path_or_make_it_read_only
        - set_requests_limits_for_containers
        - disallow_privilege_escalation
        - check_read_only_root_file_system
      network_security:
        - use_encryption_with_aws_load_balancers
      encryption_secrets:
        - disallow_secrets_from_env_vars    
      runtime_security:
        - disallow_linux_capabilities
    reliability:
      applications:
        - check_horizontal_pod_autoscaling_exists
        - schedule_replicas_across_nodes
        - run_multiple_replicas
        - avoid_running_singleton_pods
```

Abaixo outras opções que a ferramenta possui:

- Executar a checagem em um determinado namespace;
- Executar a checagem em uma determinada região;
- Executar a checagem em um determinado contexto ou por nome de
    cluster;
- Exportar esse report para formato html ou txt;

Uma ideia de uso seria a possibilidade de executar essa ferramenta em
uma pipeline de CI/CD.

## Conclusão 

A ideia desse post foi demonstrar a utilização da ferramenta hardeneks
para efetuar uma checagens de boas práticas na utilização do kubernetes.


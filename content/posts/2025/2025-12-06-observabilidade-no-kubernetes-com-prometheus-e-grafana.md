---
title: "Observabilidade no Kubernetes com Prometheus e Grafana"
description: "Como instalar o kube-prometheus-stack com Helm para monitorar um cluster Kubernetes com Prometheus, Grafana e Alertmanager, com exemplos de PromQL e alertas."
image: "/images/kubernetes.jpg"
author: Jason
date: 2025-12-06
tags: [ "Kubernetes", "Observability", "Containers" ]
type: post
showTableOfContents: true
---

![kubernetes](/images/kubernetes.jpg)

O objetivo desse artigo é montar uma stack de monitoramento para um
cluster Kubernetes usando o **kube-prometheus-stack**, um Helm chart da
comunidade Prometheus que instala de uma vez o Prometheus, o Grafana, o
Alertmanager e os exportadores de métricas, já com dashboards e alertas
prontos.

## Observabilidade em poucas palavras

Observabilidade é a capacidade de entender o que está acontecendo dentro
de um sistema a partir dos dados que ele gera. Normalmente esses dados
são divididos em três pilares:

-   **Métricas**: valores numéricos ao longo do tempo, como uso de CPU,
    requisições por segundo e taxa de erro.
-   **Logs**: registros de eventos gerados pelas aplicações.
-   **Traces**: o caminho de uma requisição passando por vários serviços.

Nesse artigo vamos focar em **métricas**, que normalmente são o primeiro
passo e a base para os alertas.

## Pré Requisitos

-   Um cluster kubernetes. Para testes, um cluster local com o kind
    funciona bem; veja como criar nesse
    [link](/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind/).
-   kubectl e Helm 3 instalados.

## O que o kube-prometheus-stack instala

-   **Prometheus Operator**: gerencia o Prometheus e o Alertmanager através
    de recursos customizados (CRDs) como `ServiceMonitor` e
    `PrometheusRule`.
-   **Prometheus**: coleta e armazena as métricas.
-   **Alertmanager**: recebe os alertas do Prometheus e envia para e-mail,
    Slack etc.
-   **Grafana**: visualização, com dezenas de dashboards prontos.
-   **node-exporter**: um DaemonSet que expõe métricas dos nós (CPU,
    memória, disco).
-   **kube-state-metrics**: expõe o estado dos objetos do Kubernetes
    (pods, deployments, restarts etc.).

## Instalação

Adicione o repositório e instale o chart no namespace `monitoring`:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```

Acompanhe os pods até todos ficarem prontos:

```bash
kubectl get pods -n monitoring -w
```

Os nomes dos recursos usam o nome da release, `kube-prometheus-stack`.
Para ver os Services criados:

```bash
kubectl get svc -n monitoring
```

## Acessando o Grafana

O usuário é `admin` e a senha fica em um Secret criado pelo chart:

```bash
kubectl get secret -n monitoring kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d; echo
```

Faça o port-forward e acesse `http://localhost:3000`:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

Em **Dashboards** já existem vários painéis prontos. Alguns que valem a
pena olhar primeiro:

-   **Kubernetes / Compute Resources / Cluster**: visão geral de CPU e
    memória do cluster.
-   **Kubernetes / Compute Resources / Namespace (Pods)**: consumo por pod
    em um namespace.
-   **Node Exporter / Nodes**: detalhes de cada nó.

## Acessando o Prometheus

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Acesse `http://localhost:9090`. Em **Status → Targets** você vê tudo que
o Prometheus está coletando.

Em um cluster kind é normal alguns alvos do control plane aparecerem
como **down**, como o etcd, o scheduler, o controller-manager e o
kube-proxy. Nesses clusters esses componentes expõem métricas apenas em
`127.0.0.1` por padrão. Isso não afeta o restante do monitoramento.

## Algumas consultas PromQL

Na aba **Graph** do Prometheus (ou no **Explore** do Grafana) teste as
consultas abaixo.

Uso de CPU por pod em um namespace, em cores:

```promql
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="monitoring", container!=""}[5m]))
```

Memória em uso por pod:

```promql
sum by (pod) (container_memory_working_set_bytes{namespace="monitoring", container!=""})
```

Containers que reiniciaram na última hora (métrica do kube-state-metrics):

```promql
increase(kube_pod_container_status_restarts_total[1h]) > 0
```

Alvos que estão fora do ar:

```promql
up == 0
```

## Monitorando a sua aplicação com ServiceMonitor

Para o Prometheus coletar as métricas de uma aplicação sua, não é preciso
editar a configuração dele: basta criar um `ServiceMonitor`. Supondo
uma aplicação com um Service com o label `app: minha-app` e uma porta
chamada `http-metrics` que expõe `/metrics`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: minha-app
  namespace: app
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: minha-app
  endpoints:
  - port: http-metrics
    path: /metrics
    interval: 30s
```

O label `release: kube-prometheus-stack` é **obrigatório** nessa
instalação: por padrão o Prometheus só considera ServiceMonitors que
tenham o label da release do Helm. Esse é um dos erros mais comuns: o
ServiceMonitor é criado e as métricas nunca aparecem.

## Criando um alerta

Os alertas são definidos com o recurso `PrometheusRule`, que segue a
mesma regra do label. O exemplo abaixo dispara quando um container
reinicia mais de 3 vezes em 15 minutos:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: alertas-app
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
  - name: app.rules
    rules:
    - alert: ContainerReiniciandoMuito
      expr: increase(kube_pod_container_status_restarts_total[15m]) > 3
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Container {{ $labels.container }} reiniciando no pod {{ $labels.pod }}"
        description: "O container reiniciou mais de 3 vezes nos últimos 15 minutos."
```

```bash
kubectl apply -f alertas-app.yaml
```

A regra aparece em **Alerts** no Prometheus. Para receber as
notificações, configure um receiver (Slack, e-mail etc.) no Alertmanager
através dos values do chart.

Se esse alerta disparar, uma das causas comuns é uma liveness probe mal
configurada. Veja mais no artigo sobre
[probes no Kubernetes](/posts/2025/2025-02-15-kubernetes-configurando-liveness-readiness-e-startup-probes/).

## Limpeza

```bash
helm uninstall kube-prometheus-stack -n monitoring
kubectl delete namespace monitoring
```

O Helm **não remove as CRDs** do chart (`servicemonitors`,
`prometheusrules` etc.). Se quiser removê-las também:

```bash
kubectl get crd -o name | grep monitoring.coreos.com | xargs kubectl delete
```

## Conclusão

Com um único Helm chart temos uma stack completa de métricas, com
dashboards e alertas prontos para o cluster. A partir daqui os próximos
passos são criar ServiceMonitors para as suas aplicações, configurar o
Alertmanager para enviar notificações e evoluir para os outros pilares,
como logs (Loki, por exemplo) e traces (OpenTelemetry).

## Referências

-   [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
-   [Prometheus Operator](https://prometheus-operator.dev/)
-   [Consultas PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/)

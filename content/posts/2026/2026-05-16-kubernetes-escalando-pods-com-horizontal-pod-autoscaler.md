---
title: "Kubernetes: escalando pods automaticamente com o Horizontal Pod Autoscaler"
description: "Como configurar o Horizontal Pod Autoscaler (HPA) no Kubernetes com o metrics-server, gerar carga e ver os pods escalando, usando um cluster local com kind."
image: "/images/kubernetes.jpg"
author: Jason
date: 2026-05-16
tags: [ "Kubernetes", "Containers" ]
type: post
showTableOfContents: true
---

![kubernetes](/images/kubernetes.jpg)

O objetivo desse artigo é demonstrar como usar o **Horizontal Pod
Autoscaler (HPA)** para aumentar e diminuir automaticamente a quantidade
de pods de uma aplicação de acordo com o uso de CPU. Vamos gerar carga
na aplicação e acompanhar o HPA criando novos pods e, depois, removendo
os pods quando a carga acaba.

Para os testes vamos usar um cluster local com o kind. Se precisar, veja
como criar um nesse
[link](/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind/).

## Pré Requisitos

-   Ter um cluster kubernetes, no nosso caso utilizaremos o kind.
-   Ter o kubectl instalado.

## Como o HPA funciona

O HPA é um controlador que, a cada 15 segundos por padrão, consulta as
métricas dos pods e calcula quantas réplicas são necessárias para atingir
o objetivo configurado. A fórmula é:

```
réplicas desejadas = ceil(réplicas atuais × (valor atual da métrica / valor desejado))
```

Por exemplo: com 2 réplicas usando em média 100% de CPU e um objetivo de
50%, o HPA calcula `ceil(2 × 100 / 50) = 4` réplicas.

Dois pontos importantes:

-   O uso de CPU é calculado **em relação ao `requests`** do container.
    Sem `requests` de CPU definido, o HPA não consegue calcular a
    porcentagem.
-   As métricas vêm do **metrics-server**, que não vem instalado no kind.

## Instalando o metrics-server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

No kind os certificados do kubelet não são assinados por uma CA que o
metrics-server reconheça. Para ambientes de teste, adicionamos a flag
`--kubelet-insecure-tls`:

```bash
kubectl patch deployment metrics-server -n kube-system --type=json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

**Não use essa flag em produção.** Em clusters gerenciados, como o EKS,
o metrics-server pode ser instalado sem ela.

Aguarde o pod ficar pronto e teste:

```bash
kubectl top nodes
```

Se o comando mostrar o uso de CPU e memória dos nós, o metrics-server
está funcionando.

## Subindo a aplicação

Vamos usar a imagem `hpa-example`, da documentação do Kubernetes, que
executa um cálculo pesado de CPU a cada requisição. Crie um arquivo
chamado app.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
          limits:
            cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  selector:
    run: php-apache
  ports:
  - port: 80
```

```bash
kubectl apply -f app.yaml
```

## Criando o HPA

O jeito mais rápido é pelo comando `kubectl autoscale`:

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

Mas o ideal é manter o HPA versionado em YAML. O equivalente, usando a
API `autoscaling/v2`, é:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

Verifique o HPA:

```bash
kubectl get hpa php-apache
```

A coluna **TARGETS** mostra o uso atual em relação ao objetivo, por
exemplo `cpu: 0%/50%`. Logo depois da criação pode aparecer
`<unknown>` por alguns segundos, até a primeira coleta de métricas.

## Gerando carga

Em outro terminal, suba um pod que faz requisições sem parar para a
aplicação:

```bash
kubectl run -i --tty load-generator --rm --image=busybox:1.36 --restart=Never \
  -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

No primeiro terminal, acompanhe o HPA:

```bash
kubectl get hpa php-apache --watch
```

Em cerca de um minuto o uso de CPU passa bastante dos 50% e o HPA
começa a aumentar a quantidade de réplicas. Você pode ver os pods novos
com:

```bash
kubectl get pods -l run=php-apache
```

## Parando a carga

Pare o gerador de carga com `Ctrl + C`. O uso de CPU cai rapidamente,
mas as réplicas **não diminuem na hora**. Por padrão o HPA espera uma
janela de estabilização de **5 minutos** antes de reduzir, para evitar
que a quantidade de pods fique subindo e descendo a cada oscilação de
carga.

Esse comportamento pode ser ajustado no campo `behavior` do HPA. Por
exemplo, para reduzir no máximo 1 pod por minuto, depois de 5 minutos de
estabilidade:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
```

## Boas práticas

-   **Defina `requests` realistas.** Como a porcentagem é calculada sobre
    o `requests`, um valor muito baixo faz o HPA escalar cedo demais, e
    um valor muito alto faz ele demorar para reagir.
-   **Não defina `replicas` no Deployment quando usar HPA com GitOps.**
    Se o ArgoCD ou o `kubectl apply` reaplicarem o número de réplicas do
    YAML, eles vão brigar com o HPA. Veja mais sobre GitOps no artigo do
    [ArgoCD](/posts/2024/2024-06-09-gerenciamento-de-infraestrutura-com-gitops-e-argocd/).
-   **Configure readiness probes.** Os pods novos só devem receber
    tráfego quando estiverem prontos. Veja o artigo sobre
    [probes](/posts/2025/2025-02-15-kubernetes-configurando-liveness-readiness-e-startup-probes/).
-   **Lembre dos nós.** O HPA cria pods, mas se não houver espaço no
    cluster eles ficam em `Pending`. Em produção, combine o HPA com um
    autoscaler de nós, como o Cluster Autoscaler ou o Karpenter no EKS.

## Limpeza

```bash
kubectl delete hpa php-apache
kubectl delete -f app.yaml
```

## Conclusão

O HPA é a forma mais simples de fazer uma aplicação acompanhar a
demanda no Kubernetes: mais pods quando a carga sobe e menos pods quando
ela cai, economizando recursos. Com o metrics-server instalado e os
`requests` bem definidos, a configuração básica é feita com poucas
linhas de YAML.

## Referências

-   [Horizontal Pod Autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
-   [HorizontalPodAutoscaler Walkthrough](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
-   [metrics-server](https://github.com/kubernetes-sigs/metrics-server)

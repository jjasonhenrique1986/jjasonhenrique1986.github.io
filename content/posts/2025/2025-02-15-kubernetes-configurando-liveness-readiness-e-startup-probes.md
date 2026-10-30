---
title: "Kubernetes: configurando Liveness, Readiness e Startup Probes"
description: "Entenda a diferença entre Liveness, Readiness e Startup Probes no Kubernetes e veja exemplos práticos de cada uma em um cluster local com kind."
image: "/images/kubernetes.jpg"
author: Jason
date: 2025-02-15
tags: [ "Kubernetes", "Containers" ]
type: post
showTableOfContents: true
---

![kubernetes](/images/kubernetes.jpg)

O objetivo desse artigo é explicar e demonstrar as **probes** do
Kubernetes: Liveness, Readiness e Startup. Elas são a forma que o
Kubernetes tem de saber se o seu container está saudável e se ele está
pronto para receber tráfego. Configurar bem as probes evita dois
problemas bem comuns: aplicação travada que nunca é reiniciada e
aplicação recebendo requisições antes de estar pronta.

Para os testes vamos usar um cluster local com o kind. Se ainda não tem
um, você pode seguir esse
[link](/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind/).

## Pré Requisitos

-   Ter um cluster kubernetes, no nosso caso utilizaremos o kind.
-   Ter o kubectl instalado.

Vamos criar um namespace para os testes:

```bash
kubectl create ns probes
```

## Conceitos

Quem executa as probes é o **kubelet**, em cada nó. Cada tipo de probe
tem uma consequência diferente quando falha:

| Probe | Pergunta que responde | O que acontece quando falha |
|---|---|---|
| **Liveness** | O container está vivo? | O kubelet **reinicia** o container |
| **Readiness** | O container está pronto para receber tráfego? | O pod é **removido dos endpoints** do Service (não reinicia) |
| **Startup** | A aplicação já terminou de iniciar? | Enquanto não passar, liveness e readiness ficam **desativadas**; se estourar o limite, o container é reiniciado |

As probes podem verificar o container de quatro formas:

-   **httpGet**: faz uma requisição HTTP; status entre 200 e 399 é sucesso.
-   **tcpSocket**: tenta abrir uma conexão TCP na porta.
-   **exec**: executa um comando dentro do container; exit code 0 é sucesso.
-   **grpc**: usa o protocolo de health check do gRPC.

E os principais parâmetros (com os valores padrão):

-   `initialDelaySeconds` (0): tempo de espera antes da primeira checagem.
-   `periodSeconds` (10): intervalo entre as checagens.
-   `timeoutSeconds` (1): tempo máximo de resposta.
-   `failureThreshold` (3): falhas seguidas para considerar a probe como falha.
-   `successThreshold` (1): sucessos seguidos para voltar a considerar
    sucesso. Para liveness e startup esse valor precisa ser 1.

## Liveness Probe

Para ver a liveness em ação vamos usar um container que cria um arquivo,
espera 30 segundos e apaga esse arquivo. A probe verifica se o arquivo
existe, então depois de 30 segundos ela começa a falhar.

Crie um arquivo chamado liveness.yaml com o conteúdo abaixo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec
  namespace: probes
spec:
  containers:
  - name: app
    image: busybox:1.36
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

Aplique o manifesto:

```bash
kubectl apply -f liveness.yaml
```

Depois de cerca de um minuto, verifique os eventos do pod:

```bash
kubectl describe pod liveness-exec -n probes
```

Nos eventos vai aparecer a mensagem `Liveness probe failed` e, em
seguida, que o container será reiniciado. Acompanhando o pod, a coluna
**RESTARTS** vai aumentando a cada ciclo:

```bash
kubectl get pod liveness-exec -n probes -w
```

## Readiness Probe

Agora vamos ver a readiness. Vamos subir um nginx com uma readiness
probe que consulta o caminho `/ready`, que ainda não existe. O nginx
responde 404, a probe falha e o pod nunca fica pronto.

Crie um arquivo chamado readiness.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: probes
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: probes
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f readiness.yaml
kubectl get pods -n probes -l app=web
```

O pod fica com status **Running**, mas a coluna READY mostra **0/1**.
Repare que ele **não é reiniciado**, apenas não recebe tráfego. Podemos
confirmar que o Service não tem nenhum endpoint pronto:

```bash
kubectl get endpointslices -n probes -l kubernetes.io/service-name=web
```

Agora vamos criar o arquivo que a probe procura:

```bash
kubectl exec -n probes deploy/web -- sh -c 'echo ok > /usr/share/nginx/html/ready'
```

Em poucos segundos o pod passa para **1/1** e o IP dele aparece no
EndpointSlice do Service. Esse é exatamente o comportamento que queremos
durante um deploy: o pod novo só recebe requisições quando estiver
pronto.

## Startup Probe

Algumas aplicações demoram para iniciar, por exemplo aplicações Java
que carregam muitas dependências. Se a liveness começar a checar antes
da aplicação subir, o container é reiniciado em loop.

A startup probe resolve isso: enquanto ela não tiver sucesso, as outras
probes não são executadas. No exemplo abaixo a aplicação tem até
`30 x 10 = 300` segundos para iniciar. Depois disso a liveness assume
com um intervalo menor:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-exemplo
  namespace: probes
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
    startupProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 10
```

## Boas práticas

-   **Liveness não deve depender de serviços externos.** Se o banco de
    dados cair e a liveness checar o banco, todos os pods serão
    reiniciados ao mesmo tempo, e isso não resolve o problema.
-   **Readiness pode checar dependências**, já que a consequência é
    apenas tirar o pod do balanceamento.
-   **Use startup probe para aplicações lentas** em vez de aumentar muito
    o `initialDelaySeconds` da liveness.
-   **Crie endpoints leves**, como `/healthz` e `/ready`, que respondam
    rápido e não façam processamento pesado.

## Limpeza

```bash
kubectl delete ns probes
```

## Conclusão

As probes são uma configuração simples que faz muita diferença na
disponibilidade da aplicação. A liveness garante que um container
travado seja reiniciado, a readiness garante que só pods prontos recebam
tráfego e a startup protege aplicações que demoram para subir.

Se quiser entender melhor os controladores que usam essas probes, veja
também o artigo sobre
[DaemonSet, Deployment e StatefulSet](/posts/2024/2024-10-12-kubernetes-entenda-daemonset-deployment-e-statefulset/).

## Referências

-   [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

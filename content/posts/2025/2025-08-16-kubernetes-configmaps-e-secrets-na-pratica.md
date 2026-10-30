---
title: "Kubernetes: ConfigMaps e Secrets na prática"
description: "Como separar configuração do código no Kubernetes usando ConfigMaps e Secrets: criação, uso como variáveis de ambiente e volumes, e cuidados de segurança."
image: "/images/kubernetes.jpg"
author: Jason
date: 2025-08-16
tags: [ "Kubernetes", "Containers", "Security" ]
type: post
showTableOfContents: true
---

![kubernetes](/images/kubernetes.jpg)

O objetivo desse artigo é demonstrar como usar **ConfigMaps** e
**Secrets** no Kubernetes. Eles permitem separar a configuração da
imagem do container: a mesma imagem pode rodar em desenvolvimento e em
produção mudando apenas a configuração.

Para os testes vamos usar um cluster local com o kind. Se precisar, veja
como criar um nesse
[link](/posts/2023/2023-11-15-criando-um-cluster-local-de-kubernetes-com-o-kind/).

## Pré Requisitos

-   Ter um cluster kubernetes, no nosso caso utilizaremos o kind.
-   Ter o kubectl instalado.

```bash
kubectl create ns config
```

## Conceitos

-   **ConfigMap**: guarda dados de configuração **não sensíveis**, como
    URLs, nível de log e arquivos de configuração.
-   **Secret**: guarda dados **sensíveis**, como senhas, tokens e
    certificados.

Um ponto importante: por padrão os Secrets são apenas codificados em
**base64**, e base64 **não é criptografia**. Qualquer pessoa com
permissão de leitura no Secret consegue ver o valor. Vamos voltar a esse
ponto no final.

Os dois podem ser consumidos pelo pod de duas formas: como **variáveis
de ambiente** ou como **arquivos montados em um volume**.

## Criando um ConfigMap

Podemos criar a partir de valores literais e de arquivos. Primeiro crie
um arquivo chamado app.properties:

```bash
cat > app.properties <<EOF
cache.ttl=300
feature.novo-checkout=true
EOF
```

Agora crie o ConfigMap com dois valores literais e o arquivo:

```bash
kubectl create configmap app-config -n config \
  --from-literal=APP_ENV=producao \
  --from-literal=LOG_LEVEL=info \
  --from-file=app.properties
```

Para ver o conteúdo:

```bash
kubectl get configmap app-config -n config -o yaml
```

O mesmo ConfigMap pode ser escrito em YAML, o que é o ideal para manter
versionado no Git:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: config
data:
  APP_ENV: producao
  LOG_LEVEL: info
  app.properties: |
    cache.ttl=300
    feature.novo-checkout=true
```

## Criando um Secret

```bash
kubectl create secret generic db-credentials -n config \
  --from-literal=DB_USER=app \
  --from-literal=DB_PASSWORD='S3nh@Forte'
```

Veja que o valor é apenas base64:

```bash
kubectl get secret db-credentials -n config -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
```

O comando acima mostra a senha em texto puro. Por isso **não versione
Secrets em YAML no Git** sem algum tipo de criptografia.

## Usando no pod

Crie um arquivo chamado pod.yaml. Nele usamos o ConfigMap de três
formas: todas as chaves como variáveis (`envFrom`), uma chave específica
do Secret como variável (`secretKeyRef`) e o arquivo `app.properties`
montado em um volume:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: config
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: DB_USER
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: DB_PASSWORD
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app
      readOnly: true
  volumes:
  - name: config-volume
    configMap:
      name: app-config
      items:
      - key: app.properties
        path: app.properties
```

```bash
kubectl apply -f pod.yaml
```

Validando as variáveis de ambiente e o arquivo:

```bash
kubectl exec -n config app -- env | grep -E 'APP_ENV|LOG_LEVEL|DB_USER'
kubectl exec -n config app -- cat /etc/app/app.properties
```

## Atualizando a configuração

Aqui existe uma diferença importante:

-   **Variáveis de ambiente não são atualizadas** em um pod que já está
    rodando. É preciso recriar o pod, por exemplo com
    `kubectl rollout restart deployment/<nome>` quando se usa um
    Deployment.
-   **Arquivos montados por volume são atualizados** automaticamente
    pelo kubelet depois de alguns segundos. Isso não vale quando o volume
    é montado com `subPath`.

Vamos testar alterando o arquivo no ConfigMap:

```bash
kubectl patch configmap app-config -n config --type merge \
  -p '{"data":{"app.properties":"cache.ttl=600\nfeature.novo-checkout=true\n"}}'
```

Aguarde um pouco e leia o arquivo de novo. O valor de `cache.ttl` vai
mudar para 600, mas a variável `LOG_LEVEL`, por exemplo, só mudaria
recriando o pod. A aplicação também precisa reler o arquivo para usar o
novo valor.

## ConfigMaps e Secrets imutáveis

Para configurações que não devem mudar, você pode marcar o objeto como
imutável. Isso evita alterações acidentais e reduz a carga no API
server, já que o kubelet deixa de observar mudanças:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
  namespace: config
data:
  APP_ENV: producao
immutable: true
```

Para mudar um objeto imutável é preciso apagar e criar de novo. Uma
prática comum é colocar a versão no nome (`app-config-v2`, `v3`...) e
atualizar o Deployment para apontar para o novo nome.

## Cuidados com Secrets

-   **Restrinja o acesso com RBAC.** Quem pode ler Secrets em um
    namespace consegue ver todas as senhas dele. Veja como configurar
    no artigo sobre
    [RBAC no Kubernetes](/posts/2024/2024-11-15-gerenciando-acesso-no-kubernetes-com-rbac/).
-   **Habilite a criptografia em repouso no etcd.** Em clusters
    gerenciados como o EKS, isso pode ser feito com chaves do AWS KMS.
-   **Considere um gerenciador de segredos externo**, como o AWS Secrets
    Manager, integrado ao cluster pelo External Secrets Operator ou pelo
    Secrets Store CSI Driver. Assim a fonte da verdade fica fora do
    cluster.
-   **Prefira volumes a variáveis de ambiente** para dados sensíveis,
    pois variáveis podem acabar aparecendo em logs e em dumps de erro.

## Limpeza

```bash
kubectl delete ns config
rm app.properties
```

## Conclusão

ConfigMaps e Secrets são a base para manter a configuração fora da
imagem. Use ConfigMap para dados comuns e Secret para dados sensíveis,
lembrando que o Secret sozinho não é criptografado: combine com RBAC,
criptografia no etcd e, se possível, um gerenciador de segredos externo.

## Referências

-   [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
-   [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)

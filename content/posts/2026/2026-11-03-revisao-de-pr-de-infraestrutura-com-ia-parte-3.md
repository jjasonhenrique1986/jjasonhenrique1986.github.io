---
title: "Revisão de PR de infraestrutura com IA, parte 3: a action e a segurança"
description: "Última parte da série: a action reutilizável do GitHub que roda checkov, Trivy, kube-linter e kubeconform, chama a IA no Amazon Bedrock e comenta no Pull Request, o resultado num PR real e os cuidados com forks, prompt injection e custo."
author: Jason
date: 2026-11-03
tags: [ "AWS", "CI/CD", "IA", "Security" ]
type: post
showTableOfContents: true
draft: false
---

Nas partes [1](/posts/2026/2026-11-01-revisao-de-pr-de-infraestrutura-com-ia-parte-1/)
e [2](/posts/2026/2026-11-02-revisao-de-pr-de-infraestrutura-com-ia-parte-2/)
criamos a role IAM e o script. Agora vamos empacotar tudo numa action do GitHub
e colocar para rodar a cada Pull Request.

## A action

Em vez de copiar o script e os passos do workflow para cada repositório, o
revisor virou uma **action composite** no repositório
[github-actions](https://github.com/jjasonhenrique1986/github-actions). O código
Python, os testes e os exemplos ficam na mesma pasta da action, então a versão
da action (`@v1`) define exatamente o código que roda.

No repositório que vai ser revisado, o workflow fica assim
(`.github/workflows/ai-review.yml`):

```yaml
name: Revisão de IaC com IA

on:
  pull_request:
    paths: ["**.tf", "**.tfvars", "**.yaml", "**.yml"]

permissions:
  id-token: write       # token OIDC para assumir a role na AWS
  contents: read
  pull-requests: write  # comentar no PR

concurrency:
  group: ai-review-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  review:
    # PRs de forks não recebem o token OIDC
    if: github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - uses: jjasonhenrique1986/github-actions/iac-ai-review@v1
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          model-id: ${{ vars.MODEL_ID }}
```

Pontos importantes:

- **`fetch-depth: 0`**: sem o histórico completo, o `git diff` contra a branch
  base não funciona. A action confere isso e falha com uma mensagem clara.
- **`concurrency`**: se você fizer vários pushes seguidos, só a última execução
  continua. Isso economiza chamadas ao modelo.
- **`MODEL_ID`**: variável do repositório que escolhe o modelo. Sem ela, o
  padrão é o `anthropic.claude-opus-5-5`. Nos meus testes ela está como
  `openai.gpt-oss-120b-1:0`, porque a cota do Claude na minha conta ainda está
  zerada (veja a [parte 1](/posts/2026/2026-11-01-revisao-de-pr-de-infraestrutura-com-ia-parte-1/)).
- **Nenhuma access key**: o único secret é o `AWS_ROLE_ARN`, e ele não é uma
  credencial. Fica como secret só para o ID da conta não aparecer no repositório
  nem nos logs. A action também mascara o ID da conta.
- **Uma role por repositório**: cada repositório que usa a action tem a sua
  própria role, presa ao `sub` dele, como na parte 1.

### Por dentro da action

O `action.yml` encadeia os mesmos passos que antes ficavam no workflow. Dois
detalhes valem destaque. O primeiro: antes de tocar na AWS, a action procura
mudanças de infraestrutura com os mesmos padrões do script:

```yaml
- name: Detectar mudanças de infraestrutura
  id: changes
  shell: bash
  env:
    BASE: ${{ steps.context.outputs.base }}
  run: |
    files="$(git diff --name-only "$BASE...HEAD" -- '*.tf' '*.tfvars' '*.yaml' '*.yml')"
    if [ -n "$files" ]; then
      echo "has-changes=true" >> "$GITHUB_OUTPUT"
    else
      echo "has-changes=false" >> "$GITHUB_OUTPUT"
    fi
```

Os scanners e as credenciais só rodam com `has-changes == 'true'`. Num PR que só
mexe em documentação, a action não assume a role, não chama o modelo e só
comenta que não há nada para revisar.

O segundo: o script roda da pasta da própria action (`github.action_path`), não
do repositório revisado. Assim o código do revisor não entra no diff nem nos
scanners de quem usa a action:

```yaml
- name: Revisar com IA
  id: review
  shell: bash
  env:
    DIR: ${{ github.action_path }}
    BASE: ${{ steps.context.outputs.base }}
    MODEL_ID: ${{ inputs.model-id }}
    AWS_REGION: ${{ inputs.aws-region }}
    STATUS_checkov: ${{ steps.checkov.outputs.status }}
    STATUS_trivy: ${{ steps.trivy.outputs.status }}
    STATUS_kube_linter: ${{ steps.kube-linter.outputs.status }}
    STATUS_kubeconform: ${{ steps.kubeconform.outputs.status }}
  run: |
    out="$RUNNER_TEMP/review.md"
    args=()
    for name in checkov trivy kube-linter kubeconform; do
      var="STATUS_${name//-/_}"
      case "${!var:-}" in
        ok) args+=(--scan "$name=$RUNNER_TEMP/scan-$name.json") ;;
        failed) args+=(--scan-failed "$name") ;;
      esac
    done
    "$RUNNER_TEMP/venv-reviewer/bin/python" "$DIR/reviewer/review.py" \
      --base "$BASE" "${args[@]}" --output "$out"
    echo "review-file=$out" >> "$GITHUB_OUTPUT"
```

Cada scanner roda num passo próprio e grava um relatório em JSON. O script
recebe um `--scan` por scanner que funcionou e um `--scan-failed` por scanner
que falhou, e o comentário avisa quais não rodaram. Um scanner fora do ar não
derruba a revisão.

Os outros passos são o `setup-python`, um venv para o revisor, outro para o
checkov (instalados juntos, o checkov rebaixou o `boto3` num teste meu), o
download do Trivy, do kube-linter e do kubeconform, que só rodam se o SHA-256 do
binário bater com o fixado na action, o `configure-aws-credentials` e o
`sticky-pull-request-comment`, que atualiza o mesmo comentário a cada push em vez
de encher o PR de comentários. Nenhum scanner quebra o job por encontrar
problemas: a ideia é passar os achados para o modelo.

A action também é testada no próprio repositório: `actionlint`, `pytest` (com
saídas reais de cada scanner), um job que confere que um PR sem infraestrutura
não pega credenciais, um que confere que um scanner desconhecido é recusado e um
que revisa um Terraform com SSH aberto e um manifest com `cluster-admin`,
chamando o modelo de verdade com os quatro scanners.

## Segurança

### PRs de forks

Por segurança, o GitHub **não entrega o token OIDC** para workflows disparados
por PRs de forks. A condição `if` pula o job nesses casos, em vez de deixar ele
falhar.

Não tente contornar isso trocando `pull_request` por `pull_request_target`.
Esse evento roda com as permissões do repositório base, e fazer checkout do
código do PR nele é uma forma conhecida de vazar credenciais.

### Prompt injection

Quem abre o PR controla o conteúdo do diff, e o diff vai para o modelo. As
defesas do projeto são em camadas:

1. O diff vai dentro de tags `<diff>`, e o prompt manda ignorar instruções que
   apareçam ali.
2. A role **só** pode chamar o modelo. Ela não tem acesso à infraestrutura.
3. O token do GitHub no job só **lê** o código e **comenta** no PR.
4. A IA **não aprova** o PR. No pior caso, um ataque bem-sucedido resulta num
   comentário errado, que a revisão humana deve pegar.

O que realmente protege são as camadas 2 a 4: limitar o que o job **pode
fazer**, em vez de confiar que o modelo sempre vai seguir o prompt.

### Dados

O diff e os relatórios dos scanners são enviados ao Bedrock, dentro da sua conta AWS.
Mesmo assim, confira a política da sua empresa antes de usar em repositórios
com dados sensíveis.

## Custo

O custo depende do tamanho do diff e do modelo. Algumas formas de controlar:

- O filtro `paths` do workflow evita rodar em PRs que não mexem em
  infraestrutura.
- O `concurrency` cancela execuções antigas.
- O `MAX_DIFF_CHARS` evita mandar PRs enormes.
- A variável `MODEL_ID` permite trocar de modelo, por exemplo do
  `openai.gpt-oss-120b-1:0` para o `openai.gpt-oss-20b-1:0`, que é menor, e
  comparar a qualidade das revisões.

Acompanhe o gasto no **AWS Cost Explorer** filtrando pelo serviço Bedrock.

## Testando

A pasta da action tem exemplos **propositalmente inseguros** em `examples/`. Não
aplique nenhum deles numa conta ou num cluster de verdade:

| Arquivo | Problemas |
|---|---|
| `terraform/main.tf` | Security Group com SSH aberto para `0.0.0.0/0` |
| `terraform/rds/main.tf` | RDS público, sem criptografia, senha no código e sem backup |
| `terraform/iam/main.tf` | usuário IAM com access key e política `Action: "*"` em `Resource: "*"` |
| `terraform/s3-publico/main.tf` | bloqueio de acesso público desligado e leitura para `Principal: "*"` |
| `terraform/eks/main.tf` | endpoint da API aberto para `0.0.0.0/0`, sem logs e sem KMS nos Secrets |
| `k8s/deployment.yaml` | imagem `latest`, senha em variável de ambiente, sem limites de recursos |
| `k8s/pod-privilegiado.yaml` | DaemonSet privilegiado, como root, com `hostNetwork`, `hostPID` e `/` do nó montado |
| `k8s/rbac-cluster-admin.yaml` | ServiceAccount de aplicação com `cluster-admin` |
| `k8s/banco-exposto.yaml` | PostgreSQL num `LoadBalancer` público e senha num ConfigMap |
| `k8s/ingress-sem-tls.yaml` | painel administrativo publicado por HTTP, sem TLS e sem autenticação |

### O resultado num PR real

Os exemplos novos entraram no
[PR #4](https://github.com/jjasonhenrique1986/iac-ai-reviewer/pull/4) do
repositório original do projeto, antes de ele virar a action. O workflow rodou com o `openai.gpt-oss-120b-1:0` e publicou um
comentário com risco geral **alto** e 12 achados. Um trecho:

> **Risco geral:** 🔴 alto
>
> ### 🔴 ClusterRoleBinding concedendo cluster-admin a ServiceAccount comum
> `examples/k8s/rbac-cluster-admin.yaml` · severidade **alta**
>
> A ServiceAccount `app` recebe o `ClusterRole` `cluster-admin`, concedendo
> permissões administrativas em todo o cluster, violando o princípio de menor
> privilégio.
>
> **Sugestão:** Crie papéis específicos com permissões necessárias e vincule
> apenas ao ServiceAccount. Remova o binding `cluster-admin`.
>
> ### 🔴 Ingress HTTP sem TLS nem autenticação
> `examples/k8s/ingress-sem-tls.yaml` · severidade **alta**
>
> O Ingress expõe o admin em HTTP puro (`host: admin.exemplo.com.br`) sem TLS
> nem camada de autenticação, facilitando ataques de interceptação e acesso não
> autorizado.

### O que a IA acrescenta ao checkov

Nos mesmos exemplos, o checkov falhou 49 checagens de Terraform e 59 de
Kubernetes. Mesmo assim, **nenhuma** delas aponta os dois achados acima: a
ServiceAccount com `cluster-admin` e o Ingress sem TLS. A IA pegou os dois no
PR e nas três execuções locais que fiz com o mesmo diff.

É aqui que a divisão de trabalho da parte 1 aparece: o checkov é exaustivo
dentro das regras que ele conhece, e a IA lê a intenção do recurso (um painel
`admin` na internet, uma aplicação comum com poder de administrador do cluster).

### Mais scanners: Trivy, kube-linter e kubeconform

O checkov não precisa ser o único. Rodei mais três ferramentas nos mesmos
exemplos para ver o que cada uma acrescenta:

| Risco | checkov | Trivy | kube-linter | IA |
|---|---|---|---|---|
| SSH aberto, S3 público, RDS público, EKS aberto | ✅ | ✅ | — | ✅ |
| Política IAM `*` em `*` | ✅ | ❌ | — | ✅ |
| Container privilegiado, `hostNetwork`, `hostPID` | ✅ | ✅ | ✅ | ✅ |
| ServiceAccount com `cluster-admin` | ❌ | ✅ | ✅ | ✅ |
| Senha num ConfigMap | ❌ | ✅ | ❌ | ✅ |
| PostgreSQL num `LoadBalancer` público | ❌ | ❌ | ✅ | ✅ |
| Ingress sem TLS | ❌ | ❌ | ❌ | ✅ |

- O **Trivy** (`trivy config`) complementa o checkov: pega o `cluster-admin` e a
  senha no ConfigMap, mas perde a política IAM com `*`, que o checkov pega bem.
- O **kube-linter** só acrescenta com duas regras além das padrão,
  `cluster-admin-role-binding` e `exposed-services`. Com todas as regras ligadas,
  ele fica ruidoso demais (ordem de chaves, label `owner`, mínimo de três réplicas).
- O **kubeconform** não procura riscos: ele valida os manifests contra o schema
  da API do Kubernetes e pega campo inexistente, tipo errado ou `apiVersion`
  removida, que os outros não pegam.
- O **Ingress sem TLS** nenhum scanner apontou. Só a IA.

Por isso a action roda os quatro por padrão. Dá para escolher pelo input
`scanners`, por exemplo `scanners: checkov,trivy` num repositório só de
Terraform. Um detalhe prático: a saída em texto do Trivy nos exemplos passou de
60 KB, e a do checkov de 30 KB. Mandar isso cru para o modelo gasta tokens e
chega perto do limite do diff, então o script resume cada relatório em uma linha
por achado (`arquivo:linha [SEVERIDADE] ID título`). Nos exemplos, os quatro
relatórios juntos ficaram em 16 mil caracteres.

### A IA também erra

No mesmo comentário, o achado do EKS diz que o endpoint público "permite que
qualquer pessoa na internet acesse a API do cluster e leia segredos". Isso é
exagero: o endpoint aberto para `0.0.0.0/0` aumenta muito a exposição, mas
quem chega nele ainda precisa se autenticar no IAM e ter permissão no RBAC do
cluster. É por isso que o rodapé de todo comentário diz que a IA é apoio, não
aprovação.

Para testar no seu repositório, configure a action, copie um desses arquivos e
abra um PR.

Se o seu manifest tem problemas de probes, o post
[Kubernetes: configurando liveness, readiness e startup probes](/posts/2025/2025-02-15-kubernetes-configurando-liveness-readiness-e-startup-probes/)
ajuda a corrigir.

## Conclusão

Com um script em Python empacotado numa action, cada PR de
infraestrutura ganha uma segunda leitura que explica os riscos em linguagem
simples, sem nenhuma credencial fixa e sem dar à IA o poder de aprovar nada.

O código completo, com a documentação da action, está em
[github-actions/iac-ai-review](https://github.com/jjasonhenrique1986/github-actions/tree/main/iac-ai-review).

## Referências

- [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [Creating a composite action](https://docs.github.com/en/actions/sharing-automations/creating-actions/creating-a-composite-action)
- [checkov](https://www.checkov.io/)
- [Trivy](https://trivy.dev/)
- [kube-linter](https://github.com/stackrox/kube-linter)
- [kubeconform](https://github.com/yannh/kubeconform)
- [marocchino/sticky-pull-request-comment](https://github.com/marocchino/sticky-pull-request-comment)

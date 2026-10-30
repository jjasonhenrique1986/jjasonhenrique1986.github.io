---
title: "Resenha do livro Site Reliability Engineering (SRE) do Google"
description: "Resenha do livro Site Reliability Engineering do Google: SLOs, error budget, toil, monitoramento, on-call e postmortems sem culpados."
author: Jason
date: 2026-03-14
tags: [ "SRE", "DevOps", "Livros", "Observability" ]
type: post
showTableOfContents: true
---

Depois da resenha do livro de
[Engenharia de Plataforma](/posts/2026/2026-01-31-livro-engenharia-de-plataforma/),
resolvi voltar a um clássico: o **Site Reliability Engineering: How
Google Runs Production Systems**, conhecido como "o livro de SRE do
Google". No Brasil ele foi publicado pela Novatec como **Engenharia de
Confiabilidade do Google**, e a versão em inglês está disponível de
graça no site [sre.google](https://sre.google/books/).

O livro foi lançado em 2016, mas continua muito atual. Muitos termos que
usamos hoje no dia a dia, como SLO, error budget e toil, ficaram
populares a partir dele.

Abaixo, compartilho os principais aprendizados.

---

## O que é SRE

O livro traz uma definição de Ben Treynor Sloss, que criou a área no
Google, que resume bem a ideia: SRE é o que acontece quando você pede
para um **engenheiro de software projetar uma equipe de operações**.

Na prática isso significa tratar operação como um problema de
engenharia: em vez de resolver tudo manualmente, o time escreve software
para automatizar, medir e melhorar a confiabilidade dos sistemas.

Um princípio que aparece logo no começo: o time de SRE deve limitar o
trabalho operacional a **no máximo 50% do tempo**. O restante deve ir
para projetos de engenharia que reduzam esse trabalho operacional no
futuro.

---

## Aceitar o risco: 100% não é a meta

Um dos pontos que mais muda a forma de pensar é a ideia de que **100% de
disponibilidade é o alvo errado**. Cada "nove" a mais custa muito caro,
e o usuário normalmente nem percebe a diferença, porque a rede, o
celular ou o provedor de internet dele falham mais do que isso.

Daí vem o conceito de **error budget**: se o objetivo é 99,9% de
disponibilidade, os 0,1% restantes são um orçamento que pode ser
"gasto" com mudanças, deploys e experimentos.

-   Sobrou orçamento? O time pode lançar funcionalidades com mais
    velocidade.
-   O orçamento acabou? As mudanças arriscadas param e o foco vai para a
    estabilidade.

O que eu acho mais interessante aqui é que o error budget resolve um
conflito clássico entre desenvolvimento (que quer lançar rápido) e
operação (que quer estabilidade) com um **número acordado**, e não com
opinião.

---

## SLI, SLO e SLA

O livro deixa bem clara a diferença entre esses três termos, que muita
gente usa como sinônimos:

-   **SLI** (Service Level Indicator): a métrica medida. Exemplo: a
    porcentagem de requisições respondidas com sucesso em menos de
    300 ms.
-   **SLO** (Service Level Objective): o objetivo para esse indicador.
    Exemplo: 99,9% das requisições em uma janela de 30 dias.
-   **SLA** (Service Level Agreement): o contrato com o cliente, com
    consequências (normalmente financeiras) se não for cumprido.

Uma dica prática do livro: o SLO interno deve ser **mais rígido** que o
SLA, para que o time perceba e reaja ao problema antes de quebrar o
contrato.

---

## Eliminando o toil

O livro dá nome a algo que todo mundo de operações conhece bem: o
**toil**. É o trabalho que é:

-   manual
-   repetitivo
-   automatizável
-   tático (reativo)
-   sem valor duradouro
-   que cresce na mesma proporção que o serviço

Nem todo trabalho operacional é toil, mas o toil em excesso é o que
impede o time de evoluir. Por isso a regra dos 50% de tempo para
engenharia é tão importante.

---

## Monitoramento e os quatro sinais de ouro

O capítulo de monitoramento de sistemas distribuídos apresenta os
**quatro sinais de ouro**, que até hoje são uma ótima referência para
começar a monitorar qualquer serviço:

-   **Latência**: o tempo para atender uma requisição, separando as de
    sucesso das que falharam.
-   **Tráfego**: a demanda sobre o sistema, como requisições por segundo.
-   **Erros**: a taxa de requisições que falham.
-   **Saturação**: o quanto o serviço está "cheio", como uso de CPU,
    memória ou fila.

O livro também insiste que **alerta deve ser acionável**: se alguém é
acordado de madrugada, deve ser por algo que exige ação humana imediata.
Alertas que ninguém consegue resolver só geram cansaço e fazem o time
ignorar os alertas importantes.

Quem quiser colocar esses conceitos em prática no Kubernetes pode
começar pelo artigo sobre
[Prometheus e Grafana](/posts/2025/2025-12-06-observabilidade-no-kubernetes-com-prometheus-e-grafana/).

---

## On-call e resposta a incidentes

Os capítulos sobre plantão (on-call) são bem humanos. O livro defende
que o on-call precisa ser **sustentável**: uma quantidade limitada de
incidentes por turno, tempo para analisar cada um com calma e
compensação pelo esforço.

Na gestão de incidentes, o ponto principal é ter **papéis claros**
durante a crise, como quem coordena o incidente, quem atua na correção e
quem cuida da comunicação. Isso evita aquela situação em que todo mundo
mexe em tudo ao mesmo tempo e ninguém sabe o estado real do problema.

---

## Postmortem sem culpados

Talvez o capítulo mais importante do ponto de vista cultural seja o de
**postmortem blameless** (sem culpados). Depois de um incidente, o
objetivo é entender **o que** permitiu que a falha acontecesse e o que
pode ser melhorado no sistema e nos processos, e não encontrar **quem**
errou.

Se as pessoas têm medo de serem punidas, elas escondem informações, e o
mesmo problema volta a acontecer. Um bom postmortem tem linha do tempo,
impacto, causa raiz e, principalmente, **ações de melhoria com dono e
prazo**.

---

## Automação e simplicidade

O livro também tem capítulos sobre automação, engenharia de releases e
simplicidade. Duas ideias que ficaram comigo:

-   **Automação** não serve só para economizar tempo: ela traz
    consistência e permite que o sistema escale sem crescer o time na
    mesma proporção.
-   **Simplicidade** é uma característica de confiabilidade. Cada linha
    de código e cada funcionalidade nova é algo a mais que pode quebrar.
    Remover código que não é mais usado também é trabalho de engenharia.

---

## Conclusão

O **Site Reliability Engineering** não é um livro para ler de uma vez do
começo ao fim. Ele é uma coleção de capítulos relativamente
independentes, e dá para começar pelos temas que fazem mais sentido para
o seu momento. Para quem está começando, recomendo os capítulos sobre
risco, SLOs, toil, monitoramento e postmortems.

Mesmo que a sua empresa não tenha um time chamado SRE, os conceitos se
aplicam a qualquer time de **DevOps, plataforma ou infraestrutura**.
Junto com o livro de Engenharia de Plataforma, ele forma uma base muito
boa para pensar em como entregar rápido sem abrir mão da confiabilidade.

Também vale conhecer os outros livros do Google sobre o tema,
disponíveis no mesmo site: **The Site Reliability Workbook**, mais
prático, e **Building Secure and Reliable Systems**.

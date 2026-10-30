---
title: "Resenha do livro Engenharia de Plataforma"
description: "Resenha do livro Engenharia de Plataforma: por que tratar a plataforma interna como um produto e os principais aprendizados."
image: "/images/eng-plataforma.jpg"
author: Jason
date: 2026-01-31
tags: [ "Engenharia de Plataforma", "DevOps", "Livros" ]
type: post
showTableOfContents: true
---

![eng-plataforma](/images/eng-plataforma.jpg)

Recentemente li o engenharia de plataforma conforme esse [link](https://www.amazon.com.br/Engenharia-Plataforma-l%C3%ADderes-t%C3%A9cnicos-produtos/dp/8575229419) e resolvi escrever uma resenha sobre o livro.

Se você trabalha com tecnologia há alguns anos, provavelmente já sentiu isso: times cada vez mais rápidos, sistemas cada vez mais complexos e uma sensação constante de que estamos gastando mais energia fazendo o básico funcionar do que criando valor de verdade.

É exatamente nesse cenário que o livro **Engenharia de Plataforma: um guia para líderes técnicos, de produtos e de pessoas** se encaixa.

Mais do que falar de ferramentas, o livro propõe uma **mudança de mentalidade**: tratar a plataforma interna como um produto, com usuários claros (desenvolvedores), objetivos bem definidos e evolução contínua. Os autores conectam tecnologia, organização e liderança de uma forma muito prática.

Abaixo, compartilho um panorama com os principais aprendizados.

---

## O começo: entendendo o problema

Nos primeiros capítulos, o livro faz algo essencial: explica por que **engenharia de plataforma** existe. Não como moda, mas como resposta direta ao crescimento da complexidade técnica.

Times independentes, microserviços, cloud, CI/CD e segurança criaram um ambiente poderoso — mas difícil de operar sem estrutura.

Aqui fica claro que **plataforma não é só infraestrutura**, nem sinônimo de DevOps. Ela surge para:

- reduzir fricção  
- padronizar o que faz sentido  
- permitir que os times foquem no domínio do negócio  

---

## Plataforma como produto (e não como suporte)

Um dos pontos mais fortes do livro é a defesa da **plataforma como produto interno**. Isso muda completamente a forma de pensar prioridades e sucesso.

Nos capítulos seguintes, os autores reforçam que uma plataforma só é bem-sucedida se:

- resolve problemas reais dos desenvolvedores  
- tem roadmap, prioridades e métricas  
- evolui com base em feedback constante  

Tratar a plataforma como algo “que todo mundo tem que usar” é receita para fracasso.  
Tratar como produto é o caminho para **adoção natural**.

---

## O desenvolvedor no centro

Vários capítulos giram em torno da **Developer Experience (DX)**. A mensagem é simples e poderosa: se a experiência do desenvolvedor é ruim, a entrega sofre — e a empresa também.

O livro fala bastante sobre:

- self-service de ambientes  
- automação bem feita  
- documentação clara  
- redução de tarefas manuais e repetitivas  

Tudo isso não como luxo, mas como **estratégia de escala**.

---

## Organização, times e interação

Outro bloco importante do livro discute como estruturar **times de plataforma**. Não existe um modelo único, mas existem princípios claros:

- evitar virar gargalo  
- ter limites bem definidos de responsabilidade  
- manter uma relação saudável com os times de produto  

O livro também aborda como plataformas se comunicam com seus consumidores por meio de:

- APIs  
- contratos  
- abstrações bem definidas  

Tudo isso ajuda a reduzir **dependências invisíveis** e conflitos entre times.

---

## Matriz de partes interessadas: alinhamento antes de escala

Um capítulo extremamente prático do livro apresenta a **matriz de partes interessadas (stakeholders)**. A ideia é simples, mas poderosa: identificar quem são os envolvidos com a plataforma, qual o nível de interesse de cada um e quanto poder de influência possuem.

Essa matriz ajuda a responder perguntas fundamentais como:

- quem precisa ser envolvido nas decisões da plataforma  
- quem precisa ser constantemente informado  
- quem pode bloquear ou acelerar a adoção  

Usar essa ferramenta evita um erro comum em iniciativas de plataforma: **construir algo tecnicamente excelente, mas politicamente inviável**. Ela força líderes a pensarem em comunicação, alinhamento e expectativa desde o início.

---

## As 5 vitórias e os 2 desafios da engenharia de plataforma

O livro também apresenta um modelo muito didático ao falar das **5 vitórias** que uma boa plataforma entrega, ao mesmo tempo em que reconhece **2 grandes desafios** inevitáveis.O modelo das 5 vitórias e 2 desafios não é uma checklist técnica. Ele é um framework de comunicação e alinhamento.

A ideia central é simples: A engenharia de plataforma só se sustenta se gera valor visível para múltiplos públicos, ao mesmo tempo em que reconhece tensões inevitáveis. Abaixo segue um exemplo desse framework:

### As 5 vitórias

- aumento da velocidade de entrega  
- melhor experiência do desenvolvedor  
- maior consistência e padronização  
- segurança e governança embutidas  
- redução de custo operacional e retrabalho  

Essas vitórias ajudam a criar **narrativas claras de valor** para liderança, produto e engenharia.

### Os 2 desafios

- risco de centralização excessiva  
- necessidade constante de alinhamento com múltiplos times  

O livro não romantiza a plataforma: ele deixa claro que o sucesso depende de decisões conscientes para não virar gargalo nem torre de marfim.

---

## Segurança e governança sem travar o time

Um ponto muito bem equilibrado do livro é a forma como trata **segurança e governança**. Em vez de controles pesados e processos lentos, a proposta é criar **guardrails**: regras embutidas na própria plataforma.

Assim, os times continuam autônomos, mas dentro de limites seguros e alinhados com:

- compliance  
- boas práticas  
- padrões organizacionais  

Segurança deixa de ser um bloqueio e passa a ser **parte natural do fluxo de entrega**.

---

## Rearquitetura em vez de V2

Um dos capítulos mais maduros do livro aborda a ideia de **rearquitetar em vez de criar uma V2 do zero**. A provocação é direta: V2s costumam falhar porque tentam resolver tudo de uma vez, ignorando usuários atuais e custos de transição.

A proposta do livro é:

- evoluir a plataforma de forma incremental  
- reduzir riscos e big bangs  
- permitir aprendizado contínuo  
- respeitar o que já funciona  

Essa visão conversa muito bem com princípios de produto, arquitetura evolutiva e plataformas como sistemas vivos — não projetos fechados.

---

## Medindo sucesso de verdade

Não dá para melhorar o que não se mede. Um dos capítulos é dedicado às **métricas de plataforma**, indo além de uptime ou custo de infraestrutura.

O livro fala de métricas como:

- adoção da plataforma  
- satisfação dos desenvolvedores  
- impacto na velocidade de entrega  
- redução de incidentes e retrabalho  

Esses indicadores ajudam a justificar investimentos e orientar decisões estratégicas.

---

## Escala, desafios e armadilhas

Nos capítulos finais, o livro entra em um território muito realista: **o que pode dar errado**.

Algumas armadilhas comuns citadas:

- centralização excessiva  
- plataformas genéricas demais  
- falta de visão de produto  
- desconexão com os times  

O fechamento olha para o futuro da engenharia de plataforma e reforça que ela **não é um projeto com fim**, mas uma jornada contínua de aprendizado e adaptação.

---

## Conclusão

**Engenharia de Plataforma** não é um livro sobre ferramentas específicas. Ele é sobre como criar bases sólidas para que times consigam entregar melhor, com menos fricção e mais autonomia.

Se você é líder técnico ou atua com **DevOps, SRE, arquitetura ou produto**, esse livro ajuda a dar nome a problemas que muita gente já sente no dia a dia — e aponta caminhos práticos, realistas e sustentáveis para resolvê-los.

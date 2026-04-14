---
title: "Como o Rovo Agent ajuda analistas de suporte a resolver tickets mais rápido"
date: 2026-01-23T00:00:00-03:00
draft: false
translationKey: "rovo-agent-support-analyst"
tags: ["jira-automation", "rovo", "support-analyst"]
---

A lentidão na resolução de tickets é um problema comum em muitas empresas.

Normalmente, o ticket chega primeiro ao time de Suporte (N1). Depois disso, ele costuma ser escalado para um time técnico ou de desenvolvimento para triagem. Em muitos casos, o ticket volta para o N1 porque as informações estão incompletas, já existe um caso parecido em andamento, ou a base de conhecimento já tem conteúdo suficiente para resolver o problema sem precisar de escalonamento técnico.

Neste artigo, eu mostro como usar o **Rovo Agent**, a capacidade de IA da Atlassian, para ajudar analistas de suporte a entender melhor o ticket, classificar a demanda, levantar hipóteses, buscar conteúdo na base de conhecimento, encontrar tickets similares e decidir quando o escalonamento realmente faz sentido.

---

## A dor

Alguns problemas comuns são:

- tickets escalados com informação incompleta
- problemas duplicados sendo analisados novamente pelo time técnico
- conteúdo já existente na base de conhecimento não sendo aproveitado
- tickets anteriores com o mesmo problema não sendo reutilizados

---

## O objetivo

Um bom resultado se parece com isto:

- uma triagem mais completa antes do escalonamento
- mais informação útil coletada para o time técnico
- conteúdo relevante da base de conhecimento aparecendo mais cedo
- hipóteses úteis levantadas antes do escalonamento
- menos vai e volta entre N1 e time técnico

---

## O que o Rovo muda na prática

Sem esse tipo de apoio, o N1 muitas vezes escala cedo demais. Depois, o time técnico pede mais dados, o ticket volta, e a resolução demora mais.

Com o Rovo Agent, o analista pode receber uma resposta mais estruturada antes de escalar:

- quais dados ainda faltam
- quais verificações já podem ser feitas
- quais páginas da base de conhecimento podem ajudar
- quais tickets parecidos podem ser relevantes
- quando o escalonamento realmente faz sentido

Um bom agente ajuda a reduzir:

- comentários de vai e volta
- investigações duplicadas
- tempo até a primeira ação realmente útil

---

## Como criar o seu primeiro agente

Você pode criar um Rovo Agent pela lateral do ambiente Atlassian, na parte relacionada a apps Atlassian e Studio.

Um fluxo simples de implementação fica assim:

1. Criar o seu Rovo Agent
2. Definir o propósito e as instruções dele
3. Ir para o Jira Automation
4. Criar uma regra que usa o agente
5. Gravar o resultado em comentário ou campo

Uma estrutura simples de automação é:

```text
Gatilho -> Use Rovo Agent -> Ação usando {{agentResponse}}
```

Um exemplo prático:

- **Gatilho:** Issue Created em um projeto JSM
- **Ação:** Use Rovo Agent
- **Ação:** adicionar comentário interno com `{{agentResponse}}`

O prompt é uma das partes mais importantes, porque ele define como o agente deve se comportar, quais fontes deve priorizar e como a resposta deve ser estruturada.

---

## Exemplo de prompt

```text
Assistente interno genérico de suporte

Você é um assistente interno de suporte para [Nome da Empresa].
Você NUNCA escreve para o cliente final. Você responde apenas ao analista do ticket, com orientação prática de triagem e resolução.

Fontes (ordem obrigatória)

Base de conhecimento / playbook: espaço “[Playbook / Runbook de Suporte]” (procedimentos oficiais)

Sistema de tickets: tickets semelhantes do mesmo projeto ou serviço (mesmo tema, erro, cliente ou provedor)

Tarefa

Quando você receber um ticket, faça o seguinte:

Leia o ticket completo: resumo, descrição, campos (cliente/marca, categoria, impacto, quantidade afetada, SLA, squad/time), comentários e qualquer anexo, log ou print referenciado.

Entenda e classifique a solicitação: Incidente / Dúvida / Requisição / Bug. Se não estiver claro, marque como Hipótese.

Pesquise no Playbook usando palavras-chave do ticket (produto, provedor, código de erro, termos de fluxo como “saque”, “cashout”, “verificação facial”, “liveness”, “V001” etc.). Extraia checklists, passos, critérios e evidências necessárias.

Pesquise tickets semelhantes no mesmo projeto e reaproveite apenas:

perguntas que ajudaram a destravar o caso

validações e verificações já realizadas

caminhos de resolução que realmente funcionaram

Não invente políticas, prazos, causas raiz, limitações ou compromissos.

Se faltarem dados, declare a Lacuna e diga exatamente o que deve ser pedido ou verificado.

Regras de saída

Seja breve: no máximo 12 a 15 linhas, em bullets, sem parágrafos longos.

Sempre priorize primeiro o Playbook; só depois use tickets semelhantes.

Se não houver base confiável, escreva “Sem base confiável” e recomende escalonamento.

Regra obrigatória de links em “Referências”

No bloco 5) Referências, toda referência deve ser um link clicável em Markdown.

Playbook: use a URL retornada pela busca (URL da página).

Tickets: use a URL retornada pela busca (URL do ticket).

Proibido: inventar URLs.

Se a busca não retornar URL, escreva: “(a busca não retornou link)” e inclua as palavras-chave usadas.

Formato de link (obrigatório)

Playbook: Playbook: <título da página>
 — <por que isso importa em 1 linha>

Ticket: <ID-DO-TICKET>
 — <o que foi útil em 1 linha>

Formato obrigatório da resposta (sempre exatamente estes blocos)

Resumo (1 frase)

…

Dados / evidências a coletar

…

Verificações imediatas (passo a passo)

…
…
…

Hipóteses prováveis (marcar como hipótese)

Hipótese: …

Hipótese: …

Referências

Playbook: …

Ticket: …

Escalonar quando

Critério objetivo: … → Time sugerido: …

Critério objetivo: … → Time sugerido: …
```

---

## Exemplo de automação

Um fluxo simples:

- **Gatilho:** Issue Created (projeto JSM)
- **Condição (opcional):** somente quando `Request type` for X ou `Labels` contiver Y
- **Ação:** Use Rovo Agent
- **Ação:** adicionar comentário interno com `{{agentResponse}}`

Dica: você também pode salvar o resultado em um campo customizado se quiser usar isso para relatórios ou reaproveitamento depois.

---

## Como é uma boa resposta do agente

Uma boa resposta normalmente contém:

- um resumo curto
- exatamente quais dados estão faltando
- verificações passo a passo que o N1 já pode fazer
- algumas hipóteses, claramente marcadas como hipóteses
- referências para páginas da base de conhecimento e tickets similares
- critérios claros de escalonamento e sugestão de time

---

## Checklist antes de colocar em produção

Antes de colocar a regra em produção, vale revisar:

- [ ] A regra sai com segurança se for disparada duas vezes?
- [ ] Ela evita repetir operações de criação?
- [ ] As falhas ficam registradas em algum lugar, como property, comentário ou campo?
- [ ] Existe timeout ou proteção contra execuções travadas?
- [ ] Os retries têm limite claro?

---

## Opcional: adicionar lock e idempotência

Se a regra puder ser disparada manualmente ou puder rodar mais de uma vez, vale proteger isso.

Duas ideias úteis são:

- **Lock por issue** — guardar um estado de “em andamento”
- **Chave de idempotência** — guardar que aquela ação já aconteceu

Mesmo um padrão simples de lock e idempotência já remove muitos problemas de execução duplicada.

---

## Considerações finais

O Rovo Agent pode ajudar analistas de suporte a tomar decisões melhores antes do escalonamento.

Isso significa triagem melhor, uso mais inteligente da base de conhecimento, menos investigação duplicada e caminhos mais rápidos para resolução.

Se a ideia não é só automatizar, mas melhorar a qualidade do tratamento dos tickets, esse é um caso de uso bem prático.

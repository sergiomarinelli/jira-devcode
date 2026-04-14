---
title: "Advanced Branching no Jira Automation: por que usar, quando usar e como aplicar"
date: 2026-02-20T00:00:00-03:00
draft: false
translationKey: "advanced-branching-jira-automation"
tags: ["jira-automation", "advanced-branching", "smart-values", "watchers", "jsm"]
---

Se você trabalha bastante com Jira Automation, provavelmente já usa laços de repetição de várias formas.

Na maior parte do tempo, isso significa percorrer issues: subtasks, issues ligadas, parent, child ou até resultados vindos de JQL. Em outros casos, objetos do Assets também ajudam bastante quando a ideia é aplicar ações em massa.

Mas em algum momento aparece um tipo diferente de necessidade: você precisa percorrer algo que não é uma issue e também não é um objeto do Assets.

Um exemplo comum é quando você precisa trabalhar com watchers, request participants, fixVersions ou organizations. Nesses casos, as opções mais tradicionais de branching não resolvem tão bem.

É aí que o **Advanced Branching** entra.

Ele permite que o Jira Automation percorra smart values que retornam listas, para que você consiga aplicar ações em cada item dessa lista.

Neste artigo, eu mostro qual problema ele resolve, quando ele faz mais sentido do que o branch normal e como usar isso na prática.

---

## O problema

No Jira Automation, as opções mais comuns para ações em massa costumam ser:

- branch em issues relacionadas
- iteração sobre resultados de JQL
- trabalho com objetos do Assets via AQL

Essas opções resolvem muitos cenários reais.

Mas existem situações em que o dado que você precisa percorrer não é uma coleção de issues. Em vez disso, ele vem de um smart value que retorna uma lista.

Por exemplo:

- watchers de uma issue
- fixVersions
- organizations no Jira Service Management
- request participants

Nesses casos, o **Advanced Branching** passa a ser a ferramenta certa.

Ele permite definir um smart value como lista de origem, dar um nome para cada item e depois usar essa variável dentro das ações que ficam no branch.

---

## Branch normal vs Advanced Branching

À primeira vista, os dois podem parecer parecidos, mas eles foram feitos para cenários diferentes.

### Use o branch normal quando o Jira já te entrega um contexto de issue

Exemplos típicos:

- para cada subtask
- para issues ligadas
- para a parent issue
- para issues retornadas por JQL

Isso funciona bem quando o Jira já está trabalhando com issues como entidade principal.

### Use o Advanced Branching quando a origem é uma lista vinda de smart value

Exemplos típicos:

- `{{issue.watchers}}`
- `{{issue.fixVersions}}`
- `{{issue.organizations}}`

Essa é a principal diferença: **o Advanced Branching não é sobre relacionamento entre issues. Ele é sobre percorrer listas vindas de smart values.**

---

## Como usar o Advanced Branching na prática

Você encontra o **Advanced Branching** dentro do Jira Automation ao montar uma regra.

Depois de escolher o gatilho, adicione um componente de **Advanced Branching** e coloque as ações dentro dele.

Neste exemplo, a ideia é percorrer a lista de watchers de uma issue e enviar um e-mail para cada um.

### Estrutura da regra

```text
Branch: {{issue.watchers}} as watcher
Action: Send email
To: {{watcher.emailAddress}}
```

É um exemplo simples, mas resolve um problema que o branch normal não resolve diretamente.

### Observação importante

No Jira Cloud, o `emailAddress` pode estar oculto dependendo das configurações de privacidade e visibilidade dos usuários.

Ou seja: a lógica da regra continua válida, mas o resultado pode depender de como o seu ambiente trata dados pessoais.

---

## Mais exemplos

O mesmo padrão pode ser usado com outras listas retornadas por smart values.

### Percorrendo fixVersions

```text
Branch: {{issue.fixVersions}} as version
```

Depois, dentro do branch, você pode usar valores como:

```text
{{version.name}}
```

### Percorrendo organizations

```text
Branch: {{issue.organizations}} as org
```

Dentro do branch, você pode usar a variável do item atual nas ações seguintes.

É isso que faz o Advanced Branching ser tão útil: quando você entende o padrão, consegue reaproveitar em vários cenários diferentes.

---

## Boas práticas e armadilhas comuns

Como várias funcionalidades do Jira Automation, o Advanced Branching é simples quando você entende a lógica, mas ainda assim existem alguns pontos que valem atenção.

### 1. Use nomes de variável claros

Evite nomes genéricos demais.

Melhor:

```text
{{issue.watchers}} as watcher
{{issue.fixVersions}} as version
```

Pior:

```text
{{issue.watchers}} as item
```

Um nome claro deixa a regra mais fácil de ler e manter.

### 2. Lembre do escopo da variável

A variável criada no Advanced Branching só existe dentro daquele branch.

Isso significa que você não pode definir `watcher` dentro do branch e esperar usar essa variável depois, fora dele.

### 3. Trate listas vazias

Se a lista estiver vazia, o branch simplesmente não vai percorrer nada.

Na maioria dos casos isso é ok, mas em alguns cenários pode valer a pena checar antes se a lista existe ou se tem itens.

### 4. Cuidado com duplicidade

Dependendo da origem da lista, valores repetidos podem aparecer.

Se isso for relevante para o seu caso, vale pensar em como evitar ações duplicadas.

### 5. Verifique permissões e privacidade

Isso é especialmente importante quando você está lidando com usuários e endereços de e-mail.

O smart value pode existir, mas o dado pode não estar disponível na prática por causa de controles de privacidade.

---

## Por que isso importa

Advanced Branching é uma daquelas funcionalidades do Jira Automation que passa fácil despercebida no começo, mas se torna muito útil quando você entende o que ela resolve.

Ele preenche um espaço entre o branch baseado em issues e as listas retornadas por smart values.

Na prática, isso faz dele uma solução muito útil para cenários em que o dado já existe no Jira, mas não em um formato que o branch normal consiga usar diretamente.

---

## Considerações finais

Se você já trabalha com Jira Automation, vale a pena aprender Advanced Branching.

É simples, prático e resolve um tipo de problema que aparece bastante quando você começa a montar regras mais avançadas.

A ideia principal é esta:

- use branch normal para relacionamentos entre issues
- use Advanced Branching para listas vindas de smart values

Quando essa diferença fica clara, a funcionalidade passa a ser muito mais fácil de aplicar.

#jira #jiraautomation #atlassian #jsm #jiraservicemanagement #automation #smartvalues #advancedbranching

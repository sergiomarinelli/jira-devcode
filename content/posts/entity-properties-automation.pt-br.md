---
title: "Entity Properties no Jira Automation: padrões de idempotência, locks e espera de API"
date: 2026-01-14T00:00:00-03:00
draft: false
translationKey: "entity-properties-automation"
tags: ["jira-automation", "entity-properties", "idempotency", "locks", "rest-api", "patterns"]
---

Triggers manuais e web requests são uma combinação perfeita para duplicar execução.

Alguém clica na regra duas vezes. Dois admins executam “só para garantir”. Ou duas automações correm ao mesmo tempo e, no fim, você fica com comentários duplicados, chamadas repetidas de API ou dados inconsistentes.

Neste artigo, eu mostro padrões mais seguros para produção usando **Entity Properties** como um pequeno armazenamento de estado dentro do Jira Automation.

A ideia é simples: deixar suas regras mais seguras quando elas precisam rodar uma vez só, conversar com APIs externas ou evitar ficar presas em estados ruins.

Os padrões que aparecem aqui são:

- locks por issue
- chaves de idempotência
- orquestração de espera de API
- takeover de lock expirado
- retries seguros com contador de tentativas

---

## O que são Entity Properties

Entity Properties são pequenos blocos de dados estruturados que o Jira consegue armazenar em entidades, como issues.

Na prática, isso funciona muito bem como um armazenamento leve de estado para automações.

Isso é útil quando você precisa guardar coisas como:

- se uma regra já está rodando
- se uma ação externa já aconteceu
- qual `jobId` foi retornado por uma API
- quantas tentativas de retry já aconteceram
- qual foi o último erro

---

## A ideia principal: guardar estado na issue

Muitos problemas de automação acontecem porque a regra não tem memória da execução anterior.

Uma forma prática de reduzir isso é guardar um estado explícito diretamente na issue.

Exemplo de JSON armazenado como property da issue:

```json
{
  "status": "running",
  "runId": "rule-123-exec-456",
  "startedAt": "2026-01-14T03:00:00Z",
  "attempts": 1,
  "lastError": null
}
```

Toda execução deveria conseguir responder algumas perguntas importantes:

- Já tem algo rodando? (lock)
- Essa operação já foi concluída? (idempotência)
- Se falhou, devo tentar de novo? (política de retry)
- Se ainda está marcado como running, esse estado já expirou? (timeout ou takeover)

---

## Padrão 1: anti-duplicação em trigger manual (lock por issue)

**Objetivo:** se a regra for disparada duas vezes, a segunda execução sai imediatamente.

### Chave da property

Use uma chave previsível, por exemplo:

- `automation.lock.manualSync`

### Fluxo da regra

1. **Trigger:** Manual trigger
2. **Condition (Advanced compare):** lock está vazio
3. **Action:** definir a entity property para criar o lock
4. Executar o trabalho real
5. Atualizar a property para marcar a execução como concluída

### Exemplo de condição

- First value: `{{issue.properties.automation.lock.manualSync}}`
- Condition: **is empty**

### Criando o lock

```json
{
  "status": "running",
  "runId": "{{rule.id}}-{{executionId}}",
  "startedAt": "{{now}}"
}
```

### Marcando como concluído

```json
{
  "status": "done",
  "runId": "{{rule.id}}-{{executionId}}",
  "finishedAt": "{{now}}"
}
```

### Por que funciona

A primeira execução grava um estado. Execuções paralelas leem esse estado e param.

---

## Padrão 2: chave de idempotência (não repetir efeito colateral)

Locks evitam concorrência. Idempotência evita repetir a mesma operação depois, mesmo que o lock já não exista mais.

Isso é útil para ações que deveriam acontecer apenas uma vez por issue, como:

- criar algo em um sistema externo
- provisionar uma conta ou recurso
- chamar um endpoint de criação em uma API externa

### Chave da property

- `automation.idempotency.createExternal`

### Fluxo da regra

1. Se `{{issue.properties.automation.idempotency.createExternal}}` **não estiver vazio**, interrompa a regra
2. Caso contrário, execute a operação
3. Grave o estado de conclusão

### Exemplo de estado final

```json
{
  "done": true,
  "doneAt": "{{now}}",
  "by": "{{initiator.accountId}}"
}
```

### Dica útil

Se a API externa também suportar header de idempotência, envie uma chave estável, por exemplo:

- `{{issue.key}}:createExternal`

Assim, mesmo que o Jira tente novamente, o sistema externo ainda consegue bloquear uma criação duplicada.

---

## Padrão 3: esperar API sem duplicar chamadas

Muitas APIs não são realmente síncronas.

Um padrão comum é este:

- `POST /jobs` retorna imediatamente com um `jobId`
- depois você faz polling em `GET /jobs/{jobId}` até o job ficar `DONE`

Uma abordagem mais segura é dividir a lógica em **duas regras**:

- uma regra de início
- uma regra de polling

### Etapa A: regra inicial

- **Trigger:** manual trigger ou evento da issue
- **Guards:** lock + idempotência
- **Action:** enviar um web request para criar o job externo

Depois disso, guarde o `jobId` retornado.

### Chave da property

- `automation.job.externalSync`

### Exemplo de estado salvo

```json
{
  "status": "started",
  "jobId": "{{webResponse.body.jobId}}",
  "startedAt": "{{now}}",
  "attempts": 1
}
```

Você também pode adicionar um comentário ou atualizar um campo, como `Sync status = Running`.

### Etapa B: regra de polling

Possíveis triggers:

- uma regra agendada a cada poucos minutos com filtro JQL
- ou uma regra disparada por atualização de campo, se você quiser mais controle

### Exemplo de JQL

```text
issue.property[automation.job.externalSync].status = "started"
```

Depois disso:

- envie `GET /jobs/{{issue.properties.automation.job.externalSync.jobId}}`
- se a API retornar `DONE`, atualize a issue e marque a property como concluída
- se retornar `FAILED`, grave o erro e decida se vai tentar novamente
- se ainda estiver rodando, não faça nada e deixe a agenda rodar de novo depois

### Marcando como concluído

```json
{
  "status": "done",
  "jobId": "{{issue.properties.automation.job.externalSync.jobId}}",
  "finishedAt": "{{now}}",
  "result": "{{webResponse.body.result}}"
}
```

### Marcando como falha

```json
{
  "status": "failed",
  "jobId": "{{issue.properties.automation.job.externalSync.jobId}}",
  "failedAt": "{{now}}",
  "lastError": "{{webResponse.status}} - {{webResponse.body}}"
}
```

---

## Padrão 4: takeover de lock expirado

Execuções podem falhar, expirar ou parar no meio do caminho.

Quando isso acontece, uma property pode continuar em `running` para sempre e bloquear novas execuções.

Para evitar isso, defina uma política de timeout.

Exemplo:

- se o lock tiver mais de 10 minutos, considere como expirado e permita takeover

### Fluxo prático de decisão

- se o lock estiver vazio, prossiga
- se o lock estiver running, mas expirado, sobrescreva e prossiga
- caso contrário, interrompa a regra

### Dica prática

Comece com um timeout conservador, como 10 a 30 minutos.

---

## Padrão 5: retries seguros com contador de tentativas

Retries podem ser úteis, mas são perigosos quando a regra gera efeito colateral.

Em vez de tentar de novo às cegas, deixe os retries explícitos e baseados em estado.

### Chave da property

- `automation.retry.externalSync`

### Exemplo de estado de retry

```json
{
  "status": "failed",
  "attempts": 2,
  "lastAttemptAt": "{{now}}",
  "lastStatus": 502
}
```

### Exemplo de política de retry

- se attempts >= 3, pare e notifique alguém
- se o último status foi 429, 502, 503 ou 504, tente novamente
- caso contrário, pare e exija revisão manual

---

## Convenção de nomes que escala

Se você pretende usar esse padrão em mais de uma regra, mantenha as chaves previsíveis.

Uma convenção prática é:

- `automation.lock.<ruleKey>`
- `automation.idempotency.<operation>`
- `automation.job.<integrationName>`
- `automation.retry.<operation>`

Isso deixa as properties mais fáceis de entender, localizar e manter.

---

## Checklist rápido antes de colocar em produção

Antes de levar esse tipo de automação para produção, valide o básico:

- A regra sai com segurança se for disparada duas vezes?
- Ela evita repetir operações de criação?
- Falhas ficam registradas em algum lugar útil?
- Existe um timeout para evitar lock preso para sempre?
- Os retries têm um limite claro?

Se você implementar só o **Padrão 1 (lock)** e o **Padrão 2 (idempotência)**, já elimina boa parte do risco de execução duplicada.

---

## Considerações finais

Entity Properties são simples, mas muito poderosas quando você precisa que regras de automação se comportem mais como sistemas reais.

Elas ajudam o Jira Automation a ter memória do que já aconteceu — e isso é exatamente o que falta em muitas regras mais avançadas.

Se suas regras chamam APIs, criam recursos externos ou dependem de jobs longos, esse é um padrão que vale muito a pena aprender.

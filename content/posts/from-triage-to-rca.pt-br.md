+++
date = '2026-04-13T10:14:47-03:00'
draft = false
title = 'Da triagem ao RCA: automatizando a comunicação de incidentes críticos com Jira, Statuspage API, Rovo e Confluence'
description = 'Um fluxo prático para triar incidentes críticos, publicar atualizações públicas sanitizadas e gerar documentação de RCA com Jira, Statuspage API, Rovo e Confluence.'
translationKey = 'from-triage-to-rca'
tags = ['jira', 'jira service management', 'statuspage', 'rovo', 'confluence', 'incident management', 'automation', 'rca']
categories = ['Atlassian', 'Automation', 'Incident Management']
+++

## Introdução

Quando um incidente crítico acontece, resolver o problema técnico é só uma parte do trabalho. O time também precisa se comunicar com clareza, atualizar clientes no momento certo e documentar o que aconteceu depois que o incidente termina.

Em muitos times, essas etapas ainda acontecem de forma desconectada. O incidente é triado no Jira, a mensagem pública é escrita manualmente e o RCA é preparado depois, às vezes quando parte do contexto já se perdeu.

Neste artigo, eu mostro uma forma prática de conectar essas etapas.

O fluxo começa no Jira Service Management, onde o incidente é criado e triado. Se o caso for grave o bastante, o Jira Automation abre um incidente público no Statuspage. O Rovo ajuda a gerar mensagens públicas de forma mais segura, removendo detalhes internos sensíveis antes da publicação. Depois, quando o incidente é resolvido, o mesmo processo envia a mensagem final e prepara a documentação de RCA.

A ideia aqui é simples: reduzir trabalho manual durante incidentes críticos, deixar a comunicação mais consistente e gerar documentação melhor para análises futuras.

---

## O papel de cada ferramenta neste fluxo

Antes de entrar nas automações, vale explicar o papel de cada produto.

### Jira Service Management

O Jira Service Management é o ponto principal onde o incidente é criado, triado e acompanhado. Neste fluxo, ele funciona como a fonte de verdade do ciclo de vida do incidente.

### Statuspage

O Statuspage é a camada de comunicação pública. É onde clientes ou stakeholders acompanham atualizações do incidente, degradações de serviço e mensagens de resolução. Nesta implementação, o Jira Automation atualiza o Statuspage por chamadas de API.

### Rovo

O Rovo é a capacidade de IA da Atlassian. Neste fluxo, ele ajuda em tarefas que normalmente tomariam esforço manual, como classificar o incidente, reescrever notas internas em mensagens públicas seguras e gerar o primeiro rascunho do RCA.

### Confluence

O Confluence é a camada de documentação. Depois que o incidente é resolvido, ele guarda a página final de RCA para que o time revise o que aconteceu, o que foi aprendido e o que precisa melhorar.

---

## Por que esse fluxo importa

A comunicação de incidentes ainda é um ponto fraco em muitos times.

Problemas comuns incluem:

- atualizações públicas demoradas porque o time técnico está focado na investigação
- mensagens externas escritas sem padrão
- vazamento de detalhes internos em atualizações para clientes
- desalinhamento entre a timeline pública e o fluxo interno do Jira
- RCA escrito tarde demais, quando parte do contexto já se perdeu

Ao conectar Jira, Statuspage, Rovo e Confluence em um único fluxo, fica possível:

- reduzir o tempo entre a triagem e a comunicação pública
- padronizar a linguagem usada nas atualizações
- evitar exposição de detalhes internos
- manter a timeline pública alinhada com o ciclo de vida interno da issue
- gerar documentação de RCA no fim do incidente enquanto o contexto ainda está fresco

---

## Visão geral da arquitetura

Esta implementação se apoia em três regras de automação:

1. **Automação 1 — Criação e triagem do incidente**
   - Um novo incidente é criado no Jira.
   - O Rovo classifica o incidente.
   - Se ele for crítico e exigir comunicação externa, a regra abre um incidente público no Statuspage.
   - O ID retornado do incidente no Statuspage é salvo no Jira.

2. **Automação 2 — Atualização do incidente**
   - O incidente no Jira é atualizado durante a investigação.
   - O Rovo gera uma nova mensagem pública sanitizada de acordo com o estágio atual.
   - O Jira Automation atualiza o incidente existente no Statuspage por API.

3. **Automação 3 — Resolução do incidente e publicação do RCA**
   - O incidente no Jira é resolvido.
   - O Jira Automation encerra o incidente no Statuspage com uma mensagem pública final.
   - O Rovo gera o rascunho do RCA.
   - O Jira Automation publica uma nova página no Confluence com o conteúdo do RCA.
   - O link da página é salvo de volta no Jira.

---

## Pré-requisitos

Antes de montar as automações, garanta que os itens abaixo já existam.

### Produtos necessários

- Jira Cloud / Jira Service Management
- Statuspage
- Confluence Cloud
- Rovo / Atlassian AI habilitado no ambiente
- acesso ao Jira Automation com uso de web request habilitado

### Campos sugeridos no Jira

Crie estes campos antes de configurar as regras:

- **Statuspage Incident ID** — texto de linha única
- **Public Communication Required** — checkbox ou lista
- **Incident Severity** — lista
- **Incident Criticality** — lista
- **Sanitized Public Message** — texto longo
- **Last Public Update Stage** — lista
- **Confluence RCA URL** — URL ou texto
- **RCA Draft** — texto longo

### Valores sugeridos para o ciclo público

Você pode modelar os estágios públicos com um campo como **Last Public Update Stage**.

Valores recomendados:

- investigando
- identificado
- contorno_disponivel
- correcao_em_andamento
- implantando_correcao
- monitorando
- resolvido

Isso deixa a automação de atualização mais fácil de manter.

---

# Automação 1 — Criação e triagem do incidente

Esta automação é responsável por:

- detectar novos incidentes
- classificá-los com o Rovo
- decidir se comunicação pública é necessária
- abrir o incidente público no Statuspage
- guardar no Jira o ID retornado pelo Statuspage

---

## Regra 1A — Classificar o incidente na criação

### Escopo da regra

```text
Escopo do projeto: seu projeto Jira Service Management
Tipo de regra: regra de projeto
```

### Gatilho

```text
Gatilho: Item de trabalho criado
```

### Condições

```text
Tipo da issue é igual a Incident
Summary não está vazio
Description não está vazia
```

### Fluxo

1. Disparar a regra quando o incidente for criado.
2. Confirmar que o tipo da issue é Incident.
3. Usar o Rovo para classificar a issue.
4. Salvar o resultado da triagem em uma variável.
5. Atualizar os campos da issue com a classificação.
6. Decidir se o incidente deve seguir para comunicação pública.

### Prompt do Rovo para triagem do incidente

```text
Você está ajudando a triar um incidente recém-criado no Jira Service Management.

Sua tarefa é analisar o resumo e a descrição do incidente e classificá-lo para tratamento interno.

Retorne sua resposta somente em JSON válido.

Regras:
- Não adicione explicações fora do JSON.
- Use apenas valores simples.
- Se a descrição estiver pouco clara, faça a classificação mais razoável com base no que estiver disponível.
- Não invente detalhes técnicos que não estejam presentes.
- "public_communication_required" deve ser true somente se o problema aparentar afetar clientes, usuários externos ou serviços críticos para o negócio.
- "criticality" deve ser "critical" somente quando o incidente sugerir grande impacto de negócio, degradação importante ou indisponibilidade.
- "priority_name" deve corresponder exatamente a um dos nomes de prioridade existentes no Jira. Neste exemplo, use: Low, Medium, High, Highest.
- "severity" deve ser um entre: sev4, sev3, sev2, sev1

Formato JSON:
{
  "priority_name": "",
  "severity": "",
  "criticality": "",
  "public_communication_required": false,
  "reason": ""
}

Entrada:
Issue key: {{issue.key}}
Summary: {{issue.summary}}
Description: {{issue.description}}
Reporter: {{issue.reporter.displayName}}
Service: {{issue.customfield_service}}
```

### Exemplo de saída esperada

```json
{
  "priority_name": "Highest",
  "severity": "sev1",
  "criticality": "critical",
  "public_communication_required": true,
  "reason": "O problema aparenta afetar um serviço voltado ao cliente e sugere uma interrupção importante."
}
```

### Salvar a saída da triagem

```text
Ação: Create variable
Nome da variável: triageResultRaw
Smart value:
{{agentResponse.asString}}
```

> Para fazer parsing estruturado do JSON em ações seguintes, use `{{agentResponse.asObject}}`. A string bruta é útil mais para auditoria ou depuração.

### Comentário interno opcional

```text
Ação: Add internal comment

Resultado da triagem:
{{triageResultRaw}}
```

### Atualizar campos do Jira

Exemplo de mapeamento:

```text
Incident Severity              <- sev1 / sev2 / sev3 / sev4
Priority                       <- Highest / High / Medium / Low
Incident Criticality           <- critical / non-critical
Public Communication Required  <- true / false
```

Exemplo de edição avançada em JSON:

```json
{
  "fields": {
    "priority": {
      "name": "{{agentResponse.asObject.priority_name}}"
    },
    "customfield_incident_severity": "{{agentResponse.asObject.severity}}",
    "customfield_incident_criticality": "{{agentResponse.asObject.criticality}}",
    "customfield_public_communication_required": {{agentResponse.asObject.public_communication_required}}
  }
}
```

### Ponto de decisão

Só continue para comunicação pública quando todas as condições abaixo forem verdadeiras:

```text
Priority é High ou Highest
Incident Criticality é Critical
Public Communication Required é true
```

---

## Regra 1B — Abrir o incidente público no Statuspage

Quando o incidente é classificado como crítico, a próxima parte da regra cria o incidente público.

### Condições

```text
Tipo da issue é igual a Incident
Priority é High ou Highest
Incident Criticality é Critical
Public Communication Required é true
Statuspage Incident ID está vazio
```

### Gerar a primeira mensagem pública com Rovo

```text
Você está ajudando a escrever uma atualização pública no Statuspage para um incidente crítico recém-identificado.

Tarefa:
Reescreva os detalhes internos do incidente em uma mensagem curta, clara e voltada ao cliente.

Regras:
- Não mencione nomes de clientes.
- Não mencione IDs internos de ticket, IDs de conta, URLs internas, hostnames, nomes de pod, nomes de região, stack traces ou endereços IP.
- Não exponha nomes de times internos nem detalhes de escalonamento.
- Não especule sobre a causa raiz.
- Não atribua culpa.
- Use linguagem calma e profissional.
- Mencione apenas o serviço ou a funcionalidade afetada em termos genéricos.
- Diga que o time está investigando.
- Mantenha a saída entre 2 e 4 frases.
- Retorne somente a mensagem final.

Entrada:
Issue key: {{issue.key}}
Summary: {{issue.summary}}
Description: {{issue.description}}
Latest internal comment: {{issue.comments.last.body}}
Severity: {{issue.customfield_incident_severity}}
Service: {{issue.customfield_service}}
```

### Salvar a mensagem pública

```text
Ação: Create variable
Nome da variável: publicMessage
Smart value:
{{agentResponse.asString}}
```

> `{{agentResponse.asString}}` é mais seguro aqui porque a mensagem será enviada como texto simples para uma API externa.

### Campo opcional de rastreabilidade

```text
Ação: Edit issue
Campo: Sanitized Public Message
Valor: {{publicMessage}}
```

### Criar o incidente no Statuspage

```http
POST https://api.statuspage.io/v1/pages/{page_id}/incidents
Authorization: OAuth {STATUSPAGE_API_KEY}
Content-Type: application/json
```

### Corpo da requisição

```json
{
  "incident": {
    "name": "{{issue.key}} - {{issue.summary}}",
    "status": "investigating",
    "impact_override": "critical",
    "deliver_notifications": true,
    "body": "{{publicMessage}}",
    "metadata": {
      "jira_issue_key": "{{issue.key}}"
    }
  }
}
```

### Web request no Jira Automation

```text
Ação: Send web request
Método: POST
URL: https://api.statuspage.io/v1/pages/{page_id}/incidents
Headers:
  Authorization: OAuth {STATUSPAGE_API_KEY}
  Content-Type: application/json
Web request body: custom data
Wait for response: enabled
```

### Salvar o ID retornado do incidente do Statuspage

```text
Ação: Create variable
Nome da variável: statuspageIncidentId
Smart value:
{{webResponse.body.id}}
```

```text
Ação: Edit issue
Campo: Statuspage Incident ID
Valor: {{statuspageIncidentId}}
```

### Comentário interno opcional

```text
Ação: Add internal comment

Incidente no Statuspage criado com sucesso.
Statuspage Incident ID: {{statuspageIncidentId}}
Mensagem pública: {{publicMessage}}
```

---

# Automação 2 — Atualização do incidente e comunicação pública

Esta automação mantém o incidente público alinhado com o ciclo de investigação interna.

Em vez de usar uma única mensagem genérica, a regra usa o estágio atual do incidente para produzir uma atualização pública mais adequada.

---

## Objetivo desta regra

Ela deve rodar sempre que o incidente no Jira mudar de forma relevante, por exemplo:

- mudança do estágio do incidente
- nova atualização interna de investigação
- confirmação de degradação de serviço
- contorno disponível
- correção sendo preparada
- início de implantação
- validação em andamento
- início da fase de monitoramento

---

## Gatilho

```text
Gatilho: Item de trabalho atualizado
```

### Condições recomendadas

```text
Tipo da issue é igual a Incident
Statuspage Incident ID não está vazio
Public Communication Required é true
Incident Criticality é Critical
```

### Condição extra recomendada

Para evitar ruído, rode a regra apenas se um destes pontos mudar:

```text
Campo de estágio do incidente mudou
Campo da última nota interna mudou
Status mudou
Campo de resolução mudou
```

Se o seu processo usa um campo dedicado como **Last Public Update Stage**, melhor ainda.

---

## Estágios públicos sugeridos

Use um campo que represente o estágio da comunicação pública. Exemplos:

```text
investigating
identified
workaround_available
fix_in_progress
deploying_fix
monitoring
```

Isso dá contexto para o Rovo gerar o tipo certo de atualização.

---

## Criar uma variável para o estágio atual

```text
Ação: Create variable
Nome da variável: incidentStage
Smart value:
{{issue.customfield_last_public_update_stage}}
```

---

## Prompt do Rovo para mensagens de atualização

```text
Você está ajudando a escrever uma atualização pública no Statuspage para um incidente crítico em andamento.

Tarefa:
Gere uma atualização curta e voltada ao cliente com base no estágio atual do incidente e nas últimas informações internas.

Regras:
- Não mencione nomes de clientes.
- Não mencione IDs internos de ticket, IDs de conta, hostnames, URLs internas, IPs, stack traces, nomes de banco, nomes de pod, valores secretos ou detalhes internos de implantação.
- Não exponha nomes de times internos nem detalhes de escalonamento.
- Não especule além do que estiver confirmado.
- Não atribua culpa.
- Use linguagem calma e profissional.
- Mantenha a saída entre 2 e 4 frases.
- Retorne somente a mensagem final.
- Adapte o tom e a redação de acordo com o estágio:
  - investigating: diga que o time está investigando
  - identified: diga que a causa foi identificada e que há ações de mitigação em andamento
  - workaround_available: diga que existe um contorno disponível para alguns usuários, quando isso estiver confirmado
  - fix_in_progress: diga que uma correção está sendo preparada ou aplicada
  - deploying_fix: diga que mudanças corretivas estão sendo implantadas com cuidado
  - monitoring: diga que o serviço está se recuperando e sendo monitorado de perto

Entrada:
Issue key: {{issue.key}}
Summary: {{issue.summary}}
Current stage: {{incidentStage}}
Latest internal comment: {{issue.comments.last.body}}
Description: {{issue.description}}
Service: {{issue.customfield_service}}
Severity: {{issue.customfield_incident_severity}}
```

---

## Exemplos de saídas do Rovo por estágio

### Investigando

```text
Estamos investigando um problema que afeta parte da experiência do serviço para alguns usuários. Nosso time está analisando a situação e trabalhando para identificar a causa. Compartilharemos uma nova atualização assim que houver mais informações confirmadas.
```

### Identificado

```text
Já identificamos a causa da atual indisponibilidade e estamos trabalhando nas ações de mitigação. O time está focado em restaurar o comportamento normal do serviço o mais rápido possível. Seguiremos compartilhando atualizações conforme houver progresso confirmado.
```

### Correção em andamento

```text
Estamos aplicando mudanças corretivas relacionadas ao problema atual do serviço. O time está trabalhando com cuidado para restaurar a operação normal e minimizar impactos adicionais. Uma nova atualização será compartilhada assim que a validação avançar.
```

### Implantando correção

```text
Estamos implantando uma mudança corretiva para resolver o problema atual. O time está monitorando a implantação de perto para confirmar a recuperação do serviço. Faremos uma nova atualização quando a validação for concluída.
```

### Monitorando

```text
O desempenho do serviço melhorou e agora estamos monitorando o ambiente de perto. O time está validando a estabilidade para confirmar que o problema foi totalmente mitigado. Continuaremos compartilhando atualizações até que a recuperação esteja totalmente confirmada.
```

---

## Salvar a mensagem de atualização

```text
Ação: Create variable
Nome da variável: publicUpdateMessage
Smart value:
{{agentResponse.asString}}
```

Opcional:

```text
Ação: Edit issue
Campo: Sanitized Public Message
Valor: {{publicUpdateMessage}}
```

---

## Mapear estágios do Jira para status do Statuspage

Você pode mapear o estágio interno para o status do incidente no Statuspage antes de chamar a API.

Mapeamento sugerido:

```text
investigating        -> investigating
identified           -> identified
workaround_available -> identified
fix_in_progress      -> identified
deploying_fix        -> identified
monitoring           -> monitoring
```

Se quiser deixar a lógica mais limpa, guarde o valor mapeado em uma variável.

```text
Ação: Create variable
Nome da variável: statuspageStatus
Smart value:
{{#if(equals(incidentStage,"monitoring"))}}monitoring{{else}}identified{{/}}
```

Se a sua automação não suportar exatamente essa expressão, substitua por branches condicionais.

---

## Atualizar o incidente existente no Statuspage

```http
PATCH https://api.statuspage.io/v1/pages/{page_id}/incidents/{{issue.customfield_statuspage_incident_id}}
Authorization: OAuth {STATUSPAGE_API_KEY}
Content-Type: application/json
```

### Corpo da requisição

```json
{
  "incident": {
    "status": "{{statuspageStatus}}",
    "deliver_notifications": true,
    "body": "{{publicUpdateMessage}}"
  }
}
```

### Web request no Jira Automation

```text
Ação: Send web request
Método: PATCH
URL: https://api.statuspage.io/v1/pages/{page_id}/incidents/{{issue.customfield_statuspage_incident_id}}
Headers:
  Authorization: OAuth {STATUSPAGE_API_KEY}
  Content-Type: application/json
Web request body: custom data
Wait for response: enabled
```

### Comentário interno opcional

```text
Ação: Add internal comment

Incidente do Statuspage atualizado com sucesso.
Estágio: {{incidentStage}}
Atualização pública: {{publicUpdateMessage}}
```

---

## Observações para uso em produção

Para evitar notificações públicas excessivas, vale aplicar um ou mais controles abaixo:

- notificar apenas quando o estágio público mudar
- suprimir atualizações duplicadas se a mensagem não mudou
- adicionar um intervalo mínimo entre notificações públicas
- guardar em campo o último estágio publicado e comparar antes de enviar

Isso mantém a timeline pública mais limpa e evita excesso de comunicação.

---

# Automação 3 — Resolução do incidente e publicação do RCA

Esta automação fecha o ciclo da comunicação pública e publica a documentação final do incidente.

Ela é responsável por:

- gerar a mensagem pública final de resolução
- resolver o incidente no Statuspage
- gerar o rascunho do RCA com Rovo
- salvar o rascunho do RCA no Jira
- publicar uma nova página no Confluence com o conteúdo do RCA
- salvar a URL da página do Confluence de volta no Jira

---

## Gatilho

```text
Gatilho: Item de trabalho transicionado para Resolved
```

Você também pode usar:

```text
Gatilho: Item de trabalho atualizado
Condição: Resolution não está vazia
```

---

## Condições

```text
Tipo da issue é igual a Incident
Statuspage Incident ID não está vazio
Public Communication Required é true
Incident Criticality é Critical
```

---

## Gerar a mensagem final de resolução para o cliente

```text
Você está ajudando a escrever a atualização pública final no Statuspage para um incidente crítico resolvido.

Tarefa:
Transforme as notas internas de resolução em uma mensagem curta, segura e voltada ao cliente.

Regras:
- Não mencione nomes de clientes.
- Não mencione IDs internos de ticket, IDs de conta, IPs, hostnames, URLs internas, nomes de banco, nomes de pod, valores secretos, stack traces ou detalhes internos de implantação.
- Não exponha nomes de times internos nem detalhes de escalonamento.
- Não especule se a causa raiz ainda estiver sob investigação.
- Deixe claro que o problema foi resolvido.
- Descreva rapidamente o impacto ao cliente em termos genéricos.
- Descreva resumidamente a resolução em linguagem segura para o público externo.
- Mencione monitoramento contínuo se fizer sentido.
- Mantenha a saída entre 2 e 4 frases.
- Retorne somente a mensagem final.

Entrada:
Issue key: {{issue.key}}
Summary: {{issue.summary}}
Resolution notes: {{issue.comments.last.body}}
Description: {{issue.description}}
Service: {{issue.customfield_service}}
Severity: {{issue.customfield_incident_severity}}
```

### Salvar a mensagem pública final

```text
Ação: Create variable
Nome da variável: finalPublicMessage
Smart value:
{{agentResponse.asString}}
```

---

## Resolver o incidente no Statuspage

```http
PATCH https://api.statuspage.io/v1/pages/{page_id}/incidents/{{issue.customfield_statuspage_incident_id}}
Authorization: OAuth {STATUSPAGE_API_KEY}
Content-Type: application/json
```

### Corpo da requisição

```json
{
  "incident": {
    "status": "resolved",
    "deliver_notifications": true,
    "body": "{{finalPublicMessage}}"
  }
}
```

### Web request no Jira Automation

```text
Ação: Send web request
Método: PATCH
URL: https://api.statuspage.io/v1/pages/{page_id}/incidents/{{issue.customfield_statuspage_incident_id}}
Headers:
  Authorization: OAuth {STATUSPAGE_API_KEY}
  Content-Type: application/json
Web request body: custom data
Wait for response: enabled
```

### Comentário interno opcional

```text
Ação: Add internal comment

Incidente do Statuspage resolvido com sucesso.
Mensagem pública final: {{finalPublicMessage}}
```

---

## Gerar o rascunho do RCA com Rovo

Depois que o incidente público estiver encerrado, o fluxo pode gerar o rascunho do RCA.

### Prompt do Rovo para geração do RCA

```text
Você está ajudando a gerar o conteúdo final de RCA para um incidente crítico resolvido.

Tarefa:
Crie um rascunho estruturado de RCA em português profissional.

Público:
Times internos de engenharia, suporte, operações e service management.

Formato de saída:
Use exatamente as seções abaixo:

1. Resumo executivo
2. Impacto ao cliente
3. Detecção
4. Linha do tempo
5. Causa raiz
6. Fatores contribuintes
7. Resolução
8. Ações corretivas
9. Ações preventivas
10. Lições aprendidas
11. Itens de acompanhamento

Regras:
- Escreva de forma clara e direta.
- Mantenha um tom técnico e sem busca por culpados.
- Se a causa raiz não estiver totalmente confirmada, escreva: A causa raiz ainda está sob investigação.
- Não inclua nomes de clientes, IDs de conta, credenciais, segredos, URLs internas, IPs ou detalhes sensíveis de infraestrutura.
- Em Ações corretivas, explique o que foi feito para restaurar o serviço.
- Em Ações preventivas, explique o que deve mudar para reduzir a chance de o mesmo incidente acontecer novamente.
- Em Itens de acompanhamento, use bullets e inclua responsável ou próximo passo quando possível.
- Retorne somente o rascunho do RCA.

Entrada:
Issue key: {{issue.key}}
Summary: {{issue.summary}}
Description: {{issue.description}}
Created date: {{issue.created}}
Resolved date: {{issue.resolutiondate}}
All comments: {{issue.comments}}
Resolution notes: {{issue.comments.last.body}}
Severity: {{issue.customfield_incident_severity}}
Service: {{issue.customfield_service}}
```

### Salvar o rascunho do RCA

```text
Ação: Create variable
Nome da variável: rcaDraft
Smart value:
{{agentResponse.asString}}
```

### Salvar o rascunho do RCA no Jira

```text
Ação: Edit issue
Campo: RCA Draft
Valor: {{rcaDraft}}
```

Opcional:

```text
Ação: Add internal comment

Rascunho de RCA gerado com sucesso.

{{rcaDraft}}
```

---

## Publicar a página de RCA no Confluence

Neste ponto, o Jira Automation pode publicar a página de RCA diretamente no Confluence usando a ação **Publish new page in Confluence**.

Essa abordagem é limpa porque a regra cria a página e publica o conteúdo do RCA na mesma etapa. Aqui, o Rovo gera o texto e a automação envia o resultado para o campo **Page content**.

### Ação no Jira Automation

```text
Ação: Publish new page in Confluence
Publish a new page in: seu espaço de RCA no Confluence
Parent: opcional
Enter page title:
RCA - {{issue.key}} - {{issue.summary}}
Page content:
{{rcaDraft}}
```

### Observação importante sobre rich text

Se o editor da automação estiver em modo rich text, envolva o smart value em inline code ou snippet para que o valor seja interpretado corretamente na execução.

Exemplo:

```text
`{{rcaDraft}}`
```

ou

````text
```
{{rcaDraft}}
```
````

### Smart values disponíveis após a publicação da página

```text
{{createdPage.title}}
{{createdPage.url}}
```

### Salvar a URL da página do Confluence de volta no Jira

```text
Ação: Edit issue
Campo: Confluence RCA URL
Valor: {{createdPage.url}}
```

### Comentário interno opcional

```text
Ação: Add internal comment

Página de RCA publicada com sucesso no Confluence.
Título da página: {{createdPage.title}}
URL da página: {{createdPage.url}}
```

---

# Checklist final

Antes de levar essa solução para produção, valide os pontos abaixo:

- [ ] A regra sai com segurança se for disparada duas vezes?
- [ ] O Statuspage Incident ID é criado apenas uma vez?
- [ ] A automação de atualização não publica mensagens duplicadas?
- [ ] Detalhes sensíveis nunca são expostos pelos prompts?
- [ ] Os campos customizados do Jira estão mapeados corretamente?
- [ ] As credenciais da API do Statuspage estão armazenadas com segurança?
- [ ] Os estágios públicos do ciclo de vida estão padronizados?
- [ ] A geração do RCA funciona mesmo quando a causa raiz ainda não está totalmente confirmada?
- [ ] A página do Confluence está sendo publicada no espaço correto?
- [ ] O editor do Confluence está interpretando corretamente o smart value no campo de conteúdo?

---

## Considerações finais

O valor real desse fluxo não está só na automação. O principal ganho está na consistência: a triagem inicia a comunicação, as atualizações públicas acompanham o ciclo real do incidente e a documentação de RCA passa a fazer parte do processo em vez de ser algo deixado para depois.

Ao conectar Jira, Statuspage, Rovo e Confluence, os times conseguem lidar com incidentes críticos de forma mais organizada e deixar uma documentação melhor para o próximo caso parecido.

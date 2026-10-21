# Documentação Técnica: Sistema de Suporte e Automação (Jurema)

## 1. Visão Geral do Funcionamento dos Fluxos

O ecossistema é composto por 6 arquivos principais de workflows do n8n que se comunicam para gerenciar a base de conhecimento, capturar mensagens de chat, processar inteligência artificial (RAG) e automatizar regras de negócio sensíveis (como cancelamentos e retenções).

### Os Arquivos e Seus Papéis:

1. **`Ingestão_Jurema.json`**
    - **Objetivo:** Automatizar o *web scraping* da Central de Ajuda (`help.eemovel.com.br`) ou importar arquivos de texto para indexar e estruturar o conhecimento em um vector store (**Qdrant**), gerando *embeddings* via OpenAI.
2. **`Jurema_incoming_message_v1.json`**
    - **Objetivo:** Atuar como o **Webhook de Entrada** principal. Recebe as mensagens enviadas pelos clientes (via plataforma, WhatsApp ou ambiente de desenvolvimento), valida o canal, valida a existência de interações prévias no banco PostgreSQL e registra as mensagens recebidas (`assistente_conversas_mensagens`).
3. **`Jurema_response_v3 (2).json`**
    - **Objetivo:** O **Agente Principal de Resposta**. Disparado por agendamento (`Schedule Trigger`), ele varre mensagens não lidas, alimenta o modelo de IA (OpenAI GPT-4o com memória de chat no PostgreSQL e ferramenta de RAG via HTTP), processa a intenção do usuário (resolvido, fora do escopo, venda ou cancelamento) e dispara subfluxos.
4. **`Jurema_cancelamento_execution_v1.json`**
    - **Objetivo:** Fluxo especializado em gerenciar o **Churn (Cancelamentos e Retenções)**. Trata vigências de planos (mensal, semestral, anual), valida gestores, interage com o CRM (**Pipedrive**), cria tickets de suporte (**Freshdesk**), envia notificações e atualiza status de assinatura.
5. **`Jurema_marcar_lida.json`**
    - **Objetivo:** Subfluxo utilitário focado em iterar sobre lotes de IDs de mensagens e atualizar o status de leitura (`read = true`) no banco de dados.

## 2. Diagrama de Arquitetura e Integrações

Abaixo está a representação visual do fluxo de dados e comunicação entre os arquivos JSON e os serviços externos:

```
       [ Jurema_incoming_message_v1 ] (Webhook Principal)
       │ (Recebe mensagens do cliente via WPP/Plataforma)
       │ ➔ Grava mensagens no PostgreSQL (assistente_conversas_mensagens)
       ▼
 ┌────────────────────────────────────────────────────────┐
 │            Jurema_response_v3 (2) (Agente AI)          │
 │  (Disparado por Schedule Trigger / Varre não lidas)    │
 └───┬────────────────────────────────┬───────────────────┘
     │                                │
     │ 1. Chama para marcar lidas     │ 2. Se detectar intenção de cancelamento
     ▼                                ▼
┌────────────────────────┐   ┌──────────────────────────────────────────────┐
│  Jurema_marcar_lida    │   │    Jurema_cancelamento_execution_v1          │
│  (Subfluxo utilitário) │   │    (Gerencia Churn, Vigências, Pipedrive     │
│  Atualiza read = true  │   │     e Freshdesk)                             │
└────────────────────────┘   └──────────────────────────────────────────────┘
```

## 3. Principais Queries SQL Utilizadas nas Consultas

O sistema faz uso intensivo de banco de dados relacional PostgreSQL (esquemas `brokers_cs` e `analytics_v2`) para persistência de conversas, histórico de chat e cruzamento de dados de assinaturas.

### A. Consulta de Mensagens Pendentes (`Jurema_response_v3 (2).json`)

Responsável por buscar conversas ativas que possuem mensagens não lidas de clientes (excluindo o ambiente de desenvolvimento):

```sql
SELECT
    ac.id as assistente_conversa_id,
    ac.interaction_id,
    ac.conversation_id,
    ac.user_id,
    ARRAY_AGG(acm.id) AS ids_mensagens,
    STRING_AGG(acm.message, E'\n') AS mensagens
FROM
    brokers_cs.assistente_conversas AS ac
INNER JOIN
    brokers_cs.assistente_conversas_mensagens AS acm
    ON ac.id = acm.assistente_conversa
WHERE
    acm.read = false
    AND ac.closed = false
    AND ac.plataforma != 'dev'
GROUP BY
    ac.id, ac.conversation_id, ac.user_id, ac.interaction_id;
```

### B. Consulta de Dados de Assinatura e Vigência (`Jurema_cancelamento_execution_v1.json`)

Utilizada para recuperar informações financeiras e do plano do usuário a partir da base analítica combinando usuários, inscrições e dados da Iugu:

```sql
SELECT
	pbus.id as id_brokers,
	pbu.email as email,
	ibs.expires_at as final,
	ibs.subscription_id as iugu_id_sub,
	ibs.plan_name as nome_plano,
	case
		when lower(ibs.plan_identifier) like '%anual%' then 'anual'
		when lower(ibs.plan_identifier) like '%semestral%' then 'semestral'
		when lower(ibs.plan_identifier) like '%mensal%' then 'mensal'
		else 'mensal'
	end as vigencia
from
	analytics_v2.platform_brokers_user_subscription pbus
left join
	analytics_v2.platform_brokers_user pbu on pbus.id = pbu.id
left join
	analytics_v2.iugu_brokers_subscriptions ibs on pbu.email = ibs.customer_email
where
	pbus.id = {{ $('gather info').item.json.user_id_brokers }}
	and ibs.active = 't'
```

### C. Consulta de Gestores/Managers (`Jurema_cancelamento_execution_v1.json`)

Busca a hierarquia de contas corporativas ou multi-usuários para identificar se o usuário possui um gestor associado:

```sql
SELECT
  pbu.manager_id
from
	analytics_v2.platform_brokers_user_subscription pbus
left join
	analytics_v2.platform_brokers_user pbu on pbus.id = pbu.id
where
  pbus.id = {{ $('gather info').item.json.user_id_brokers }}
  and pbu.manager_id is not null
limit 1
```

## 4. Ligações e Interações entre os Arquivos (Workflows)

A arquitetura do n8n permite que fluxos principais chamem subfluxos de forma modular utilizando o nó `Execute Workflow`. As principais conexões entre arquivos são:

1. **Entrada (`Jurema_incoming_message_v1.json`):**
    - Funciona como a porta de entrada (Webhook). O usuário envia uma mensagem, o sistema valida o canal (Plataforma, WhatsApp ou Dev), busca sessões anteriores e registra os dados no banco PostgreSQL.
2. **Processamento e IA (`Jurema_response_v3 (2).json`):**
    - É o cérebro do sistema. Roda periodicamente via agendamento, busca mensagens não lidas e aciona o subfluxo **`Jurema_marcar_lida.json`** para limpá-las da fila de pendências. Em seguida, processa a inteligência artificial (GPT-4o + RAG via Qdrant/HTTP).
3. **Execução de Negócio / Churn (`Jurema_cancelamento_execution_v1.json`):**
    - Caso o cliente solicite encerramento ou cancelamento de plano, o agente principal (`Jurema_response_v3`) extrai o motivo e invoca via nó de execução este subfluxo especializado para calcular vigências, criar tickets no **Freshdesk**, registrar atividades no **Pipedrive** e notificar a equipe via **Google Chat**.
4. **Inclusão da Base de Conhecimento (`Ingestão_Jurema.json`):**
    - Executado periodicamente ou sob demanda para garantir que o banco vetorial (**Qdrant**) tenha os dados mais recentes extraídos do helpdesk da eEmovel.
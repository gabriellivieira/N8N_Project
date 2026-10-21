# Documentação Técnica: Fluxos de Automação Financeira e CS

Esta documentação descreve o ecossistema de automações voltado para a gestão de cobrança, inadimplência e notificações operacionais via **n8n**, integrado com bases de dados relacionais (PostgreSQL e Redshift), o CRM (Pipedrive), ferramentas de mensageria (Attemics/WhatsApp) e canais de comunicação (Google Chat e Notion).

## 1. Visão Geral dos Arquivos e Fluxos

### 1. `Automação financeiro - Contagem Cobrança.json`

- **Objetivo:** Realizar a contagem diária das mensagens de cobrança agendadas/enviadas e publicar um resumo consolidado no Google Chat.
- **Funcionamento:**
    1. **Schedule Trigger:** Acionado diariamente às 18:00.
    2. **Postgres:** Executa uma query para contar quantos registros possuem envio programado/executado para o dia corrente.
    3. **HTTP Request:** Formata e envia um card estruturado para o Google Chat com o total contabilizado.

### 2. `Automação Financeiro - Envio cobrança.json`

- **Objetivo:** Executar o disparo automatizado das mensagens de cobrança via WhatsApp (utilizando a API da Attemics) com base na cadência de vencimentos e registrar interações ou erros no Pipedrive e PostgreSQL.
- **Funcionamento:**
    1. **Schedule Trigger:** Executa periodicamente em dias úteis (segunda a sexta, das 07:00 às 18:00, a cada 2 horas).
    2. **Consulta de Registros (`registro`):** Busca na tabela de cadência o próximo registro pendente (`status = false` e `erro_envio = false`) cuja data máxima de envio seja compatível com a data atual.
    3. **Validação de Contato & Canal (Attemics):** Verifica se o contato existe no Attemics, cria o contato caso não exista e inicia/valida o chat.
    4. **Disparo e Atualização:** Envia o texto da cobrança via HTTP Request, cria a atividade correspondente no Pipedrive (separando por pessoa ou organização através de blocos `Switch`) e atualiza o status na tabela do banco de dados.

### 3. `Automação Financeiro - Listener cobrança.json`

- **Objetivo:** Monitorar e escutar as respostas ou interações dos clientes em chats de cobrança abertos, encaminhando casos de suporte, cancelamento ou negociação para a equipe responsável.
- **Funcionamento:**
    1. **Webhook:** Recebe eventos em tempo real de mensagens recebidas via WhatsApp/Attemics.
    2. **Filtros e Normalização (`formata numero`):** Valida se a mensagem não é proveniente de um grupo e se pertence a um cliente cadastrado e com chat aberto na tabela `cobranca_attemics_contact`.
    3. **Processamento de Conversa:** Caso o cliente responda, recupera o histórico da conversa (`get conversa`), formata as mensagens, cria uma atividade de atendimento no Pipedrive contendo o histórico nas notas e encerra o atendimento no sistema.

### 4. `Automação Financeiro - Registros de cobrança.json`

- **Objetivo:** Rotina principal de geração de cadência de cobrança. Varre as faturas em aberto na base de dados analítica (Redshift/Postgres) e popula a tabela de controle de cadência de acordo com a proximidade ou tempo de atraso do vencimento.
- **Funcionamento:**
    1. **Schedule Trigger:** Executa diariamente às 06:00 da manhã.
    2. **Extração de Faturas:** Consulta faturas pendentes na base analítica (`analytics_v2.iugu_brokers_invoices` e assinaturas).
    3. **Fila e Tratamento (`loop`):** Itera sobre cada fatura identificada, valida se já existe registro de cobrança no dia, cruza dados com o Pipedrive para identificar se há negociações ativas no funil e insere os registros mapeados de acordo com a regra de dias (ex: 2 dias antes, no dia, 2 dias depois, 5, 7, 10, 15, 20, 25 e 30 dias de atraso).

### 5. `CS - INADS.json`

- **Objetivo:** Monitoramento e gestão proativa de clientes inadimplentes (INAD) com base no vencimento das assinaturas e faturas.
- **Funcionamento:**
    1. **Trigger Manual / Agendado:** Dispara a consulta consolidada de inadimplência no Redshift.
    2. **Processamento por Lote:** Itera sobre os clientes identificados, busca o negócio correspondente no Pipedrive (`Search a deal`), verifica se está no estágio correto (*Clientes Base*), duplica ou atualiza o negócio e cria uma tarefa de acompanhamento para o responsável.

### 6. `Notificação (#brokers-comercial-financeiro).json`

- **Objetivo:** Central de notificações para solicitações feitas via Notion (Links externos de pagamento e Consultas de CNPJ).
- **Funcionamento:**
    1. **Webhooks:** Recebe notificações do Notion quando novas páginas ou solicitações são criadas/atualizadas.
    2. **Tratamento de Dados (`Code`):** Ajusta fusos horários e formata datas.
    3. **Envio de Alertas:** Dispara cards formatados via Google Chat para os canais internos da equipe comercial e financeira com base no status (*Aprovado*, *Reprovado* ou *Novas Solicitações*).

## 2. Diagramas e Desenhos de Arquitetura

### 2.1. Fluxo de Integração entre Sistemas

```
               [ REDSHIFT (Base Analítica) ]
                            │
                            │ (1. Consulta faturas/assinaturas)
                            ▼
          ┌───────────────────────────────────┐
          │ Automação Financeiro - Registros  │
          │            de cobrança            │
          └─────────────────┬─────────────────┘
                            │
                            │ (2. Popula fila de cobrança)
                            ▼
               [ POSTGRES: brokers_cs ]
               ┌──────────────────────┐
               │  cobranca_cadencia   │◄────────────────┐
               └──────────┬───────────┘                 │
                          │                             │
    ┌─────────────────────┴─────────────────────┐       │
    │ (3. Lê próximo registro)                  │ (7. Atualiza status
    ▼                                           ▼  e manda suporte)
┌───────────────────────────────┐     ┌───────────────────────────────┐
│     Automação Financeiro -    │     │     Automação Financeiro -    │
│         Envio cobrança        │     │       Listener cobrança       │
└───────────────┬───────────────┘     └───────────────▲───────────────┘
                │                                     │
                │ (4. Registra chat ativo)            │ (6. Recebe resposta)
                ▼                                     │
   ┌───────────────────────────┐                      │
   │ POSTGRES: cobranca_       │                      │
   │ attemics_contact          │                      │
   └────────────┬──────────────┘                      │
                │                                     │
                │ (5. Envia WhatsApp)                 │
                ▼                                     │
      [ ATTEMICS / WHATSAPP ] ────────────────────────┘

─────────────────────────────────────────────────────────────────────────
                      FLUXOS AUXILIARES E ALERTAS

  ┌────────────────────────┐      ┌────────────────────────┐
  │ Automação financeiro - │      │        CS - INADS      │
  │    Contagem Cobrança   │      └───────────┬────────────┘
  └─────────────┬──────────┘                  │ (Busca e duplica deals)
                │ (Resumo diário)             ▼
                ├──────────────────────► [ PIPEDRIVE (CRM) ]
                │                             ▲
  ┌─────────────┴──────────┐                  │ (Notificações & Deals)
  │ Notificação (#brokers- │                  │
  │  comercial-financeiro) │──────────────────┘
  └────────────────────────┘
```

## 3. Queries Utilizadas nas Automações

Abaixo estão listadas as principais consultas SQL extraídas dos nós de banco de dados dos fluxos:

### A. Contagem de Cobranças do Dia (`Automação financeiro - Contagem Cobrança`)

```sql
SELECT
	count(id)
FROM brokers_cs.cobranca_cadencia
WHERE status = TRUE AND data_max_envio::date = current_date;
```

### B. Busca de Próximo Registro de Cobrança (`Automação Financeiro - Envio cobrança`)

```sql
select *
FROM brokers_cs.cobranca_cadencia cc
WHERE status = false
  and erro_envio = false
  AND data_max_envio::date >= CURRENT_DATE
limit 1;
```

### C. Verificação de Mensagens com Chat Aberto (`Automação Financeiro - Listener cobrança`)

```sql
SELECT *
FROM brokers_cs.cobranca_cadencia cc
left join brokers_cs.cobranca_attemics_contact cac
  on cc.id = cac.id_contato
WHERE cc.cliente_whatsapp = '+{{ $('formata numero').item.json.number }}'
  AND cc.status = true
  AND cc.mandar_suporte = false
  AND cac.closed = false;
```

### D. Extração de Faturas e Regras de Cadência (`Automação Financeiro - Registros de cobrança`)

```sql
WITH faturas_base AS (
    SELECT
        ibi.customer_name AS nome,
        ibi.customer_id AS id_user,
        pbus.user_id as brokers_id,
        ibi.payer_email AS email,
        concat(ibi.payer_phone_prefix, ibi.payer_phone) as numero,
        (ibi.due_date AT TIME ZONE 'America/Sao_Paulo')::date AS validade,
        ibi.secure_url AS link
    FROM
        analytics_v2.iugu_brokers_invoices ibi
    LEFT JOIN
        analytics_v2.iugu_brokers_subscriptions ibs
        ON ibi.customer_id = ibs.customer_id
    left join
        analytics_v2.platform_brokers_user_subscription pbus
        on ibi.subscription_id = pbus.subscription_integration_id
    WHERE
        ibs.payable_with <> 'credit_card'
        and ibs.plan_identifier <> 'plano_pacotes'
        AND ibi.expired_at IS NULL
        AND ibi.status = 'pending'
)
SELECT
    nome,
    id_user,
    brokers_id,
    email,
    numero,
    link,
    validade,
    CASE (validade - CURRENT_DATE)
        WHEN  2  THEN -2    -- lembrete para 2 dias para vencer
        WHEN -2  THEN 2     -- vencido há 2 dias
        WHEN -5  THEN 5     -- vencido há 5 dias
        WHEN -7  THEN 7     -- vencido há 7 dias
        ELSE NULL           -- fallback
    END AS vencido
FROM faturas_base
WHERE (validade - CURRENT_DATE) IN (2, -2, -5, -7)
group by faturas_base.nome, faturas_base.id_user, faturas_base.brokers_id, faturas_base.email, faturas_base.numero, faturas_base.validade, faturas_base.link;
```

### E. Mapeamento Consolidado de Inadimplência (`CS - INADS`)

```sql
WITH ranked_subscriptions AS (
    SELECT
        pbu.user_id                    AS brokers_user_id,
        ibs.customer_id                AS iugu_customer_id,
        ibs.customer_name              AS iugu_customer_name,
        ibs.customer_email             AS iugu_customer_email,
        ibs.subscription_id            AS iugu_subscription_id,
        ibs.plan_name                  AS plan_name,
        ibs.created_at                 AS criacao_assinatura,
        ibs.expires_at                 AS vencimento_assinatura,

        DATEDIFF(day, GETDATE(), ibs.expires_at) AS INAD_invoice,
        CASE
            WHEN DATEDIFF(day, ibs.expires_at, GETDATE()) < 7 THEN 'INAD 7'
            WHEN DATEDIFF(day, ibs.expires_at, GETDATE()) < 30 THEN 'INAD 30'
            ELSE 'INAD > 30'
        END AS INAD_subscription_status,
        ibi.status                     AS iugu_invoice_status,
        ibi.due_date                   AS vencimento_fatura,
        ibi.booklet_id                 AS id_carne,

        ABS(DATEDIFF(day, ibi.due_date, ibs.expires_at)) AS due_expires_diff,

        ROW_NUMBER() OVER (
            PARTITION BY ibs.customer_id
            ORDER BY ABS(DATEDIFF(day, ibi.due_date, ibs.expires_at)) ASC,
                     ibi.due_date DESC
        ) AS rn

    FROM analytics_v2.iugu_brokers_subscriptions AS ibs
    LEFT JOIN analytics_v2.platform_brokers_user_subscription AS pbu
        ON ibs.subscription_id = pbu.subscription_integration_id
    LEFT JOIN analytics_v2.iugu_brokers_invoices AS ibi
        ON ibs.subscription_id = ibi.subscription_id
    WHERE
        ibs.expires_at BETWEEN
            DATE_TRUNC('month', CURRENT_DATE)::date
            AND (DATE_TRUNC('month', CURRENT_DATE) + INTERVAL '1 month - 1 day')::date
        AND ibi.expired_at IS NULL
        AND ibs.active = 't'
        AND pbu.subscription_integration_id IS NOT NULL
        AND ibi.status <> 'refunded'
		AND ibi.status <> 'partially_refunded'
)
SELECT
    brokers_user_id,
    iugu_customer_id,
    iugu_customer_name,
    iugu_customer_email,
    iugu_subscription_id,
    plan_name,
    criacao_assinatura,
    vencimento_assinatura,
    INAD_invoice,
    INAD_subscription_status,
    iugu_invoice_status,
    vencimento_fatura,
    id_carne
FROM ranked_subscriptions
WHERE rn = 1
ORDER BY vencimento_fatura DESC;
```

## 4. Tabela de Relacionamento entre Tabelas e Sistemas

| Origem / Integração | Destino / Tabela | Chave / Campo de Ligação | Finalidade |
| :--- | :--- | :--- | :--- |
| **Redshift** (`iugu_brokers_invoices`) | **Postgres** (`cobranca_cadencia`) | `email` / `data_vencimento` | Evitar duplicidade de registros de cobrança no mesmo dia. |
| **Postgres** (`cobranca_cadencia`) | **Pipedrive** | `crm_type_id` | Vincular atividades ao perfil correto (Pessoa ou Organização). |
| **Postgres** (`cobranca_cadencia`) | **Postgres** (`cobranca_attemics_contact`) | `id` = `id_contato` | Relacionar o registro da cadência com o chat aberto na Attemics. |
| **Webhook Attemics** | **Postgres** (`cobranca_attemics_contact`) | `cliente_whatsapp` / `chat_id` | Identificar o cliente que respondeu à cobrança. |
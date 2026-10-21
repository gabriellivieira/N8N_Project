# 📚 Documentação Técnica Consolidada: Arquitetura e Automação de Atendimento (CS)

## 📌 Objetivo do Fluxo e Resultados Alcançados

O objetivo principal deste fluxo é **automatizar as tratativas e solicitações que chegam via e-mail por meio de tickets de suporte**.

Como resultado da implementação:

- **75% dos atendimentos** são respondidos e solucionados de forma **totalmente autônoma**.
- Os **25% restantes** demandam apoio ou **validação humana interativa via Google Chat** antes do envio final ao cliente.

---

## 1. Visão Geral da Arquitetura

O sistema é dividido em dois grandes fluxos integrados no n8n:

- **Fluxo A (Criação da Base de Conhecimento / RAG):** Responsável por extrair conteúdos da Central de Ajuda, realizar o parse HTML, fragmentar o texto (*Token Splitter*), gerar os embeddings vetoriais através da OpenAI e persistir os dados no **Qdrant Vector Store** (`brokers_vector_store`)[cite: 1].
- **Fluxo B (Automação de Resposta a Tickets):** Responsável por monitorar tickets abertos no **Freshdesk**, triar e classificar as intenções dos clientes, consultar a base de conhecimento via Agente IA, realizar validação humana (*Human-in-the-Loop*) via **Google Chat** quando necessário, e enviar as respostas formatadas por e-mail[cite: 1, 2].

---

## 2. Diagrama Conceitual do Fluxo Integrado

```
[Fluxo A: RAG]
Central de Ajuda ➔ HTML Parser ➔ Token Splitter ➔ OpenAI Embeddings ➔ Qdrant Vector Store
                                                                             │
[Fluxo B: Atendimento]                                                       │ (Consulta Vetorial)
Schedule ➔ Get Tickets ➔ Filter ➔ Get Ticket ➔ Switch (Classificação)        │
                     │                                                       ▼
                     ├─► Cancelamento/Retenção ──► Formatadores (Transformers) ──► Envio de E-mail
                     ├─► DPO ──────────────────► Update Status (Resolved)       
                     └─► Suporte Geral ────────► AI Agent (GPT-4.1) ──────────┘
                                                       │
                                                       ▼
                                               Google Chat (25% Validação Humana)
                                                       │
                                                       ├─► Aprovado ──► Envio de E-mail / Fechar Ticket
                                                       ├─► Manual ────► Atribuir ao Atendente
                                                       └─► Reprovado ─► AI Revision Agent ──► Correção
```

## 3. Fluxo de Funcionamento Detalhado (`Atendimento ticket.json`)

1. **Gatilho e Filtragem:**
    - **`Schedule Trigger`** executa periodicamente.
    - O **`Filter`** valida se o ticket pertence ao `group_id` `67000574208` e possui status aberto (`2`).
2. **Triagem por Palavras-Chave e Regex (`Switch1`):**
    - **Cancelamento:** Processado pelo `Transformer1`, enviando orientações de recesso/prazos e encerrando o ticket automaticamente.
    - **Retenção (Cobrança, Fatura, Plano):** Processado pelo `Transformer2`, direcionando o cliente para o canal financeiro via WhatsApp e encerrando o ticket.
    - **DPO (Dados/Revogação):** Atualiza diretamente o status para *Resolved*.
    - **Suporte Técnico/Geral:** Encaminha o fluxo para o **`AI Agent`**.
3. **Processamento via IA e RAG:**
    - O **`AI Agent`** (GPT-4.1) utiliza a ferramenta de busca vetorial **`BASE_DE_CONHECIMENTO`** (`brokers_vector_store` no Qdrant) para recuperar os 6 trechos mais relevantes e atualizados da base de dados.
4. **Validação Humana (*Human-in-the-Loop* - 25% dos casos):**
    - O output da IA passa pelo bloco de código `Code` e é enviado ao **Google Chat** via `First validation` com um formulário interativo de opções (*Aprovado*, *Reprovado*, *Assumir Manual*).
    - **Se Aprovado:** O bloco `Transformer` converte o texto em HTML estruturado, envia o e-mail de resposta pelo Freshdesk pra o cliente e marca o ticket como resolvido.
    - **Se Reprovado:** O operador humano insere o feedback no Google Chat (`Secont validation`), acionando o **`AI revision agent`** (GPT-4.1-mini) para reescrever a resposta com base nas correções solicitadas.

## 4. Consultas, Parâmetros e Regras de Negócio

### 4.1. Filtros e Expressões Lógicas

- **Filtro de Tickets:** `group_id == 67000574208` e `status == 2`.
- **Regex de Cancelamento:** `/cancelar|cancelamento/i` aplicado sobre a descrição do ticket.
- **Regex de Retenção/Financeiro:** `/cobrança|fatura|pagamento|plano|estorno|boleto|renovação/i` aplicado sobre o assunto.
- **Regex de DPO:** `/Revogação|Dados/i`.

### 4.2. Configuração da Base Vetorial (Qdrant)

- **Collection:** `brokers_vector_store`
- **Top K:** `6`
- **Instrução da Ferramenta (*Tool Description*):** Busca por similaridade com foco em contornar objeções, priorizando sempre os dados mais recentes com base no metadado `data_envio`.

## 5. Implementação de Código (JavaScript / Node.js)

O fluxo automatizado utiliza blocos de código em JavaScript para manipulação de payloads, tratamento de texto e conversão estruturada para HTML:

### 5.1. Normalização e Correspondência de Tickets (`Code`)

```jsx
const inputItems = $input.all();
const originalTickets = $items("Get_ticket");
const results = [];

for (let i = 0; i < inputItems.length; i++) {
  const currentItem = inputItems[i].json;
  const originalTicketData = originalTickets[i] ? originalTickets[i].json : {};
  const finalResponse = currentItem.output || "Nenhuma resposta válida foi gerada.";

  const finalItem = {
    ...originalTicketData,
    final_response: finalResponse
  };

  results.push({ json: finalItem });
}
return results;
```

### 5.2. Conversão de Texto da IA para HTML no Freshdesk (`Transformer`)

```jsx
try {
  const aiResponse = $('Code').first().json.final_response;
  const ticketData = $('Get_ticket').first().json;

  function convertToHTML(text){
    if (typeof text !== 'string') text = String(text);
    let paragraphs = text.split('\n\n');
    let html = paragraphs.map(p => {
      if (p.trim()) {
        p = p.replace(/(https?:\/\/[^\s<]+)/g, '<a href="$1" target="_blank">$1</a>');
        p = p.replace(/\n/g, '<br>');
        return `<p>${p}</p>`;
      }
      return '';
    }).filter(p => p).join('\n');
    return html;
  }

  const htmlContent = convertToHTML(aiResponse);
  const htmlBody = `<div style="font-family: Arial, sans-serif; line-height: 1.6; color: #333;">
${htmlContent}
<br><br>
<p>Atenciosamente,<br>
<strong>Equipe Brokers</strong><br>
EEmovel Brokers<br>
suporte@eemovel.com.br</p>
</div>`;

  return {
    json: {
      body: htmlBody,
      ticketId: ticketData.id,
      requesterEmail: ticketData.requester ? ticketData.requester.email : ticketData.email
    }
  };
} catch (error) {
  throw new Error(`Erro ao processar resposta: ${error.message}`);
}
```

### 5.3. Geração Dinâmica de Respostas Padrão com Saudação Horária (`Transformer1` / `Transformer2`)

```jsx
try {
  const mensagemFixa = `{{saudacao}}.\n\nOlá! 😊\n\nInformamos que nosso time está em recesso...`;
  const horaAtual = new Date().getHours();
  const saudacao = horaAtual < 12 ? 'Bom dia' : 'Boa tarde';
  const aiResponse = mensagemFixa.replace('{{saudacao}}', saudacao);

  let ticketData;
  try {
    ticketData = $('Get_ticket').first().json;
  } catch (e) {
    ticketData = { id: 'N/A', email: 'cliente@exemplo.com' };
  }

  function convertToHTML(text){
    if (typeof text !== 'string') text = String(text);
    let paragraphs = text.split('\n\n');
    let html = paragraphs.map(p => {
      if (p.trim()) {
        p = p.trim();
        p = p.replace(/(https?:\/\/[^\s<]+)/g, '<a href="$1" target="_blank" style="color: #0066cc;">$1</a>');
        p = p.replace(/([a-zA-Z0-9._-]+@[a-zA-Z0-9._-]+\.[a-zA-Z0-9_-]+)/g, '<a href="mailto:$1" style="color: #0066cc;">$1</a>');
        p = p.replace(/\n/g, '<br>');
        return `<p style="margin-bottom: 1em;">${p}</p>`;
      }
      return '';
    }).filter(p => p).join('');
    return html;
  }

  const htmlContent = convertToHTML(aiResponse);
  const htmlBody = `<div style="font-family: Arial, sans-serif; line-height: 1.6; color: #333;">
${htmlContent}
<p style="margin-top: 2em;">Atenciosamente,<br>
EElisa Assistente Virtual Equipe EEmovel Brokers<br>
Nos chame no WhatsApp: wa.me/5545999787039<br>
<a href="mailto:suporte@eemovel.com.br" style="color: #0066cc;">suporte@eemovel.com.br</a></p>
</div>`;

  return {
    json: {
      body: htmlBody,
      ticketId: ticketData.id,
      requesterEmail: ticketData.requester ? ticketData.requester.email : ticketData.email
    }
  };
} catch (error) {
  return {
      json: {
          error: true,
          message: `Erro ao processar resposta: ${error.message}`,
          stack: error.stack
      }
  };
}
```
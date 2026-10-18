# Assistente de agendamento via WhatsApp (multi-barbearia)

Workflow em **n8n** que atende clientes de barbearias pelo WhatsApp: consulta a agenda, agenda, remarca e cancela horários com um **agente de IA**. Um único workflow atende **várias barbearias**; a configuração de cada uma fica em uma planilha do Google.

## O que o fluxo faz

- **Identifica o cliente pelo telefone** e cadastra automaticamente quem ainda não existe (pede o nome e salva na planilha).
- **Agente de IA com memória por conversa**, que usa o Google Calendar como ferramenta: consultar agenda, verificar horário, criar, buscar, remarcar e cancelar agendamentos.
- **Escolha do barbeiro**: cada barbeiro tem a própria agenda; o agente pergunta com quem o cliente quer atender.
- **Áudio**: o cliente pode mandar áudio, que é baixado e transcrito com Whisper (Groq).
- **Perfil de administrador**: o dono da barbearia consulta a agenda do dia e cadastra promoções pelo WhatsApp.
- **Promoções**: o agente consulta as promoções ativas da barbearia.
- **Lembretes**: rotina que roda a cada hora e envia lembrete dos próximos horários marcados.
- **Proteção contra duplicidade**: mensagens repetidas são descartadas pelo ID da mensagem (evita agendar em dobro).
- **Economia de IA**: saudações, menu, pedido de nome e confirmações usam respostas fixas, sem acionar o modelo.

## Stack

n8n · Evolution API (WhatsApp) · Google Calendar · Google Sheets · Groq (LLM e Whisper) · Docker

## Como funciona (resumo)

```
WhatsApp → Evolution API → Webhook (n8n)
  → remove duplicadas → ignora mensagens próprias e de grupos
  → lê a planilha (barbearia, barbeiros, admins, cliente)
  → monta o contexto → (se for áudio: baixa e transcreve)
  → decide o status: novo | aguardando_nome | ativo | admin
  → resposta fixa OU agente de IA (com ferramentas do Google Calendar)
  → envia a resposta pela Evolution API
```

## Planilha de configuração

O fluxo lê tudo de uma planilha do Google com **5 abas**. A coluna **`Instancia`** é o nome da instância na Evolution API e liga todas as abas: cada barbearia tem uma instância própria. Os nomes das colunas devem ser exatamente estes.

### Aba `Barbearias`

| Coluna | Descrição |
|---|---|
| Instancia | Nome da instância na Evolution API |
| Nome | Nome da barbearia |
| Horarios | Horário de funcionamento, em texto livre |
| Servicos | Serviços, duração e preço, em texto livre |
| EvolutionUrl | URL da sua Evolution API |
| Apikey | Chave da Evolution API desta instância |
| Ativa | `sim` ou `nao` (barbearia desativada não é atendida) |

### Aba `Barbeiros`

| Coluna | Descrição |
|---|---|
| Instancia | Instância da barbearia |
| Nome | Nome do barbeiro |
| CalendarId | ID do Google Calendar do barbeiro |

### Aba `Admins`

| Coluna | Descrição |
|---|---|
| Instancia | Instância da barbearia |
| Telefone | WhatsApp do administrador, com DDI e DDD (somente números) |
| Nome | Nome do administrador |

### Aba `Clientes` (preenchida pelo próprio fluxo)

| Coluna | Descrição |
|---|---|
| Instancia | Instância da barbearia |
| Telefone | WhatsApp do cliente |
| Nome | Nome informado pelo cliente |
| Status | `aguardando_nome` ou `ativo` |
| Cadastrado em | Data e hora do cadastro |

### Aba `Promocoes`

| Coluna | Descrição |
|---|---|
| Instancia | Instância da barbearia |
| Titulo | Título da promoção |
| Desclicao | Descrição da promoção (o nome da coluna está assim no workflow) |
| Validade | Validade, em texto (ex.: `até 31/10`) |
| ativa | `sim` para a promoção aparecer no atendimento |

### Exemplo de dados fictícios

**Barbearias**

| Instancia | Nome | Horarios | Servicos | EvolutionUrl | Apikey | Ativa |
|---|---|---|---|---|---|---|
| barbearia-exemplo | Barbearia Exemplo | seg a sex 09:00-19:00, sáb 09:00-15:00, domingo fechado | Corte - 30 min - R$ 40; Barba - 20 min - R$ 30 | https://sua-evolution.exemplo.com | SUA_APIKEY | sim |

**Barbeiros**

| Instancia | Nome | CalendarId |
|---|---|---|
| barbearia-exemplo | Barbeiro 1 | agenda-exemplo@group.calendar.google.com |

## Como rodar

1. **Importe** o `workflow.json` no n8n (*Workflows → Import from file*).
2. **Credenciais**: nos nodes de Google Sheets, Google Calendar e Groq, selecione as suas credenciais (o arquivo não traz nenhuma).
3. **Planilha**: crie a planilha com as 5 abas acima e, em cada node de Google Sheets, escolha-a no lugar do placeholder `COLE_AQUI_O_ID_DA_SUA_PLANILHA`.
4. **Agendas**: compartilhe cada Google Calendar dos barbeiros com a conta Google usada na credencial e coloque o ID na aba `Barbeiros`.
5. **Evolution API**: crie uma instância por barbearia, conecte o WhatsApp (QR code) e configure o webhook para `https://SEU-N8N/webhook/barbearia`, com o evento `MESSAGES_UPSERT`.
6. **Ative** o workflow e mande uma mensagem para o número conectado.

> Importante: nunca publique a `Apikey`, IDs de agenda ou telefones reais. Em prints e exemplos, use sempre dados fictícios.

## Status

- [x] Cadastro automático de cliente (pedir nome e salvar)
- [x] Agendamento e cancelamento pelo WhatsApp
- [x] Áudio com transcrição
- [x] Proteção contra mensagens duplicadas
- [ ] Remarcação, lembretes, perfil de administrador e promoções: implementados no fluxo, em fase de validação

## Próximos passos

- Cobrança da mensalidade com link de pagamento (Mercado Pago) e renovação automática
- Modo de teste gratuito de 7 dias, com desativação automática ao fim do prazo

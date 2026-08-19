# ADR-005 — Entrega at-least-once com identificação do evento

- **Status:** Aceito
- **Data da decisão:** reunião técnica registrada em `TRANSCRICAO.md`

## Contexto

Após enviar um webhook, o worker pode falhar antes de registrar o sucesso. Assim, não é possível distinguir com certeza entre “cliente não recebeu” e “cliente recebeu, mas a confirmação local falhou”. Exactly-once exigiria coordenação entre OMS e consumidor.

## Decisão

**[ADR-005]** Oferecer entrega **at-least-once**. O `event_id` UUID é criado junto com a outbox, aparece no corpo e no header `X-Event-Id`, e permanece igual em todas as tentativas daquele evento. O consumidor é responsável por deduplicar usando esse identificador.

`X-Timestamp` informa o instante do envio e `X-Webhook-Id` identifica a configuração de destino. Uma resposta HTTP `2xx` confirma entrega; timeout, erro de transporte e resposta não `2xx` entram na política de retry.

## Alternativas consideradas

- **Exactly-once:** descartado pela coordenação e complexidade desproporcionais entre sistemas independentes.
- **At-most-once:** descartado porque uma falha transitória causaria perda definitiva do evento.

## Consequências

### Positivas

- Evita perda por falhas transitórias ou queda entre envio e confirmação.
- Identificador estável permite deduplicação simples no consumidor.

### Negativas e trade-offs

- Duplicatas são esperadas e precisam estar documentadas para clientes.
- A responsabilidade de idempotência fica parcialmente no consumidor.
- `X-Timestamp` habilita, mas não obriga nem define, uma janela anti-replay do cliente.

## Referências

- Transcrição: [09:24]–[09:26], [09:43]–[09:45], [09:48].

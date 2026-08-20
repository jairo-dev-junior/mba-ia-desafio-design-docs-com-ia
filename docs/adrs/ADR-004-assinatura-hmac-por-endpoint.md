# ADR-004 — Assinatura HMAC por endpoint

- **Status:** Aceito
- **Data da decisão:** reunião técnica registrada em `TRANSCRICAO.md`

## Contexto

O OMS enviará dados de pedidos para infraestrutura externa. O consumidor precisa verificar autenticidade e integridade, e o vazamento de uma credencial não pode comprometer todos os clientes.

## Decisão

**[ADR-004]** Assinar os bytes exatos do corpo JSON com HMAC-SHA256 e enviar o hexadecimal no header `X-Signature`. Cada endpoint possui secret única, gerada pelo OMS e devolvida somente na criação ou rotação. A rotação mantém a secret anterior válida por 24 horas. URLs devem usar HTTPS; payload acima de 64 KB falha sem truncamento.

O corpo é serializado uma vez e os mesmos bytes são usados no cálculo e na requisição, evitando divergência por reserialização. Secrets e assinaturas não podem aparecer em logs.

## Alternativas consideradas

- **Secret global da plataforma:** descartada por ampliar o raio de impacto de um vazamento.
- **Sem assinatura, confiando apenas em TLS:** descartada porque TLS protege o transporte, mas não oferece ao receptor uma credencial específica para validar a origem da mensagem.
- **Truncar payload acima do limite:** descartado porque alteraria silenciosamente o contrato assinado.

## Consequências

### Positivas

- Consumidores conseguem verificar origem e integridade com bibliotecas comuns.
- Uma secret comprometida afeta apenas um endpoint.
- Grace period permite rotação sem corte abrupto.

### Negativas e trade-offs

- O cliente deve armazenar a secret com segurança e implementar verificação HMAC.
- Duas secrets precisam ser gerenciadas durante 24 horas.
- O modo de emitir/verificar assinaturas durante a sobreposição não foi fechado; uma `X-Signature` única não representa as duas chaves e o contrato precisa de aprovação de segurança.
- O armazenamento seguro/criptografia em repouso da secret não foi decidido na reunião e permanece questão de segurança antes do deploy.

## Referências

- Transcrição: [09:19]–[09:24], [09:42]–[09:45], [09:48].
- Código: `src/shared/logger/index.ts` (padrão de redaction existente).

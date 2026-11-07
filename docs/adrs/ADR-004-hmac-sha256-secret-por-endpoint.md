# ADR-004 - Autenticação HMAC-SHA256 com secret por endpoint

## Status

Aceito em 2026-11-07.

## Contexto

A feature expõe dados de pedidos para endpoints fora da infraestrutura da empresa. O cliente precisa conseguir validar que a requisição veio realmente da plataforma e que o payload não foi adulterado no caminho. Existia também um incidente concreto no histórico: um cliente vazou secret em log da própria aplicação ([09:22] Diego).

A discussão definiu o algoritmo, o escopo da secret e o ciclo de vida dela, incluindo rotação.

## Decisão

Assinar o corpo do request com HMAC-SHA256 usando uma secret compartilhada entre a plataforma e o cliente, enviada no header `X-Signature`. Cada endpoint de webhook cadastrado tem sua própria secret, gerada pela plataforma no momento da criação do cadastro, e não existe secret global da plataforma.

A secret é rotacionável por endpoint: o cliente solicita uma nova secret pela API e a anterior permanece válida por 24 horas em paralelo, como grace period para migração dos sistemas dele. Depois desse prazo a secret antiga é invalidada.

Requisitos associados: URL de webhook obrigatoriamente `https`, recusada na validação do schema; limite de 64KB de payload, com erro caso ultrapasse.

## Alternativas Consideradas

1. **Secret global única da plataforma.** Descartada por Sofia: o vazamento de uma secret comprometeria todos os clientes de uma vez ([09:21] Sofia). O trade-off de operação mais simples foi recusado em favor de isolamento por endpoint.
2. **Confiar apenas em TLS, sem assinatura.** Plausível em termos de esforço, mas não atende ao requisito de o cliente provar autenticidade e integridade do lado dele, e não protege contra reenvio forjado para o endpoint do cliente. Descartada implicitamente pela decisão de assinar o payload ([09:19] Sofia, [09:20] Sofia).
3. **mTLS ou OAuth entre as partes.** Alternativa plausível tecnicamente, mas exige que cada cliente opere certificado ou fluxo de token, o que contradiz a premissa de integração simples por HTTP que os clientes B2B pediram. Não foi levada à mesa e não foi adotada.
4. **Rotação imediata, sem grace period.** Descartada porque quebraria a integração do cliente no instante da troca; a janela de 24 horas existe para permitir migração sem downtime ([09:21] Sofia).

## Consequências

### Positivas

- Vazamento de uma secret afeta um único endpoint, não a base de clientes.
- O cliente consegue verificar autenticidade e integridade sem depender de canal adicional.
- Rotação com grace period permite trocar credencial sem parada do lado do cliente.
- HMAC-SHA256 é padrão de mercado, com biblioteca disponível em qualquer stack relevante ([09:20] Sofia).

### Negativas

- A responsabilidade de guardar e usar a secret corretamente passa a ser do cliente, incluindo o risco de vazar em log próprio, que já aconteceu ([09:22] Diego).
- Durante 24 horas existem duas secrets válidas por endpoint, o que amplia a janela de exposição da credencial antiga.
- O worker precisa buscar a secret por endpoint a cada envio, e o payload precisa ser assinado em cima do corpo exato enviado, o que exige cuidado com serialização e com qualquer reescrita do body.
- A plataforma passa a armazenar segredo em banco, exigindo cuidado de log e de acesso. O logger atual já redige campos sensíveis por caminho (`src/shared/logger/index.ts`), e o novo campo precisa ser incluído nessa lista.

## Referências

- Transcrição: [09:19] Sofia, [09:20] Sofia, [09:20] Bruno, [09:21] Sofia, [09:21] Bruno, [09:22] Diego, [09:22] Sofia, [09:23] Sofia.
- Código: `src/shared/logger/index.ts` (redação de campos sensíveis em `redactPaths`), schemas Zod do futuro módulo de webhooks (validação de URL `https`).

# API Gateway — PZaaS

> **Documentação da API:** https://apigateway-doc.vercel.app/

Serviço 01 do projeto **PZaaS (Pizza as a Service)** — disciplina de Arquitetura de Serviços em Nuvem.

Ponto único de entrada de uma pizzaria construída como sistema distribuído, onde cada dupla da turma é dona de um serviço. Este Gateway autentica as requisições, exercita a própria resiliência e roteia para o serviço dono de cada dado, devolvendo a resposta real de quem respondeu.

Implementado em [n8n](https://n8n.io) como workflow visual.

---

## O que o Gateway faz

- **Autentica** toda requisição pela chave `x-api-key`, antes de qualquer chamada externa
- **Verifica o próprio estado de saúde** e recusa requisições quando está em manutenção
- **Testa a resiliência** chamando o Chaos Monkey da turma antes do serviço real
- **Roteia** para o serviço dono do dado (Catálogo, Pagamento, Orquestrador)
- **Repassa** o corpo e o código HTTP originais, sem reescrever
- **Degrada com elegância** — responde `503` padronizado em vez de deixar o cliente esperando

## O que o Gateway não faz — e isso é intencional

- Não guarda pedido, cardápio ou status. Nenhum banco de dados de negócio
- Não decide se um pedido é aprovado ou recusado
- Não gera o `pedido_id` — quem gera é o Orquestrador, dono do pedido

A especificação do projeto sugere que o Gateway orquestre o checkout. A escolha aqui foi outra: a documentação do Orquestrador reivindica a posse do `pedido_id` e do ciclo de status. Se o Gateway também orquestrasse, o mesmo pedido teria dois donos e a rastreabilidade fim a fim quebraria. Num Gateway de mercado (Kong, AWS API Gateway, Apigee) a responsabilidade é de transporte; regra de negócio mora na camada de baixo.

---

## Endpoints

Base: `https://pzaas.online/webhook/gateway26`

| Método | Rota | Função | Destino |
|---|---|---|---|
| `GET` | `/v1/health` | Healthcheck público | Redis (interno) |
| `GET` | `/v1/setHealth` | Alterna o estado de saúde | Redis (interno) |
| `GET` | `/v1/catalogo` | Cardápio com disponibilidade | Catálogo `turmaa12` |
| `POST` | `/v1/orquestrador` | Cria um pedido | Orquestrador `turmaa07` |
| `GET` | `/v1/status` | Consulta status de um pedido | Orquestrador `turmaa07` |
| `POST` | `/v1/pagamento` | Processa um pagamento | Pagamento `turmaa21` → `Pagamento8413` |

Todas exigem `x-api-key: turma2026`, exceto `/v1/health`.

---

## Como funciona

O padrão que se repete nas rotas de negócio:

```
webhook → health? → autentica → Chaos Monkey → serviço real → 200
             │          │            │              │
            503        401          503            503
```

1. **Portão de manutenção** — a rota consulta o estado de saúde no Redis. Se estiver `false`, para aqui com `503`
2. **Autenticação** — sem `x-api-key` válida, `401`, sem consumir recurso de ninguém
3. **Chaos Monkey** — chamada ao serviço de caos da turma, tratando a resposta dele como se fosse a própria falha
4. **Serviço real** — a chamada de verdade, com Retry e repasse fiel da resposta

### Resiliência

| Camada | Comportamento |
|---|---|
| **Retry** | 3 tentativas, 1s de espera, disparando em `5xx` e timeout |
| **Fallback A→B** | Na rota de pagamento: se `turmaa21` falhar, tenta `Pagamento8413` |
| **Degradação** | Esgotadas as tentativas, `503` no envelope padronizado |
| **Timeout** | 40s nas rotas que dependem do Orquestrador e do Catálogo |

Erros de negócio (`400`, `401`, `404`, `503`) vindos do destino **não** disparam retry — são respostas válidas e são repassadas na hora.

### Modo manutenção

`GET /v1/setHealth` inverte o estado de saúde. Com o serviço marcado como indisponível, as quatro rotas de negócio recusam com `503`.

`/v1/health` e `/v1/setHealth` ficam de fora do bloqueio: um healthcheck que para de responder perde a função que tem, e se o endpoint de religar fosse bloqueado, o serviço ficaria desligado para sempre.

### Dois formatos de erro

Quando o erro é do Gateway, `erro` é um **objeto**:

```json
{ "erro": { "codigo": 401, "mensagem": "sem x-api-key ou invalida", "servico": "catalogo" } }
```

Quando vem do serviço de destino, o formato dele é preservado, onde `erro` é uma **string**:

```json
{ "erro": "nao_encontrado", "mensagem": "Pedido PZ-... nao encontrado", "servico": "orquestrador-a" }
```

Reescrever tudo para um formato único faria perder informação — como o campo `dependencia` do Orquestrador, que diz exatamente qual peça caiu.

---

## Serviços integrados

| Serviço | Rota consumida | Situação |
|---|---|---|
| Catálogo `turmaa12` | `GET /v1/menu` | Integrado |
| Orquestrador `turmaa07` | `POST /v1/pedidos` | Integrado |
| Orquestrador `turmaa07` | `GET /v1/pedidos/{id}/status` | Integrado |
| Pagamento `turmaa21` | `POST /v1/turmaa21_Pagamento/fluxo` | Integrado (principal) |
| Pagamento `Pagamento8413` | `POST /v1/PagamentoFinal` | Integrado (fallback) |
| Chaos Monkey | `POST /` | Integrado |
| Logger | `POST /v1/logs` | Não implementado |

---

## Como rodar

O workflow roda no n8n compartilhado da turma. Para executar numa instância própria:

1. Importe `src/ApiGateway-240823.json` no n8n (**Workflows → Import from File**)
2. Configure uma credencial Redis e vincule aos nós `Renan`, `Renan2`, `Get Health Atual`, `vivo`, `morto` e aos quatro `Health Check`
3. Inicialize o estado de saúde gravando `"true"` na chave `gateway26/v1/health`
4. Ative o workflow
5. Ajuste os prefixos de webhook se houver conflito com outro workflow na mesma instância

> As URLs dos serviços de destino apontam para os workflows das outras duplas no ambiente da turma. Fora dele, substitua pelos seus próprios destinos ou mocks.

## Testando

```bash
# healthcheck (rota pública)
curl -i https://pzaas.online/webhook/gateway26/v1/health

# cardápio
curl -s https://pzaas.online/webhook/gateway26/v1/catalogo \
  -H "x-api-key: turma2026"

# criar pedido
curl -i -X POST https://pzaas.online/webhook/gateway26/v1/orquestrador \
  -H "Content-Type: application/json" \
  -H "x-api-key: turma2026" \
  -d '{"cliente":{"id":"c-1023","nome":"Renan"},
       "itens":[{"pizza_id":"mussarela","qtd":1}],
       "pagamento":{"metodo":"pix"}}'

# consultar status
curl -i "https://pzaas.online/webhook/gateway26/v1/status?pedidoId=PZ-..." \
  -H "x-api-key: turma2026"

# pagamento
curl -i -X POST https://pzaas.online/webhook/gateway26/v1/pagamento \
  -H "Content-Type: application/json" \
  -H "x-api-key: turma2026" \
  -d '{"pedido_id":"9040","cliente":"Renan","valor":42.90,"saldo_cliente":100.00}'

# sem chave → 401
curl -i https://pzaas.online/webhook/gateway26/v1/catalogo
```

Um `503` contendo *"chaos monkey"* é sorteio do serviço de caos — repita a chamada.

---

## Contrato em aberto

Pontos conhecidos que ainda não foram implementados, listados aqui de propósito:

- **Propagação do `x-pedido-id`** — o Orquestrador recomenda esse header na consulta de status. Hoje o identificador viaja apenas na URL
- **Rate limit** — o Orquestrador delega o limite de requisições ao Gateway; nenhuma rota devolve `429`
- **Envio de logs ao Logger** — a especificação pede que todos os serviços enviem log estruturado. Falta a URL da dupla responsável
- **Endpoint de pizza individual** — o Catálogo expõe `GET /v1/menu/{pizza_id}`, que o Gateway não proxya por decisão de escopo

---

## Estrutura

```
.
├── README.md
└── src/
    └── ApiGateway-240823.json    # workflow do n8n (62 nós)
```

---

**Renan Hideki** · RA 240823
Arquitetura de Serviços em Nuvem · PZaaS

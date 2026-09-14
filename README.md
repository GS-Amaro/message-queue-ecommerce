# message-queue-ecommerce

Trabalho prático da disciplina de Sistemas Distribuídos. O projeto demonstra o uso de mensageria (RabbitMQ) no backend de processamento de pedidos de uma loja virtual.

## Status

O código ainda não foi implementado. O repositório contém apenas a documentação das Etapas 1 e 2 (entregues em 17/09/2026). As Etapas 3 a 5, com o código em Go, ficam para 24/09/2026.

| Etapa | Conteúdo | Status |
|---|---|---|
| 1–2 | Cenário e arquitetura | Concluída |
| 3–5 | Configuração, casos de uso e implementação | Pendente |

## Cenário

O sistema processa pedidos de e-commerce em quatro etapas: registrar o pedido, cobrar o pagamento, reservar o estoque e notificar o cliente. Essas etapas têm latências e dependências externas muito diferentes, o que motiva desacoplá-las com um broker de mensagens em vez de executá-las de forma síncrona.

Detalhes da justificativa em [`docs/etapa1.md`](docs/etapa1.md).

## Arquitetura

Dois processos (`api` e `worker`), um broker RabbitMQ e um banco PostgreSQL. A API publica o evento `pedido.criado` em um exchange do tipo `topic`; os handlers do `worker` (pagamento, estoque, notificação) consomem e publicam os eventos seguintes, encadeando o fluxo sem se conhecerem diretamente.

Inclui estratégias de escalabilidade (competing consumers), confiabilidade (publisher confirms, ack manual, idempotência) e tolerância a falhas (retry com TTL, dead letter queue).

Detalhes completos, diagramas e justificativa de decisões em [`docs/etapa2.md`](docs/etapa2.md).

## Documentação

| Arquivo | Conteúdo |
|---|---|
| [`docs/PRD.md`](docs/PRD.md) | Especificação de build: nomes, casos de uso, regras de implementação, plano de entrega |
| [`docs/etapa1.md`](docs/etapa1.md) | Cenário e justificativa da mensageria |
| [`docs/etapa2.md`](docs/etapa2.md) | Arquitetura, topologia, escalabilidade, confiabilidade, tolerância a falhas |
| [`docs/diagramas/`](docs/diagramas) | Diagramas (`.mmd` e `.png`) |

## Stack planejada

Go 1.23 · `rabbitmq/amqp091-go` · `net/http` · `database/sql` · PostgreSQL 16 · Docker Compose

## Como rodar

Ainda não aplicável — código não implementado. As instruções de execução (`docker compose up`, endpoints, variáveis de ambiente) serão adicionadas na Etapa 5, junto com o código.

## Equipe

Divisão de trabalho e responsabilidades de apresentação em [`docs/PRD.md` §7](docs/PRD.md#7-plano).

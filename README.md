# E-commerce - Projeto 3

Projeto de e-commerce refatorado para usar InversifyJS como container de IoC, com PostgreSQL e Docker Compose, para obtenção de nota na matéria de Arquitetura de Software. Contém dois modos de execução selecionáveis por variável de ambiente (`APP_ENV`), sem mudar regras de negócio do projeto original em https://github.com/LemonGustavo/ecommerce_2/tree/main.

## Estrutura

```
src/
├── container/     # container.ts (único lugar que importa classes concretas e monta o grafo
│                  # de dependências conforme APP_ENV) e types.ts (Symbols das dependências)
├── domain/        # IProduct, PhysicalProduct, DigitalProduct, ProductFactory
├── repositories/  # IOrderRepository, PrismaOrderRepository e InMemoryOrderRepository
├── providers/     # IMailProvider, EtherealMailProvider e ConsoleMailProvider
├── payment/       # IPaymentMethod, CreditCardPayment, PixPayment e PaymentMethodRegistry
├── services/      # OrderService (regra de negócio do pedido) e NotificationService
├── controllers/   # OrderController (só HTTP: valida entrada e traduz erros em status code)
├── lib/           # anota o que acontece no sistema, mostrando no terminal
│                  # e salvando no arquivo app.log pra consultar depois.
└── app.ts         # pede o OrderController ao container e registra as rotas
```

## Pré-requisitos

- Docker e Docker Compose (plugin v2, comando `docker compose`)

Para rodar localmente sem Docker (opcional):

- Node.js
- npm

## Como rodar (Docker — recomendado)

```bash
cp .env.example .env
docker compose up --build
```

A API sobe em `http://localhost:3000`. O banco PostgreSQL roda no serviço `banco` (sem porta publicada) e os dados ficam no volume `pgdata` — `docker compose down` e `up` de novo **não** apagam os pedidos. Para apagar os dados, use `docker compose down -v`.

Na primeira execução, o container da `api`:

1. Gera o Prisma Client (`prisma generate`)
2. Cria as tabelas no PostgreSQL (`prisma db push`)
3. Roda o seed com os produtos (`prisma/seed.ts`)
4. Sobe o servidor (`tsx src/server.ts`)

## Como rodar (local, sem Docker — opcional)

Útil para desenvolvimento rápido com `tsx watch`:

```bash
npm install
cp .env.example .env
# edite o .env e troque o host "banco" por "localhost" na DATABASE_URL
npm run prisma:migrate
npm run prisma:seed
npm run dev
```

O modo de execução é definido pela variável `APP_ENV` no `.env`. O container (`src/container/container.ts`) lê essa variável e escolhe as implementações. Assim, nenhum service ou controller conhece `Prisma`, `Nodemailer` ou implementações concretas.

### Modo `dev` (padrão)

Usa Prisma como repositório e EtherealMailProvider para enviar e-mails de teste.

```env
APP_ENV=dev
```

### Modo `test` (sem banco, sem internet)

Usa **InMemoryOrderRepository** (array em memória) e **ConsoleMailProvider** (imprime o e-mail no console). Não precisa de banco nem de internet.

```env
APP_ENV=test
```

> Ao trocar o `APP_ENV`, reinicie o servidor (ou rode `docker compose up --build` de novo, se estiver no Docker).

## Testando

Com a API rodando, em outro terminal:

**Criar um pedido (POST):**

```bash
curl -X POST http://localhost:3000/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customer": "cliente@ethereal.email",
    "items": [
      { "productId": 1, "quantity": 1 }
    ],
    "paymentMethod": "pix"
  }'
```

Resposta esperada (status 200):

```json
{
  "message": "Pedido processado com sucesso",
  "orderId": 1,
  "total": 110,
  "emailPreview": "https://ethereal.email/message/..."
}
```

- No modo `dev`, `emailPreview` vem com uma URL do Ethereal.
- No modo `test`, `emailPreview` é `false` e o e-mail aparece no console do servidor.

**Buscar um pedido (GET):**

```bash
curl http://localhost:3000/orders/1
```

Retorna o pedido com `id: 1` (status 200).

**Pedido inexistente (GET):**

```bash
curl -i http://localhost:3000/orders/999
```

Retorna status **404** com `{"error":"Pedido 999 não encontrado"}`.

Também é possível testar com cartão de crédito trocando o final do JSON:

```json
"paymentMethod": "credit_card",
"paymentDetails": { "cardNumber": "4111111111111111", "cvv": "123" }
```

## Arquitetura de injeção

- `src/container/types.ts` — possui um `Symbol` para cada dependência.
- `src/container/container.ts` — único arquivo que importa classes concretas e decide o que será injetado conforme `APP_ENV`.
- Todas as formas de pagamento estão no mesmo `TYPES.PaymentMethod`, e são consumidas via `@multiInject` no `PaymentMethodRegistry`.
- `MailProvider` e os repositórios usam `inSingletonScope()`, que define que essas dependências terão uma única instância compartilhada por toda a aplicação.
- A rota `GET /orders/:id` passa pelas três camadas: `OrderController` → `OrderService` → `IOrderRepository`. O controller só trata HTTP (lê o `id`, devolve 200 ou 404), o service tem a regra (pedido inexistente é erro) e o repositório busca o pedido (implementado tanto no Prisma quanto no InMemory).

## Infraestrutura

- `Dockerfile` — build da API (Node 22 Alpine, `npm ci`, `prisma generate`, `npm start`).
- `compose.yaml` — dois serviços: `banco` (PostgreSQL 16, sem porta publicada, com healthcheck) e `api` (build local, exposta na porta 3000, `depends_on` com `condition: service_healthy`).
- `.env.example` — `DATABASE_URL` apontando para o serviço `banco` (host `banco` na rede do Compose).
- `prisma/schema.prisma` — provider `postgresql`, `url = env("DATABASE_URL")`.
- Volume `pgdata` — persiste os dados do PostgreSQL entre `down`/`up`.

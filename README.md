# 🚀 MicroSaaS To-Do & Booking — Arquitetura Distribuída em Microsserviços

Plataforma MicroSaaS para Gerenciamento Kanban e Agendamento de Reuniões utilizando uma arquitetura orientada a eventos (Event-Driven Architecture), CQRS, resiliência avançada (Circuit Breaker & Retry) e observabilidade completa (Prometheus + Grafana).

---

## 📐 Visão Geral da Arquitetura

O ecossistema é dividido em microsserviços independentes para garantir desacoplamento, alta disponibilidade e escalabilidade vertical/horizontal. A comunicação entre os serviços de escrita e leitura ocorre de forma assíncrona com consistência eventual garantida via mensageria.

```mermaid
graph TD
    %% Cliente
    subgraph Cliente ["Cliente (Mobile / Web / Desktop)"]
        FlutterApp[App Flutter]
    end

    %% Write Model (Node.js)
    subgraph WriteModel ["Write Model (Booking & Mutação)"]
        NodeAPI[Booking Service - Node.js / Express]
        PostgreSQL[(PostgreSQL - Porta 5433)]
        Cockatiel[Cockatiel - Circuit Breaker & Retry]
        SocketServer[Socket.io Server]
    end

    %% Read Model (Laravel)
    subgraph ReadModel ["Read Model (Kanban & Leitura)"]
        LaravelApp[To-Do Service - Laravel 12]
        MySQL[(MySQL - Porta 3306/3307)]
        Ganesha[Ackintosh Ganesha - CB]
    end

    %% Infraestrutura & Mensageria
    RabbitMQ[[RabbitMQ - Event Bus]]
    Redis[(Redis - Cache & Socket Adapter)]

    %% Observabilidade
    subgraph Observabilidade ["Observabilidade & Monitoramento"]
        Prometheus[Prometheus]
        Grafana[Grafana Dashboards]
    end

    %% Fluxos de Comunicação
    FlutterApp -->|Comandos de Mutação POST/PUT/DELETE| NodeAPI
    NodeAPI -->|Persistência Transacional| PostgreSQL
    NodeAPI -->|Publica Eventos com Resiliência| Cockatiel
    Cockatiel --> RabbitMQ
    NodeAPI <-->|Atualizações em Tempo Real| SocketServer
    SocketServer <-->|Pub/Sub State| Redis

    RabbitMQ -->|Consumo Assíncrono| LaravelApp
    LaravelApp -->|Gravação no Read Model CQRS| MySQL
    LaravelApp -->|Circuit Breaker & Cache| Redis

    FlutterApp -->|Consultas de Alta Performance GET| LaravelApp

    Prometheus -.->|Scrape /metrics| NodeAPI
    Prometheus -.->|Scrape /api/metrics| LaravelApp
    Grafana -.->|Dashboards| Prometheus
```

---

## 🛠️ Tecnologias e Padrões Aplicados

### 1. CQRS (Command Query Responsibility Segregation)

- **Write Model** (`Booking_Service` - Node.js + TypeScript + PostgreSQL): Responsável pelo processamento de comandos que alteram o estado da aplicação (Criação, Atualização e Agendamento de Reuniões).
- **Read Model** (`To-do_Service` - Laravel 12 + MySQL + Redis): Otimizado estritamente para consultas ultrarrápidas, mantendo tabelas desnormalizadas sincronizadas assincronamente via eventos.

### 2. Comunicação Assíncrona & Mensageria (RabbitMQ)

A criação ou alteração de uma reunião no serviço Node.js dispara um evento de integração no RabbitMQ (`reuniao_criada`, `cards_queue`), que é consumido pelos workers assíncronos do Laravel com garantia de idempotência via verificação de UUID.

### 3. Resiliência e Tolerância a Falhas

- **Circuit Breaker:** Protege a comunicação com a mensageria. Implementado via **Cockatiel** na API de Escrita e via **Ackintosh Ganesha** (Redis) na API de Leitura (interrompe chamadas em caso de taxa de falhas > 50%).
- **Políticas de Retry:** Tentativas automáticas com backoff exponencial na publicação de mensagens.

### 4. Observabilidade & Rastreabilidade Distribuída

- **Rastreabilidade (Correlation ID):** O cabeçalho `X-Correlation-ID` é injetado e repassado em toda a cadeia de chamadas para permitir rastreio de logs ponta-a-ponta entre serviços.
- **Métricas Prometheus & Grafana:** Métricas customizadas de latência, consumo de filas, quantidade de boards/cards ativos e status do sistema.
- **Probes de Saúde:** Endpoints `/health/live` e `/health/ready` compatíveis com orquestração em Kubernetes.

### 5. Interface do Usuário Reativa (Flutter)

- **Optimistic UI:** A interface do aplicativo Flutter é atualizada imediatamente no envio da ação. Caso o backend retorne uma falha, o app realiza um rollback automático.
- **Colaboração em Tempo Real:** WebSockets via Socket.io + Redis Pub/Sub atualizam instantaneamente a tela de outros usuários conectados.

---

## 🗂️ Estrutura de Repositórios

Este repositório atua como o Orquestrador Central da Arquitetura, reunindo os serviços individuais como submódulos Git:

```plaintext
MicroSaas_Architecture/
├── todo-service/          # 📝 [Laravel 12] Serviço de Kanban e Read Model (Forked)
├── booking-service/       # 📅 [Node.js + Flutter] Serviço de Agendamento e Write Model (Forked)
├── docker-compose.yml     # 🐳 Orquestração unificada de toda a infraestrutura local
└── README.md              # 📖 Documentação técnica da arquitetura
```

---

## 🚀 Como Executar o Ecossistema Completo

### Pré-requisitos

- Docker (v24+) e Docker Compose (v2+)
- Git

### Passo 1: Clonar o Repositório com Submódulos

Para clonar este repositório trazendo o código fonte de ambos os microsserviços:

```bash
git clone --recursive https://github.com/Dev-Egito/MicroSaas_Architecture.git
cd MicroSaas_Architecture
```

Caso já tenha clonado sem o `--recursive`, rode:

```bash
git submodule update --init --recursive
```

### Passo 2: Criar a Rede Compartilhada do Docker

```bash
docker network create microsaas-net
```

### Passo 3: Inicializar a Infraestrutura

```bash
docker compose up -d --build
```

---

## 📊 Mapeamento de Portas e Endereços

| Serviço / Dashboard        | Tecnologia         | Porta Host  | URL / Descrição                              |
|-----------------------------|---------------------|-------------|-----------------------------------------------|
| API Kanban (Read Model)     | Laravel 12 / Nginx  | 8081        | http://localhost:8081                          |
| API Booking (Write Model)   | Node.js / Express   | 8080 / 8082 | http://localhost:8080                          |
| Swagger UI (Kanban)         | L5-Swagger          | 8081        | http://localhost:8081/api/documentation        |
| RabbitMQ Management         | RabbitMQ 3          | 15672       | http://localhost:15672 (guest/guest)           |
| Prometheus Metrics          | Prometheus          | 9090        | http://localhost:9090                          |
| Grafana Dashboards          | Grafana             | 3000        | http://localhost:3000 (admin/admin)            |
| Banco MySQL (Read)          | MySQL 8.0           | 3306        | Banco de dados de Leitura                      |
| Banco Postgres (Write)      | PostgreSQL 15       | 5433        | Banco de dados de Escrita                      |
| Cache & Sub/Pub              | Redis 7             | 6379        | Cache global e estado de Circuit Breakers      |

---

## 👥 Autoria e Desenvolvimento

Projeto desenvolvido de forma colaborativa como trabalho prático para a disciplina de **Tópicos Avançados em Computação (TAC)**.

Todos os membros participaram ativamente em todas as etapas do ecossistema, incluindo definição da arquitetura de microsserviços, implementação de serviços (Laravel, Node.js e Flutter), orquestração de infraestrutura (Docker, Kubernetes, mensageria e observabilidade) e documentação.

- Allan Xavier
- Matheus Egito
- Murilo

---

**Licença MIT** — Livre para fins de estudo e portfólio.
# ToggleMaster

ToggleMaster é uma plataforma simples de gerenciamento e avaliação de feature flags construída com uma arquitetura de microsserviços.

O projeto permite criar flags, definir regras de segmentação, avaliar se uma funcionalidade deve ser liberada para determinado usuário e armazenar eventos para análise.

Projeto desenvolvido para o curso da FIAP de DevOps e Arquitetura Cloud, como parte do Tech Challenge - Fase 2.

## Nosso Grupo

- Larissa N.
- Luiz F.
- Nicholas L.
- Thiago S.
- Vinícius C.

## Arquitetura

A aplicação possui cinco microsserviços independentes:

### Auth Service

Serviço desenvolvido em Go responsável pela criação e validação de chaves de API. Os dados de autenticação são armazenados em PostgreSQL.

Principais endpoints:

- `GET /health`
- `GET /validate`
- `POST /admin/keys`

### Flag Service

Serviço desenvolvido em Python com Flask responsável pelo cadastro e gerenciamento das feature flags. Utiliza PostgreSQL e valida as requisições por meio do Auth Service.

Principais endpoints:

- `GET /health`
- `POST /flags`
- `GET /flags`
- `GET /flags/{nome}`
- `PUT /flags/{nome}`
- `DELETE /flags/{nome}`

### Targeting Service

Serviço desenvolvido em Python com Flask responsável pelas regras de segmentação das flags, como liberação percentual para usuários. Utiliza PostgreSQL e autenticação fornecida pelo Auth Service.

Principais endpoints:

- `GET /health`
- `POST /rules`
- `GET /rules/{flag}`
- `PUT /rules/{flag}`
- `DELETE /rules/{flag}`

### Evaluation Service

Serviço desenvolvido em Go responsável por avaliar uma feature flag para determinado usuário. Ele consulta os serviços de flags e regras, utiliza Redis como cache e publica o resultado da avaliação em uma fila SQS.

Principais endpoints:

- `GET /health`
- `GET /evaluate?user_id={usuario}&flag_name={flag}`

### Analytics Service

Worker desenvolvido em Python responsável por consumir os eventos enviados para a fila SQS e armazená-los em uma tabela DynamoDB. Também disponibiliza um endpoint de saúde.

Principal endpoint:

- `GET /health`

## Fluxo da aplicação

1. Uma chave de API é criada no Auth Service.
2. Uma feature flag é cadastrada no Flag Service.
3. Uma regra de segmentação é cadastrada no Targeting Service.
4. O Evaluation Service recebe o usuário e o nome da flag.
5. A definição e a regra são consultadas e armazenadas temporariamente no Redis.
6. O resultado da avaliação é devolvido ao cliente.
7. Um evento é enviado para o SQS.
8. O Analytics Service consome o evento e grava os dados no DynamoDB.

## Tecnologias

- Go
- Python
- Flask
- Gunicorn
- Docker
- Kubernetes
- Amazon EKS
- Amazon ECR
- Amazon RDS for PostgreSQL
- Amazon ElastiCache for Redis
- Amazon SQS
- Amazon DynamoDB
- Nginx Ingress Controller
- Kubernetes Metrics Server

## Kubernetes

Cada microsserviço possui seu próprio namespace:

- `auth`
- `flag`
- `targeting`
- `evaluation`
- `analytics`

Os manifestos incluem Deployments, Services do tipo ClusterIP, Secrets, ConfigMaps, probes de saúde e limites de recursos.

O acesso externo é feito pelo Nginx Ingress nas seguintes rotas:

- `/auth`
- `/flags`
- `/rules`
- `/evaluate`
- `/analytics`

Os serviços de avaliação e analytics utilizam Horizontal Pod Autoscaler baseado no consumo de CPU, com meta de 70%.

## Estrutura

```text
analytics-service/    Consumo de eventos e gravação no DynamoDB
auth-service/         Criação e validação de chaves de API
evaluation-service/   Avaliação das feature flags
flag-service/         Gerenciamento das feature flags
targeting-service/    Gerenciamento das regras de segmentação
yaml/                 Manifestos Kubernetes
```

## Saúde dos serviços

Todos os microsserviços possuem um endpoint `/health`, utilizado pelas probes do Kubernetes para verificar se as aplicações estão disponíveis.

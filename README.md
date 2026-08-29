<div align="center">
  <img src="https://avatars.githubusercontent.com/u/79948663?s=200&v=4" alt="FIAP" width="140">
  <h3>Tech Challenge - Fase 2</h3>
  <p>Pós-Tech DevOps e Arquitetura Cloud</p>
</div>

# ToggleMaster - Microsserviços em Kubernetes na AWS

<p align="center">
  <img src="https://img.shields.io/badge/AWS-FF9900?style=for-the-badge&logo=amazonwebservices&logoColor=white" alt="AWS">
  <img src="https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white" alt="Kubernetes">
  <img src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white" alt="Docker">
  <img src="https://img.shields.io/badge/Go-00ADD8?style=for-the-badge&logo=go&logoColor=white" alt="Go">
  <img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python">
  <img src="https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white" alt="PostgreSQL">
  <img src="https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white" alt="Redis">
  <img src="https://img.shields.io/badge/DynamoDB-4053D6?style=for-the-badge&logo=amazondynamodb&logoColor=white" alt="DynamoDB">
</p>

## 1. Sobre o projeto

O **ToggleMaster** é uma plataforma de gerenciamento e avaliação de **feature flags** desenvolvida como projeto acadêmico da FIAP para o Tech Challenge da Fase 2.

A solução permite cadastrar funcionalidades, definir regras de segmentação, decidir se uma funcionalidade deve ser liberada para determinado usuário e registrar os resultados para análise.

O sistema utiliza uma arquitetura distribuída com cinco microsserviços conteinerizados e implantados em Kubernetes na AWS.

---

## 2. Microsserviços

| Microsserviço | Tecnologia | Responsabilidade | Integração principal |
| :--- | :---: | :--- | :--- |
| **auth-service** | Go 1.21 | Criação e validação de chaves de API | Amazon RDS PostgreSQL |
| **flag-service** | Python 3.11 | Cadastro e manutenção das feature flags | Amazon RDS PostgreSQL |
| **targeting-service** | Python 3.11 | Gerenciamento das regras de segmentação | Amazon RDS PostgreSQL |
| **evaluation-service** | Go 1.21 | Avaliação das flags para cada usuário | ElastiCache Redis e Amazon SQS |
| **analytics-service** | Python 3.11 | Consumo e armazenamento dos eventos de avaliação | Amazon SQS e DynamoDB |

Cada serviço possui uma responsabilidade específica e pode ser implantado e escalado de forma independente.

---

## 3. Fluxo da aplicação

1. Uma chave de API é criada pelo `auth-service`.
2. Uma feature flag é cadastrada no `flag-service`.
3. Uma regra de segmentação é cadastrada no `targeting-service`.
4. O `evaluation-service` recebe o usuário e o nome da flag.
5. As informações são consultadas nos serviços internos e mantidas temporariamente no Redis.
6. O resultado da avaliação é devolvido ao cliente.
7. Um evento é enviado de forma assíncrona para o Amazon SQS.
8. O `analytics-service` consome a mensagem e grava o evento no DynamoDB.

```text
Cliente
   |
   v
Nginx Ingress
   |
   +--> Auth Service ------> RDS PostgreSQL
   +--> Flag Service ------> RDS PostgreSQL
   +--> Targeting Service -> RDS PostgreSQL
   +--> Evaluation Service -> ElastiCache Redis
                  |
                  v
             Amazon SQS
                  |
                  v
          Analytics Service
                  |
                  v
             DynamoDB
```

---

## 4. Infraestrutura AWS

A solução foi preparada para utilizar os seguintes serviços:

- **Amazon EKS:** execução e gerenciamento do cluster Kubernetes.
- **Amazon ECR:** armazenamento das cinco imagens Docker.
- **Amazon RDS:** três bancos PostgreSQL independentes para autenticação, flags e targeting.
- **Amazon ElastiCache:** cache Redis utilizado pelo serviço de avaliação.
- **Amazon SQS:** comunicação assíncrona entre evaluation e analytics.
- **Amazon DynamoDB:** armazenamento dos eventos de avaliação.
- **Elastic Load Balancing:** ponto de entrada externo criado pelo Nginx Ingress Controller.

O acesso de workloads aos serviços AWS utiliza Service Accounts próprias, evitando armazenar credenciais AWS diretamente nas aplicações.

---

## 5. Kubernetes

Cada aplicação está isolada em seu próprio namespace:

| Namespace | Aplicação |
| :--- | :--- |
| `auth` | auth-service |
| `flag` | flag-service |
| `targeting` | targeting-service |
| `evaluation` | evaluation-service |
| `analytics` | analytics-service |

Os manifestos incluem:

- Deployments para gerenciamento dos Pods.
- Services do tipo ClusterIP para comunicação interna.
- ConfigMaps para configurações não sensíveis.
- Secrets com valores codificados em base64.
- Readiness e Liveness Probes.
- Requests e Limits de CPU e memória.
- Jobs para inicialização das tabelas PostgreSQL.
- Ingress para acesso externo.
- Horizontal Pod Autoscaler para escalabilidade automática.

---

## 6. Rotas externas

O Nginx Ingress Controller centraliza o acesso às aplicações.

| Rota | Serviço | Porta |
| :--- | :--- | :---: |
| `/auth` | auth-service | 8001 |
| `/flags` | flag-service | 8002 |
| `/rules` | targeting-service | 8003 |
| `/evaluate` | evaluation-service | 8004 |
| `/analytics` | analytics-service | 8005 |

Todos os microsserviços possuem um endpoint `/health`, utilizado pelo Kubernetes para verificar se a aplicação está disponível.

---

## 7. Escalabilidade

O projeto utiliza o **Horizontal Pod Autoscaler (HPA)** com métricas fornecidas pelo Kubernetes Metrics Server.

| Serviço | Réplicas mínimas | Réplicas máximas | Meta de CPU |
| :--- | :---: | :---: | :---: |
| evaluation-service | 2 | 10 | 70% |
| analytics-service | 1 | 10 | 70% |

O `evaluation-service` aumenta a quantidade de Pods quando recebe muitas avaliações. O `analytics-service` pode escalar quando o processamento das mensagens aumenta seu consumo de CPU.

---

## 8. Tecnologias

- Go
- Python
- Flask
- Gunicorn
- Docker
- Kubernetes
- Nginx Ingress Controller
- Kubernetes Metrics Server
- PostgreSQL
- Redis
- Amazon Web Services

---

## 9. Estrutura do repositório

```text
analytics-service/    Worker de eventos e integração com DynamoDB
auth-service/         Autenticação e gerenciamento de chaves de API
evaluation-service/   Avaliação das feature flags
flag-service/         CRUD das feature flags
targeting-service/    Regras de segmentação
yaml/                 Manifestos Kubernetes
```

---

## 10. Checklist de demonstração

O repositório inclui um checklist com os testes do ambiente, Pods, Ingress, HPA, SQS e DynamoDB:

📄 [Checklist de Demonstração - ToggleMaster](./Checklist%20de%20Demonstra%C3%A7%C3%A3o%20%E2%80%94%20ToggleMaster.pdf)

Os endereços de recursos AWS apresentados no checklist devem ser atualizados sempre que uma nova infraestrutura for criada.

---

## 11. Nosso grupo

- Larissa N.
- Luiz F.
- Nicholas L.
- Thiago S.
- Vinícius C.

<div align="center">
  <strong>FIAP - Pós-Tech DevOps e Arquitetura Cloud - Tech Challenge Fase 2</strong>
</div>

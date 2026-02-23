# Propriedade Rural Inteligente — Documentação Técnica

## Visão Geral

Plataforma de monitoramento agrícola composta por 5 microsserviços que permite ao produtor rural acompanhar, em tempo real, as condições de suas propriedades e talhões através de dados de sensores IoT. O sistema processa dados de umidade do solo, temperatura e precipitação, gerando alertas automáticos quando condições adversas são detectadas (seca, geada, temperaturas extremas, etc.).

**Microsserviços:**

| Serviço | Tecnologia | Responsabilidade |
|---------|-----------|-----------------|
| **Frontend** | Angular 19 + Nginx | SPA com dashboard de monitoramento, gráficos e alertas |
| **IAM** | .NET 8 | Autenticação JWT, gestão de usuários e RBAC |
| **Sensor API** | .NET 8 | Ingestão de dados IoT e publicação no Kafka (Producer) |
| **Sensor Consumer** | .NET 8 | Processamento de dados IoT, persistência e motor de alertas (Consumer) |
| **Propriedade Rural** | .NET 8 | CRUD de propriedades rurais e talhões |

**Infraestrutura:**

| Componente | Tecnologia |
|-----------|-----------|
| Orquestração | Kubernetes (AKS) |
| API Gateway | Kong (DB-less) |
| Mensageria | Apache Kafka + Zookeeper |
| Banco de Dados | Azure SQL Server (Serverless) |
| Container Registry | Azure Container Registry (ACR) |
| CI/CD | Azure DevOps Pipelines |

## Containerização com Docker

### Dockerfile dos Microsserviços Backend (.NET 8)

Todos os microsserviços backend utilizam Dockerfiles com estratégia multi-stage build:

- **Stage 1 (build):** `mcr.microsoft.com/dotnet/sdk:8.0` para compilação em modo Release
- **Stage 2 (final):** `mcr.microsoft.com/dotnet/aspnet:8.0` como imagem base de runtime

Otimizações: restore de pacotes separado do build para cache de camadas Docker, publicação em Release, imagem final sem SDK.

### Dockerfile do Frontend (Angular 19)

Multi-stage build com 3 estágios:

- **Stage 1 (build):** `node:20-alpine` para `npm install` e `ng build --configuration production`
- **Stage 2 (final):** `nginx:alpine` para servir os arquivos estáticos
- **Runtime env vars:** Script `docker-entrypoint.sh` usa `sed` para substituir placeholders nos arquivos JS compilados por variáveis de ambiente do Kubernetes, permitindo que a mesma imagem funcione em diferentes ambientes

## Orquestração com Kubernetes

### Helm Charts

Cada microsserviço possui seu próprio Helm Chart para gerenciar deployments, serviços e configurações de forma declarativa.

**Estrutura do Chart:**

```
helm/
└── {servico}/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        ├── ingress.yaml
        ├── hpa.yaml
        ├── configmap.yaml
        ├── secret.yaml
        └── serviceaccount.yaml
```

**Componentes Kubernetes:**

**1. Deployment**

- Réplicas configuráveis via `values.yaml`
- Recursos (CPU e memória) com limites e requests
- Probes de liveness e readiness
- Injeção de variáveis de ambiente via Secrets e ConfigMaps

**2. Horizontal Pod Autoscaler (HPA)**

- Métricas baseadas em CPU (70%) e memória (80%)
- Frontend configurado com 2-10 réplicas
- Backend com 1-3 réplicas (desabilitado por padrão no MVP)

**3. Service**

- Backend: ClusterIP (acessível apenas internamente via Kong)
- Frontend: LoadBalancer com IP público
- Kong Gateway: LoadBalancer com IP público estático

**4. Ingress**

- Classe: Kong (API Gateway)
- Paths por serviço: `/iam`, `/sensor-consumer`, `/propriedade-rural`, `/sensor`
- Strip path habilitado para roteamento correto

**5. Secret**

- Connection strings do Azure SQL
- JWT Key (HS256)
- Kafka Bootstrap Servers
- ACR pull secret (`acr-secret`)

**Namespaces:**

| Namespace | Componentes |
|-----------|------------|
| `kong` | Kong Gateway (Proxy) |
| `kafka` | Kafka Broker + Zookeeper |
| `propriedade-rural` | IAM Service |
| `sensor` | Sensor API (Producer) |
| `sensor-consumer` | Sensor Consumer Service |
| `propriedade-rural-frontend` | Frontend Angular (Nginx) |
| `propriedade-rural-microservice` | Propriedade Rural Service |

**Recursos por Microsserviço:**

| Serviço | CPU Request | CPU Limit | Mem Request | Mem Limit |
|---------|-----------|---------|------------|----------|
| Frontend | 200m | 500m | 256Mi | 512Mi |
| Backend (.NET) | 200m | 500m | 256Mi | 512Mi |
| Kafka | 100m | 200m | 256Mi | 512Mi |
| Zookeeper | 50m | 100m | 128Mi | 256Mi |

**Health Checks:**

- **Backend (.NET):** Liveness e readiness via `GET /swagger` — initial delay 30s, period 10s
- **Frontend (Nginx):** Liveness e readiness via `GET /health` — initial delay 10s, period 5s

## CI/CD Pipeline

### Azure DevOps

Pipelines separados de CI e CD por microsserviço, com pipeline de infraestrutura adicional para Kafka e Kong.

**Estrutura:**

```
.azure-pipelines/
├── ci.yml                    # Build + Test + Docker push
├── cd.yml                    # Helm upgrade → AKS
└── infrastructure.yml        # Deploy Kafka + Kong (opcional)
```

**Pipeline de CI:**

1. Trigger em push para `develop` e `master`
2. Instalação do .NET SDK 8.0 (ou Node.js 20 para o frontend)
3. Restore, build e testes unitários
4. Docker build multi-stage
5. Push para ACR com tag `v$(Build.BuildId)` (develop) ou `latest` + `v$(Build.BuildId)` (master)

**Pipeline de CD:**

1. Helm upgrade --install no AKS
2. Rolling update com `maxSurge` e `maxUnavailable`
3. Verificação de health checks pós-deploy

**Pipeline de Infraestrutura:**

1. Deploy do Kafka (Zookeeper + Broker) via manifests
2. Deploy do Kong Gateway via Helm chart oficial
3. Configuração de rotas e plugins do Kong

## Microsserviços — Detalhamento

### Frontend (Angular 19)

- **Framework:** Angular 19 com Standalone Components
- **UI:** Angular Material
- **Gráficos:** Chart.js via ng2-charts
- **Servidor:** Nginx Alpine

**Páginas:** Login, Registro, Dashboard (KPIs + gráficos + status dos talhões), Alertas (listagem com filtros e paginação).

**Segurança:** `AuthGuard` protege rotas autenticadas. `AuthInterceptor` injeta JWT em todas as requisições. Token armazenado em `sessionStorage`.

### IAM Service

- **Runtime:** .NET 8 (ASP.NET Core)
- **ORM:** Entity Framework Core (SQL Server)
- **Database:** IAM DB (`Usuario`, `UsuarioPropriedade`)

**Responsabilidades:** Autenticação JWT (HS256, 30min de expiração), CRUD de usuários, controle de perfis (Administrador e UsuarioPadrao), vinculação usuário-propriedade via Kafka Consumer no tópico `iam-adicionar-propriedade`.

**Endpoints públicos:** `POST /api/Auth/login` e `POST /UsuarioPadrao`.

### Sensor API (Producer)

- **Runtime:** .NET 8 (ASP.NET Core)
- **Messaging:** Confluent.Kafka (Producer)

**Responsabilidades:** Receber dados IoT via `POST /Sensor`, validar (TalhaoId, UmidadeSolo 0-100%, Temperatura -50 a 60°C, Precipitação ≥ 0), publicar `SensorDataEvent` no tópico Kafka `sensor-topics` com `Acks.All`, idempotência e 3 retentativas.

### Sensor Consumer (Consumer + Motor de Alertas)

- **Runtime:** .NET 8 (ASP.NET Core)
- **ORM:** Entity Framework Core (SQL Server)
- **Messaging:** Confluent.Kafka (Consumer)
- **Background Service:** `KafkaConsumerService` (IHostedService)
- **Database:** Sensor Consumer DB (`SensorData`, `Alerta`)

**Responsabilidades:** Consumir eventos do Kafka (`sensor-consumer-group`), persistir dados de sensores, executar motor de alertas e expor APIs REST para leitura.

**Regras do Motor de Alertas:**

| Tipo | Condição |
|------|---------|
| UmidadeBaixa | Umidade < 20% |
| UmidadeAlta | Umidade > 80% |
| TemperaturaBaixa | Temperatura < 5°C |
| TemperaturaAlta | Temperatura > 40°C |
| PrecipitacaoExcessiva | Precipitação > 100mm |
| RiscoGeada | Temperatura ≤ 2°C |
| SoloSeco | Média últimas 5 leituras + atual < 20% |
| CondicoesAdversas | Temperatura extrema + Umidade extrema simultâneas |

### Propriedade Rural Service

- **Runtime:** .NET 8 (ASP.NET Core)
- **ORM:** Entity Framework Core (SQL Server)
- **Database:** Propriedade Rural DB (`PropriedadeRural`, `Talhao`)

**Responsabilidades:** CRUD completo de propriedades rurais e talhões, vinculação de talhões a propriedades, vinculação de propriedades a usuários.

### Kong API Gateway

- **Modo:** DB-less (sem PostgreSQL)
- **Tipo:** LoadBalancer com IP público estático

**Rotas:** `/iam/*` → IAM, `/sensor-consumer/*` → Sensor Consumer, `/propriedade-rural/*` → Propriedade Rural, `/sensor/*` → Sensor API. Strip path habilitado em todas as rotas.

**Plugins:** CORS (`cors-all`) — permite chamadas cross-origin do frontend.

### Apache Kafka

- **Versão:** Confluent CP Kafka 7.4.0 + Zookeeper
- **Namespace:** `kafka`
- **Partições:** 3 por tópico

**Tópicos:**

| Tópico | Producer | Consumer |
|--------|----------|----------|
| `sensor-topics` | Sensor API | Sensor Consumer |
| `iam-adicionar-propriedade` | Propriedade Rural | IAM |

## Justificativas Arquiteturais

### Arquitetura de Microsserviços

**Decisão:** Decomposição em 5 serviços independentes (Frontend, IAM, Sensor API, Sensor Consumer, Propriedade Rural).

**Justificativa:**
- **Separação de Responsabilidades (SRP):** Cada serviço encapsula um domínio específico do negócio. O IAM trata identidade, o Sensor API recebe dados IoT, o Sensor Consumer processa e analisa, e o Propriedade Rural gerencia o cadastro.
- **Deploy Independente:** Permite atualizar o Motor de Alertas sem afetar o cadastro de propriedades, ou escalar o Sensor Consumer independentemente quando o volume de dados IoT aumentar.
- **Resiliência:** A falha de um serviço (ex: Propriedade Rural) não impede a ingestão de dados IoT (Sensor API + Consumer), garantindo que dados de sensores nunca sejam perdidos.
- **Escalabilidade Seletiva:** O Sensor Consumer pode escalar horizontalmente conforme o volume de dados IoT cresce, enquanto o IAM pode manter uma única réplica.

### Comunicação Assíncrona via Apache Kafka

**Decisão:** Uso de Kafka como barramento de mensagens entre Sensor API (producer) e Sensor Consumer (consumer).

**Justificativa:**
- **Desacoplamento Temporal:** O Sensor API não precisa esperar o processamento completo (persistência + alertas) para responder ao dispositivo IoT. Isso reduz a latência de resposta.
- **Resiliência a Picos:** Kafka atua como buffer. Se o volume de dados IoT aumentar repentinamente, as mensagens ficam enfileiradas sem sobrecarga no consumer.
- **Garantia de Entrega:** Idempotência garante que nenhum dado de sensor seja perdido, mesmo em caso de falhas parciais.
- **Replay de Eventos:** Se o consumer falhar, pode reprocessar mensagens desde o último offset commitado.

### Kong API Gateway

**Decisão:** Kong Gateway em modo DB-less como ponto único de entrada para os microsserviços backend.

**Justificativa:**
- **Ponto Único de Entrada:** O frontend faz todas as chamadas para um único IP/domínio (Kong), que roteia para o serviço correto baseado no path (`/iam`, `/sensor-consumer`, `/propriedade-rural`).
- **Strip Path:** Kong remove o prefixo de rota antes de encaminhar para o backend, permitindo que cada serviço não precise conhecer seu path no gateway.
- **CORS Centralizado:** Plugin `cors-all` aplicado uma vez no gateway, evitando configuração redundante em cada microsserviço.
- **DB-less:** Modo sem banco de dados reduz a complexidade operacional, configuração feita via Kubernetes Ingress resources.
- **Extensibilidade:** Plugins de rate limiting, autenticação, logging podem ser adicionados sem modificar código dos microsserviços.

### Banco de Dados Compartilhado (Instância Única, Databases Separados)

**Decisão:** Todos os microsserviços usam o mesmo servidor Azure SQL (`propriedade-rural-core`), mas com databases lógicos separados.

**Justificativa:**
- **Custo:** O Azure SQL com tier Serverless permite pagar por uso. Uma única instância com múltiplos databases é significativamente mais barato que múltiplas instâncias separadas.
- **Simplicidade Operacional:** Um único servidor para monitorar, fazer backup e gerenciar.
- **Isolamento Lógico:** Cada microsserviço tem sua connection string e acessa apenas seu database. Não há tabelas compartilhadas.
- **Auto-pause:** O tier Serverless pausa o banco após inatividade, reduzindo custos em ambientes de desenvolvimento/staging.

### JWT para Autenticação

**Decisão:** Tokens JWT (HS256) com 30 minutos de expiração, armazenados no `sessionStorage` do navegador.

**Justificativa:**
- **Stateless:** O backend não precisa manter sessões. Cada requisição carrega toda a informação necessária (claims: sub, role, email) no token.
- **RBAC:** Claims de role (`Administrador`, `UsuarioPadrao`) permitem controle de acesso baseado em perfis sem queries adicionais ao banco.

### Containerização com Docker + Kubernetes (AKS)

**Decisão:** Todos os componentes são containerizados com Docker e orquestrados via Azure Kubernetes Service (AKS).

**Justificativa:**
- **Portabilidade:** Containers Docker garantem que o ambiente de execução é idêntico em desenvolvimento, staging e produção.
- **Orquestração:** Kubernetes gerencia automaticamente health checks (liveness/readiness probes), rolling updates, restart de containers com falha e auto-scaling (HPA).
- **Horizontal Pod Autoscaler (HPA):** Configurado para o frontend e backend, permitindo escalar automaticamente conforme a carga.
- **Service Discovery:** Serviços se comunicam internamente via DNS do Kubernetes (ex: `kafka.kafka.svc.cluster.local:9092`), sem necessidade de configuração manual de IPs.
- **Azure Integration:** AKS integra nativamente com ACR (pull de imagens), Azure Monitor (logs), e Azure Load Balancer (IPs públicos).

### CI/CD com Azure DevOps

**Decisão:** Pipelines separados de CI e CD por microsserviço, com pipeline de infraestrutura adicional.

**Justificativa:**
- **CI Independente por Serviço:** Alterações no IAM não disparam build do Sensor Consumer, reduzindo tempo e custo de pipeline.
- **CD com Helm:** Helm charts parametrizados permitem deploy consistente e reproduzível, com rollback facilitado.
- **Pipeline de Infraestrutura:** Deploy de Kafka e Kong separado do deploy de aplicação, pois a infraestrutura raramente muda.

## Segurança

### Segurança de Rede

- Microsserviços backend expostos apenas via **ClusterIP** (inacessíveis externamente)
- Tráfego externo passa obrigatoriamente pelo **Kong Gateway** ou pelo **LoadBalancer do Frontend**
- CORS configurado em todos os microsserviços (`AllowAnyOrigin`, `AllowAnyMethod`, `AllowAnyHeader`)
- Nginx adiciona headers de segurança: `X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`

### Segredos

- Connection strings, JWT Keys e configurações sensíveis armazenados em **Kubernetes Secrets**
- Secrets injetados como variáveis de ambiente nos pods via Helm templates
- Secrets do ACR (`acr-secret`) para pull de imagens privadas

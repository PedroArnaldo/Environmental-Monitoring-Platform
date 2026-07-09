# Plataforma de Monitoramento Ambiental

Sistema distribuído baseado em **microsserviços** para coleta de leituras de sensores ambientais e geração automática de alertas quando índices críticos são atingidos. Desenvolvido como Projeto de Extensão do curso de Engenharia de Computação da Universidade Veiga de Almeida (UVA).

---

## Sumário

- [Objetivo](#objetivo)
- [ODS relacionada](#ods-relacionada)
- [Arquitetura](#arquitetura)
- [Microsserviços](#microsserviços)
- [Documentação das APIs](#documentação-das-apis)
- [Como rodar o projeto localmente](#como-rodar-o-projeto-localmente)
- [Containerização com Docker](#containerização-com-docker)
- [Pipeline CI/CD](#pipeline-cicd)
- [Segurança e conformidade com a LGPD](#segurança-e-conformidade-com-a-lgpd)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Equipe](#equipe)

---

## Objetivo

Coletar, armazenar e disponibilizar dados de sensores ambientais (qualidade do ar, temperatura, etc.) e emitir alertas automáticos quando os valores ultrapassam limites seguros, ampliando o acesso da comunidade a informações ambientais confiáveis para apoiar a tomada de decisão.

## ODS relacionada

Referência: <https://brasil.un.org/pt-br/sdgs>

- **ODS 13 – Ação Contra a Mudança Global do Clima** (principal)
- **ODS 11 – Cidades e Comunidades Sustentáveis** (complementar)
- **ODS 3 – Saúde e Bem-Estar** (complementar)

**Justificativa:** ao registrar e disponibilizar dados ambientais de forma contínua, o sistema fortalece a capacidade de monitoramento e resposta a eventos climáticos, apoia políticas urbanas de redução de impactos ambientais e contribui para a prevenção de doenças respiratórias associadas à qualidade do ar.

---

## Arquitetura

O sistema é composto por dois microsserviços independentes, cada um com seu próprio banco de dados. A comunicação entre eles é feita por **HTTP/REST**. Todos os componentes sobem juntos via **Docker Compose**.

```
                 ┌─────────────────────────────────────────────────────┐
   Cliente       │                 Docker Compose (rede interna)        │
 (curl/Postman)  │                                                      │
      │          │   ┌──────────────────┐        ┌──────────────────┐   │
      ├─────────►│   │  SensorService   │◄───HTTP─┤   AlertService   │   │
      │          │   │  ASP.NET Core 8  │         │  ASP.NET Core 8  │   │
      └─────────►│   │  GET/POST        │         │  GET/POST        │   │
                 │   │  /readings       │         │  /alerts         │   │
                 │   └────────┬─────────┘         └────────┬─────────┘   │
                 │            │                            │             │
                 │      ┌─────▼──────┐              ┌──────▼─────┐       │
                 │      │ PostgreSQL │              │ PostgreSQL │       │
                 │      │ sensores_db│              │ alertas_db │       │
                 │      └────────────┘              └────────────┘       │
                 └─────────────────────────────────────────────────────┘
```

**Decisões de projeto:**

- **Separação de responsabilidades:** o `SensorService` só cuida de dados brutos; o `AlertService` concentra a regra de negócio (avaliação de limites).
- **Banco por serviço:** cada serviço acessa apenas o próprio banco, garantindo desacoplamento. A troca de informações acontece somente pelo contrato REST.
- **Comunicação síncrona via HTTP:** o `AlertService` consulta o `SensorService` quando precisa dos dados de uma leitura.

### Stack utilizada

| Camada                     | Tecnologia                     |
| -------------------------- | ------------------------------ |
| Linguagem/Framework        | C# / ASP.NET Core 8 (LTS)      |
| ORM                        | Entity Framework Core (Npgsql) |
| Banco de dados             | PostgreSQL 16                  |
| Documentação de API        | Swagger / OpenAPI              |
| Comunicação entre serviços | HttpClient (REST)              |
| Testes                     | xUnit                          |
| Containerização            | Docker + Docker Compose        |
| CI/CD                      | GitHub Actions                 |

---

## Microsserviços

### 1. SensorService

**Responsabilidade:** registrar e disponibilizar as leituras enviadas pelos sensores ambientais.

- Recebe leituras (`POST /readings`) e as persiste em `sensores_db`.
- Disponibiliza a consulta das leituras (`GET /readings` e `GET /readings/{id}`).
- É a **fonte da verdade** dos dados ambientais; não conhece o conceito de "alerta".

**Modelo de dados – `Reading`:**

| Campo     | Tipo     | Descrição                                      |
| --------- | -------- | ---------------------------------------------- |
| id        | int      | Identificador (gerado automaticamente)         |
| sensorId  | string   | Identificador do sensor de origem              |
| type      | string   | Tipo da leitura (`air_quality`, `temperature`) |
| value     | double   | Valor medido                                   |
| unit      | string   | Unidade (`AQI`, `°C`)                          |
| location  | string   | Localização do sensor                          |
| timestamp | datetime | Data/hora da medição (UTC)                     |

### 2. AlertService

**Responsabilidade:** avaliar leituras contra limites de segurança e gerar alertas.

- Avalia uma leitura recebida diretamente (`POST /alerts`) **ou** busca uma leitura no `SensorService` via HTTP e avalia (`POST /alerts/evaluate/{readingId}`).
- Persiste os alertas gerados em `alertas_db`.
- Disponibiliza a consulta dos alertas (`GET /alerts`).

**Limites configuráveis (regras de negócio):**

| Tipo        | Limite    | Severidade |
| ----------- | --------- | ---------- |
| air_quality | > 100 AQI | alta       |
| temperature | > 35 °C   | média      |

**Modelo de dados – `Alert`:**

| Campo     | Tipo     | Descrição                              |
| --------- | -------- | -------------------------------------- |
| id        | int      | Identificador (gerado automaticamente) |
| readingId | int      | Leitura que originou o alerta          |
| type      | string   | Tipo da leitura avaliada               |
| value     | double   | Valor que disparou o alerta            |
| threshold | double   | Limite ultrapassado                    |
| severity  | string   | Severidade (`alta`, `média`)           |
| message   | string   | Mensagem descritiva                    |
| createdAt | datetime | Data/hora de criação (UTC)             |

---

## Documentação das APIs

> Cada serviço também expõe a documentação interativa via Swagger em `/swagger`.

### SensorService (`http://localhost:5001`)

#### `GET /health`

Verifica se o serviço está no ar (usado pelo pipeline de CI).

- **Resposta 200:** `{ "status": "ok" }`

#### `GET /readings`

Lista todas as leituras registradas.

- **Parâmetros (query, opcionais):** `type`, `sensorId`
- **Resposta 200:**

```json
[
  {
    "id": 1,
    "sensorId": "sensor-01",
    "type": "air_quality",
    "value": 132.5,
    "unit": "AQI",
    "location": "Maricá - RJ",
    "timestamp": "2026-07-08T14:00:00Z"
  }
]
```

#### `GET /readings/{id}`

Retorna uma leitura específica.

- **Parâmetros (rota):** `id` (int)
- **Resposta 200:** objeto `Reading` · **Resposta 404:** leitura não encontrada

#### `POST /readings`

Registra uma nova leitura.

- **Corpo (JSON):**

```json
{
  "sensorId": "sensor-01",
  "type": "air_quality",
  "value": 132.5,
  "unit": "AQI",
  "location": "Maricá - RJ"
}
```

- **Resposta 201:** objeto `Reading` criado (com `id` e `timestamp`)

---

### AlertService (`http://localhost:5002`)

#### `GET /health`

Verifica se o serviço está no ar (usado pelo pipeline de CI).

- **Resposta 200:** `{ "status": "ok" }`

#### `GET /alerts`

Lista todos os alertas gerados.

- **Resposta 200:**

```json
[
  {
    "id": 1,
    "readingId": 1,
    "type": "air_quality",
    "value": 132.5,
    "threshold": 100,
    "severity": "alta",
    "message": "Qualidade do ar acima do limite seguro",
    "createdAt": "2026-07-08T14:00:05Z"
  }
]
```

#### `POST /alerts`

Avalia uma leitura enviada no corpo da requisição e cria um alerta se o limite for ultrapassado.

- **Corpo (JSON):**

```json
{
  "readingId": 1,
  "type": "air_quality",
  "value": 132.5
}
```

- **Resposta 201:** objeto `Alert` criado · **Resposta 200 (sem alerta):** valor dentro do limite

#### `POST /alerts/evaluate/{readingId}`

Busca a leitura no **SensorService** via HTTP, avalia contra os limites e cria o alerta se necessário. **Este endpoint demonstra a comunicação entre os microsserviços.**

- **Parâmetros (rota):** `readingId` (int)
- **Resposta 201:** objeto `Alert` criado · **Resposta 404:** leitura não encontrada no SensorService

**Exemplos com `curl`:**

```bash
# 1. Registrar uma leitura no SensorService
curl -X POST http://localhost:5001/readings \
  -H "Content-Type: application/json" \
  -d '{"sensorId":"sensor-01","type":"air_quality","value":132.5,"unit":"AQI","location":"Maricá - RJ"}'

# 2. Pedir ao AlertService para avaliar essa leitura (comunicação entre serviços)
curl -X POST http://localhost:5002/alerts/evaluate/1

# 3. Listar os alertas gerados
curl http://localhost:5002/alerts
```

---

## Como rodar o projeto localmente

### Pré-requisitos

- [Docker](https://docs.docker.com/get-docker/) e Docker Compose
- (Opcional, para rodar sem container) [.NET SDK 8](https://dotnet.microsoft.com/download)

### Passo único (recomendado)

```bash
git clone https://github.com/<usuario>/plataforma-monitoramento-ambiental.git
cd plataforma-monitoramento-ambiental
docker compose up --build
```

Cada serviço sobe com seu próprio PostgreSQL e volume local no diretório `src/<Servico>/data/`.

Serviços disponíveis após a subida:

- SensorService → <http://localhost:5001/swagger>
- AlertService → <http://localhost:5002/swagger>

Para verificar os containers em execução:

```bash
docker ps
```

Para parar tudo:

```bash
docker compose down
```

### Rodando um serviço isoladamente (sem Docker)

```bash
cd src/SensorService
dotnet restore
dotnet run
```

### Executando os testes

```bash
dotnet test
```

---

## Containerização com Docker

Cada microsserviço possui um `Dockerfile` próprio (build multi-stage). O arquivo `docker-compose.yml` na raiz orquestra os quatro containers (dois serviços + dois bancos).

**Exemplo de `Dockerfile` (SensorService):**

```dockerfile
# Etapa de build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore
RUN dotnet publish -c Release -o /app

# Etapa de runtime
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app .
EXPOSE 8080
ENTRYPOINT ["dotnet", "SensorService.dll"]
```

**Exemplo de `docker-compose.yml`:**

```yaml
services:
  sensor-db:
    image: postgres:16
    environment:
      POSTGRES_DB: sensores_db
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes: ["sensor_data:/var/lib/postgresql/data"]

  alert-db:
    image: postgres:16
    environment:
      POSTGRES_DB: alertas_db
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes: ["alert_data:/var/lib/postgresql/data"]

  sensor-service:
    build: ./src/SensorService
    ports: ["5001:8080"]
    environment:
      ConnectionStrings__Default: "Host=sensor-db;Database=sensores_db;Username=app;Password=${DB_PASSWORD}"
    depends_on: [sensor-db]

  alert-service:
    build: ./src/AlertService
    ports: ["5002:8080"]
    environment:
      ConnectionStrings__Default: "Host=alert-db;Database=alertas_db;Username=app;Password=${DB_PASSWORD}"
      SensorService__BaseUrl: "http://sensor-service:8080"
    depends_on: [alert-db, sensor-service]

volumes:
  sensor_data:
  alert_data:
```

> As senhas e segredos são lidos de variáveis de ambiente (arquivo `.env`, **não versionado**), nunca fixos no código.

---

## Pipeline CI/CD

Arquivo `.github/workflows/ci.yml`. É acionado a cada `push` na branch `main` e executa: instalação de dependências, build e testes (incluindo um teste de saúde da API).

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configurar .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: "8.0.x"

      - name: Restaurar dependências
        run: dotnet restore

      - name: Compilar
        run: dotnet build --no-restore --configuration Release

      - name: Executar testes
        run: dotnet test --no-build --configuration Release --verbosity normal
```

> Após o push, verifique o pipeline **verde** na aba **Actions** do GitHub e tire o print para o relatório.

---

## Segurança e conformidade com a LGPD

**Dados pessoais processados:** o sistema foi desenhado para lidar essencialmente com **dados ambientais**, que não são pessoais. Eventuais dados pessoais restringem-se a credenciais de operadores/administradores do sistema (e-mail e senha), tratados sob o princípio da **minimização de dados**.

**Autenticação:** as rotas de escrita (`POST`) são (ou seriam) protegidas por autenticação **JWT** (`Microsoft.AspNetCore.Authentication.JwtBearer`). Cada requisição precisa de um token válido; as senhas são armazenadas com **hash** (nunca em texto puro).

**Medidas de segurança / conformidade com a LGPD:**

- Comunicação sobre **HTTPS** em ambiente de produção.
- **Segredos** (senhas de banco, chaves) fora do código, em variáveis de ambiente.
- **Minimização de dados:** coleta apenas do estritamente necessário.
- **Controle de acesso** por autenticação e autorização por perfil.
- **Registro de acesso** (logs) para rastreabilidade, sem expor dados sensíveis.

---

## Estrutura do repositório

```
plataforma-monitoramento-ambiental/
├── README.md
├── docker-compose.yml
├── .env.example
├── .github/
│   └── workflows/
│       └── ci.yml
└── src/
    ├── SensorService/
    │   ├── SensorService.csproj
    │   ├── Program.cs
    │   ├── Dockerfile
    │   ├── Models/Reading.cs
    │   └── Data/SensorDbContext.cs
    └── AlertService/
        ├── AlertService.csproj
        ├── Program.cs
        ├── Dockerfile
        ├── Models/Alert.cs
        ├── Data/AlertDbContext.cs
        └── Services/SensorClient.cs
```

---

## Equipe

| Nome                        | Matrícula (RA) |
| --------------------------- | -------------- |
| Pedro Arnaldo Madeira Lopes | 1230201064     |

**Curso:** Engenharia de Computação — Universidade Veiga de Almeida (UVA)
**Disciplina:** Projeto de Extensão 9

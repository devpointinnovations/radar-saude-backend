# Radar Saúde — Backend

API REST em Spring Boot 4 com Java 25, responsavel por receber e persistir dados de autenticacao no MongoDB (Percona Server for MongoDB 7.0).

## Repositórios do projeto

| Repositório | Conteúdo |
|---|---|
| [radar-saude-mobile](https://github.com/devpointinnovations/radar-saude-mobile) | App Expo / React Native |
| [painel-admin-frontend](https://github.com/devpointinnovations/painel-admin-frontend) | Painel Admin Angular |
| **radar-saude-backend** (este) | API Spring Boot, deploy no Cloud Run (`deploy/google/`), testes de desempenho (`perf/`) |
| [radar-saude-docs](https://github.com/devpointinnovations/radar-saude-docs) | Documentação, ADRs, backlog, registros de homologação e site |

Até 29/10/2026 o backend era a pasta `backend/` do monorepo
[GabrielVogado/saude-monitor](https://github.com/GabrielVogado/saude-monitor). O histórico
foi preservado aqui; PRs antigos citados nas mensagens de commit (`#123`) são daquele
repositório. Referências a `backend/...` e `Documentos/...` em comentários antigos apontam,
agora, para a raiz deste repositório e para o radar-saude-docs.

---

## Pre-requisitos

| Ferramenta | Versao minima |
|---|---|
| Docker Desktop | 4.x |
| Docker Compose | v2 |

---

## Configuracao do ambiente

### 1. Variaveis de ambiente

Copie o arquivo de exemplo e ajuste as credenciais se necessario:

```powershell
cd D:\radar-saude-backend
Copy-Item .env.example .env
```

Variaveis disponiveis em `.env`:

| Variavel | Padrao |
|---|---|
| `MONGO_HOST` | `localhost` |
| `MONGO_PORT` | `27017` |
| `MONGO_DATABASE` | `saude_monitor` |
| `MONGO_AUTH_DB` | `admin` |
| `MONGO_USER` | `saude_monitor_user` |
| `MONGO_PASSWORD` | `saude_monitor_pass` |

---

## Subindo o ambiente com Docker (backend + banco)

### 2. Subir tudo com um comando

```powershell
cd D:\radar-saude-backend
docker compose --env-file .env up -d --build
```

Ver containers e status:

```powershell
docker compose ps
```

Ver logs do backend:

```powershell
docker compose logs -f backend
```

### 3. Validacao rapida da API

```powershell
$headers = @{"Content-Type"="application/json"}
$body    = '{"email":"usuario@teste.com","password":"senha123","rememberDevice":true}'
Invoke-WebRequest -Uri "http://localhost:8080/api/v1/auth/login" `
  -Method POST -Headers $headers -Body $body -UseBasicParsing |
  Select-Object -ExpandProperty Content
```

### 4. Opcional: executar backend local sem Docker

Se quiser rodar apenas o banco no Docker e o backend pela IDE/Gradle, voce precisa do Java 25 instalado:

```powershell
cd D:\radar-saude-backend
docker compose --env-file .env up -d percona-mongodb
Get-Content .env | ForEach-Object {
  if ($_ -match '^(?<k>[^#=]+)=(?<v>.*)$') {
    [System.Environment]::SetEnvironmentVariable($matches['k'], $matches['v'], 'Process')
  }
}
.\gradlew.bat bootRun
```

---

## Endpoints

### POST `/api/v1/auth/login`

Autentica e-mail/senha (hash BCrypt contra `senhaHash` em `users`) e emite um par de
tokens JWT. Contrato completo em `Documentos/02-arquitetura-tecnica/Especificacao-API-v2.1.md` §3.1.

**Payload:**
```json
{
  "email": "usuario@teste.com",
  "password": "senha123"
}
```

**Resposta 200 OK:**
```json
{
  "accessToken": "eyJhbGciOi...",
  "refreshToken": "eyJhbGciOi...",
  "expiraEm": 900,
  "usuario": { "id": "...", "nome": "Marina Souza", "email": "usuario@teste.com", "papel": "USER" }
}
```

**Resposta 401** (`NAO_AUTORIZADO`): e-mail ou senha inválidos.

**Teste rapido:**
```powershell
$headers = @{"Content-Type"="application/json"}
$body    = '{"email":"usuario@teste.com","password":"senha123"}'
Invoke-WebRequest -Uri "http://localhost:8080/api/v1/auth/login" `
  -Method POST -Headers $headers -Body $body -UseBasicParsing |
  Select-Object -ExpandProperty Content
# Obs: Espera-se um erro 401 caso esse usuário não tenha sido previamente cadastrado
# (POST /api/v1/auth/registro) no banco
```

---

## Testes automatizados

```powershell
cd D:\radar-saude-backend
.\gradlew.bat test
```

---

## Parar o ambiente

```powershell
cd D:\radar-saude-backend
docker compose down
```

Para remover tambem o volume de dados:

```powershell
docker compose down -v
```

# Deploy
O deploy do backend e disparado por push em backend/ (workflow cd-backend-google.yml),
no Google Cloud Run. Ver deploy/google/README.md.

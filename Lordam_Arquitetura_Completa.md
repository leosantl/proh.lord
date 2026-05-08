# Sistema Lordam — Arquitetura Técnica Completa

**Versão:** 1.0  
**Data:** Maio/2026  
**Arquiteto:** Claude (Anthropic)

---

## 1. Visão Geral da Arquitetura

O sistema Lordam é uma plataforma SaaS de gestão comercial com três camadas principais:

- **Frontend** — Next.js 14 com App Router, TypeScript e Tailwind CSS
- **Backend** — Python com FastAPI, Celery para tarefas assíncronas
- **Dados** — PostgreSQL (principal), Redis (cache + filas), S3-compatible (arquivos)

---

## 2. Estrutura de Pastas do Projeto

```
lordam/
├── backend/
│   ├── app/
│   │   ├── api/
│   │   │   └── v1/
│   │   │       ├── auth.py
│   │   │       ├── clients.py
│   │   │       ├── contracts.py
│   │   │       ├── deals.py
│   │   │       ├── services.py
│   │   │       └── calendar.py
│   │   ├── core/
│   │   │   ├── config.py          # Settings com pydantic-settings
│   │   │   ├── security.py        # JWT + hashing
│   │   │   └── deps.py            # Dependências FastAPI
│   │   ├── db/
│   │   │   ├── base.py            # SQLAlchemy base
│   │   │   ├── session.py         # Async session factory
│   │   │   └── models/
│   │   │       ├── user.py
│   │   │       ├── client.py
│   │   │       ├── deal.py
│   │   │       ├── contract.py
│   │   │       ├── service.py
│   │   │       └── calendar_event.py
│   │   ├── schemas/               # Pydantic v2 schemas
│   │   ├── services/
│   │   │   ├── ai_service.py      # Integração Claude API
│   │   │   ├── contract_service.py
│   │   │   ├── calendar_service.py
│   │   │   └── pdf_service.py     # WeasyPrint
│   │   ├── tasks/                 # Celery tasks
│   │   │   ├── celery_app.py
│   │   │   ├── contract_tasks.py
│   │   │   └── calendar_tasks.py
│   │   └── main.py
│   ├── alembic/                   # Migrations
│   ├── tests/
│   ├── Dockerfile
│   └── requirements.txt
│
├── frontend/
│   ├── app/
│   │   ├── (auth)/
│   │   │   └── login/
│   │   ├── (dashboard)/
│   │   │   ├── clientes/
│   │   │   ├── contratos/
│   │   │   ├── pipeline/
│   │   │   ├── servicos/
│   │   │   └── agenda/
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/
│   │   ├── ui/                    # shadcn/ui components
│   │   ├── forms/
│   │   └── pipeline/
│   ├── lib/
│   │   ├── api.ts                 # Axios instance
│   │   └── auth.ts                # NextAuth config
│   ├── Dockerfile
│   └── package.json
│
├── docker-compose.yml
├── docker-compose.prod.yml
└── .env.example
```

---

## 3. Modelo de Banco de Dados

### 3.1 Tabela: users

```sql
CREATE TABLE users (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    email       VARCHAR(255) UNIQUE NOT NULL,
    role        VARCHAR(50) NOT NULL DEFAULT 'agent',  -- admin | agent | viewer
    hashed_password TEXT NOT NULL,
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMPTZ DEFAULT NOW(),
    updated_at  TIMESTAMPTZ DEFAULT NOW()
);
```

### 3.2 Tabela: clients

```sql
CREATE TABLE clients (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_by   UUID REFERENCES users(id),
    full_name    VARCHAR(255) NOT NULL,
    cpf_cnpj     VARCHAR(20) UNIQUE NOT NULL,
    email        VARCHAR(255),
    phone        VARCHAR(30),
    status       VARCHAR(50) DEFAULT 'lead',  -- lead | active | finished
    address      JSONB,
    notes        TEXT,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    updated_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_clients_status ON clients(status);
CREATE INDEX idx_clients_cpf_cnpj ON clients(cpf_cnpj);
```

### 3.3 Tabela: services

```sql
CREATE TABLE services (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name             VARCHAR(255) NOT NULL,
    type             VARCHAR(100) NOT NULL,  -- intermediacao | venda | consultoria
    description      TEXT,
    price            NUMERIC(12,2) NOT NULL,
    commission_rate  NUMERIC(5,2) DEFAULT 0,  -- percentual ex: 10.00
    active           BOOLEAN DEFAULT TRUE,
    created_at       TIMESTAMPTZ DEFAULT NOW()
);
```

### 3.4 Tabela: deals (pipeline)

```sql
CREATE TABLE deals (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id        UUID NOT NULL REFERENCES clients(id) ON DELETE CASCADE,
    service_id       UUID NOT NULL REFERENCES services(id),
    owner_id         UUID NOT NULL REFERENCES users(id),
    stage            VARCHAR(50) DEFAULT 'lead',
                     -- lead | negotiating | contract_generated | in_execution | finished
    value            NUMERIC(12,2),
    commission_value NUMERIC(12,2) GENERATED ALWAYS AS
                     (value * (SELECT commission_rate/100 FROM services s WHERE s.id = service_id)) STORED,
    notes            TEXT,
    expected_close   DATE,
    created_at       TIMESTAMPTZ DEFAULT NOW(),
    updated_at       TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_deals_stage ON deals(stage);
CREATE INDEX idx_deals_client_id ON deals(client_id);
```

### 3.5 Tabela: contracts

```sql
CREATE TABLE contracts (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id        UUID NOT NULL REFERENCES deals(id),
    client_id      UUID NOT NULL REFERENCES clients(id),
    type           VARCHAR(50) NOT NULL,  -- service | sale
    status         VARCHAR(50) DEFAULT 'draft',  -- draft | pending_signature | signed | cancelled
    body_html      TEXT NOT NULL,
    pdf_url        TEXT,
    signed_url     TEXT,
    ai_generated   BOOLEAN DEFAULT FALSE,
    generated_at   TIMESTAMPTZ DEFAULT NOW(),
    signed_at      TIMESTAMPTZ,
    expires_at     TIMESTAMPTZ
);
```

### 3.6 Tabela: calendar_events

```sql
CREATE TABLE calendar_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deal_id         UUID REFERENCES deals(id),
    client_id       UUID REFERENCES clients(id),
    google_event_id VARCHAR(255),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    starts_at       TIMESTAMPTZ NOT NULL,
    ends_at         TIMESTAMPTZ NOT NULL,
    meet_link       TEXT,
    created_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### 3.7 Tabela: activity_log

```sql
CREATE TABLE activity_log (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_id    UUID NOT NULL,
    entity_type  VARCHAR(100) NOT NULL,  -- client | deal | contract
    user_id      UUID REFERENCES users(id),
    action       VARCHAR(100) NOT NULL,
    metadata     JSONB,
    created_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_activity_entity ON activity_log(entity_id, entity_type);
```

---

## 4. Endpoints da API REST

### Autenticação

| Método | Endpoint                  | Descrição                        |
|--------|---------------------------|----------------------------------|
| POST   | /api/v1/auth/login        | Login com e-mail e senha         |
| POST   | /api/v1/auth/refresh      | Renovar access token             |
| GET    | /api/v1/auth/me           | Perfil do usuário autenticado    |
| GET    | /api/v1/auth/google       | Iniciar OAuth com Google         |

### Clientes

| Método | Endpoint                        | Descrição                    |
|--------|---------------------------------|------------------------------|
| GET    | /api/v1/clients                 | Listar (filtros: status, q)  |
| POST   | /api/v1/clients                 | Criar cliente                |
| GET    | /api/v1/clients/{id}            | Detalhe do cliente           |
| PUT    | /api/v1/clients/{id}            | Atualizar                    |
| DELETE | /api/v1/clients/{id}            | Remover                      |
| GET    | /api/v1/clients/{id}/history    | Histórico de deals/contratos |

### Serviços

| Método | Endpoint                | Descrição               |
|--------|-------------------------|-------------------------|
| GET    | /api/v1/services        | Listar serviços         |
| POST   | /api/v1/services        | Cadastrar serviço       |
| PUT    | /api/v1/services/{id}   | Atualizar               |

### Deals / Pipeline

| Método | Endpoint                        | Descrição                        |
|--------|---------------------------------|----------------------------------|
| GET    | /api/v1/deals                   | Listar (filtros: stage, owner)   |
| POST   | /api/v1/deals                   | Criar deal                       |
| PATCH  | /api/v1/deals/{id}/stage        | Mover para outra etapa           |
| GET    | /api/v1/deals/kanban            | Retorna agrupado por stage       |

### Contratos

| Método | Endpoint                             | Descrição                          |
|--------|--------------------------------------|------------------------------------|
| POST   | /api/v1/contracts/generate           | Gerar contrato (com ou sem IA)     |
| GET    | /api/v1/contracts/{id}               | Detalhe do contrato                |
| PUT    | /api/v1/contracts/{id}               | Editar corpo do contrato           |
| POST   | /api/v1/contracts/{id}/export-pdf    | Gerar PDF (async)                  |
| POST   | /api/v1/contracts/{id}/sign          | Registrar assinatura               |
| POST   | /api/v1/contracts/ai/improve         | IA melhora linguagem jurídica      |
| POST   | /api/v1/contracts/ai/summarize       | IA resume o contrato               |

### Agenda / Google Calendar

| Método | Endpoint                              | Descrição                         |
|--------|---------------------------------------|-----------------------------------|
| GET    | /api/v1/calendar/events               | Listar eventos                    |
| POST   | /api/v1/calendar/events               | Criar evento (+ Google Cal)       |
| DELETE | /api/v1/calendar/events/{id}          | Remover evento                    |
| GET    | /api/v1/calendar/oauth/authorize      | URL de autorização Google         |
| POST   | /api/v1/calendar/oauth/callback       | Processar callback OAuth          |

---

## 5. Código — Exemplos Reais

### 5.1 Backend: modelo SQLAlchemy (Deal)

```python
# app/db/models/deal.py
from sqlalchemy import Column, String, Numeric, Date, ForeignKey, text
from sqlalchemy.dialects.postgresql import UUID, JSONB
from sqlalchemy.orm import relationship
from app.db.base import Base
import uuid

class Deal(Base):
    __tablename__ = "deals"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    client_id = Column(UUID(as_uuid=True), ForeignKey("clients.id", ondelete="CASCADE"), nullable=False)
    service_id = Column(UUID(as_uuid=True), ForeignKey("services.id"), nullable=False)
    owner_id = Column(UUID(as_uuid=True), ForeignKey("users.id"), nullable=False)
    stage = Column(String(50), nullable=False, default="lead",
                   server_default="lead")
    value = Column(Numeric(12, 2))
    notes = Column(String)
    expected_close = Column(Date)

    # Relationships
    client = relationship("Client", back_populates="deals")
    service = relationship("Service")
    owner = relationship("User")
    contracts = relationship("Contract", back_populates="deal")
    calendar_events = relationship("CalendarEvent", back_populates="deal")
```

### 5.2 Backend: endpoint de geração de contrato

```python
# app/api/v1/contracts.py
from fastapi import APIRouter, Depends, HTTPException, BackgroundTasks
from sqlalchemy.ext.asyncio import AsyncSession
from app.core.deps import get_db, get_current_user
from app.schemas.contract import ContractGenerateRequest, ContractResponse
from app.services.contract_service import ContractService
from app.services.ai_service import AIService

router = APIRouter(prefix="/contracts", tags=["contracts"])

@router.post("/generate", response_model=ContractResponse, status_code=201)
async def generate_contract(
    request: ContractGenerateRequest,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db),
    current_user = Depends(get_current_user),
):
    """
    Gera um contrato. Se use_ai=True, usa Claude API para redigir
    as cláusulas com linguagem jurídica otimizada.
    """
    contract_service = ContractService(db)
    ai_service = AIService()

    # Busca deal e cliente
    deal = await contract_service.get_deal_with_relations(request.deal_id)
    if not deal:
        raise HTTPException(status_code=404, detail="Deal não encontrado")

    if request.use_ai:
        body_html = await ai_service.generate_contract(
            client=deal.client,
            service=deal.service,
            deal=deal,
            contract_type=request.contract_type,
        )
    else:
        body_html = contract_service.render_template(
            deal=deal,
            contract_type=request.contract_type,
            custom_clauses=request.custom_clauses,
        )

    contract = await contract_service.create(
        deal_id=deal.id,
        client_id=deal.client_id,
        contract_type=request.contract_type,
        body_html=body_html,
        ai_generated=request.use_ai,
    )

    # Gera PDF em background
    background_tasks.add_task(contract_service.generate_pdf, contract.id)

    return contract
```

### 5.3 Backend: serviço de IA (Claude API)

```python
# app/services/ai_service.py
import anthropic
from app.core.config import settings
from app.db.models import Client, Service, Deal

class AIService:
    def __init__(self):
        self.client = anthropic.Anthropic(api_key=settings.ANTHROPIC_API_KEY)

    async def generate_contract(
        self,
        client: Client,
        service: Service,
        deal: Deal,
        contract_type: str,  # "service" | "sale"
    ) -> str:
        """Gera o corpo HTML do contrato usando Claude."""

        prompt = f"""
Você é um advogado especialista em contratos comerciais brasileiros.
Redija um contrato profissional de {contract_type} com as seguintes informações:

**CONTRATANTE:**
Lordam Intermediações Ltda.

**CONTRATADO / CLIENTE:**
Nome: {client.full_name}
CPF/CNPJ: {client.cpf_cnpj}
E-mail: {client.email}

**SERVIÇO:**
{service.name} — {service.description}
Valor: R$ {deal.value:,.2f}

**CLÁUSULAS OBRIGATÓRIAS:**
1. Objeto do contrato
2. Valor e forma de pagamento (à vista / parcelado)
3. Prazo de execução
4. Multa por rescisão antecipada (20% do valor)
5. Cláusula de sigilo e confidencialidade
6. Foro: cidade de São Paulo — SP

Formato: HTML semântico limpo, com tags <h2>, <p>, <ol>, <li>.
Linguagem: formal, objetiva e juridicamente precisa.
Não inclua instruções ou comentários, apenas o contrato.
"""

        message = self.client.messages.create(
            model="claude-opus-4-6",
            max_tokens=4096,
            messages=[{"role": "user", "content": prompt}],
        )

        return message.content[0].text

    async def improve_language(self, contract_html: str) -> str:
        """Melhora a linguagem jurídica de um contrato existente."""
        message = self.client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            messages=[{
                "role": "user",
                "content": f"Melhore a linguagem jurídica deste contrato, "
                           f"tornando-o mais formal e preciso, sem alterar o conteúdo:\n\n{contract_html}"
            }],
        )
        return message.content[0].text

    async def summarize_contract(self, contract_html: str) -> str:
        """Gera um resumo executivo do contrato."""
        message = self.client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=800,
            messages=[{
                "role": "user",
                "content": f"Faça um resumo executivo em 5 pontos deste contrato:\n\n{contract_html}"
            }],
        )
        return message.content[0].text
```

### 5.4 Backend: integração Google Calendar

```python
# app/services/calendar_service.py
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
from google_auth_oauthlib.flow import Flow
from app.core.config import settings
import json

SCOPES = ["https://www.googleapis.com/auth/calendar.events"]

class GoogleCalendarService:
    def __init__(self, user_tokens: dict):
        creds = Credentials(
            token=user_tokens["access_token"],
            refresh_token=user_tokens["refresh_token"],
            token_uri="https://oauth2.googleapis.com/token",
            client_id=settings.GOOGLE_CLIENT_ID,
            client_secret=settings.GOOGLE_CLIENT_SECRET,
        )
        self.service = build("calendar", "v3", credentials=creds)

    def create_event(
        self,
        title: str,
        description: str,
        start_iso: str,   # "2026-05-10T10:00:00-03:00"
        end_iso: str,
        attendee_email: str,
        add_meet: bool = True,
    ) -> dict:
        event_body = {
            "summary": title,
            "description": description,
            "start": {"dateTime": start_iso, "timeZone": "America/Sao_Paulo"},
            "end": {"dateTime": end_iso, "timeZone": "America/Sao_Paulo"},
            "attendees": [{"email": attendee_email}],
            "reminders": {
                "useDefault": False,
                "overrides": [
                    {"method": "email", "minutes": 24 * 60},
                    {"method": "popup", "minutes": 30},
                ],
            },
        }

        if add_meet:
            event_body["conferenceData"] = {
                "createRequest": {
                    "requestId": f"lordam-{title[:20]}",
                    "conferenceSolutionKey": {"type": "hangoutsMeet"},
                }
            }

        result = self.service.events().insert(
            calendarId="primary",
            body=event_body,
            conferenceDataVersion=1,
            sendUpdates="all",
        ).execute()

        return {
            "google_event_id": result["id"],
            "meet_link": result.get("conferenceData", {})
                              .get("entryPoints", [{}])[0]
                              .get("uri", ""),
        }
```

### 5.5 Backend: configuração (settings)

```python
# app/core/config.py
from pydantic_settings import BaseSettings
from typing import List

class Settings(BaseSettings):
    # App
    APP_NAME: str = "Lordam"
    DEBUG: bool = False
    SECRET_KEY: str
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 60
    REFRESH_TOKEN_EXPIRE_DAYS: int = 30

    # Database
    DATABASE_URL: str  # postgresql+asyncpg://user:pass@host/db
    
    # Redis
    REDIS_URL: str = "redis://redis:6379/0"

    # AWS S3 / compatible
    S3_BUCKET: str
    S3_ENDPOINT: str
    AWS_ACCESS_KEY_ID: str
    AWS_SECRET_ACCESS_KEY: str

    # Google OAuth
    GOOGLE_CLIENT_ID: str
    GOOGLE_CLIENT_SECRET: str
    GOOGLE_REDIRECT_URI: str

    # Anthropic
    ANTHROPIC_API_KEY: str

    # CORS
    ALLOWED_ORIGINS: List[str] = ["http://localhost:3000"]

    class Config:
        env_file = ".env"

settings = Settings()
```

### 5.6 Frontend: hook para o pipeline Kanban

```typescript
// components/pipeline/usePipeline.ts
import { useState, useCallback } from "react"
import { api } from "@/lib/api"
import { Deal, PipelineStage } from "@/types"

const STAGES: PipelineStage[] = [
  { id: "lead",               label: "Lead",              color: "gray" },
  { id: "negotiating",        label: "Em Negociação",     color: "blue" },
  { id: "contract_generated", label: "Contrato Gerado",   color: "amber" },
  { id: "in_execution",       label: "Em Execução",       color: "teal" },
  { id: "finished",           label: "Finalizado",        color: "green" },
]

export function usePipeline() {
  const [deals, setDeals] = useState<Record<string, Deal[]>>({})

  const moveCard = useCallback(async (dealId: string, newStage: string) => {
    // Optimistic update
    setDeals(prev => {
      const updated = { ...prev }
      for (const stage of Object.keys(updated)) {
        const idx = updated[stage].findIndex(d => d.id === dealId)
        if (idx !== -1) {
          const [deal] = updated[stage].splice(idx, 1)
          updated[newStage] = [...(updated[newStage] ?? []), { ...deal, stage: newStage }]
          break
        }
      }
      return updated
    })

    // Persist
    await api.patch(`/deals/${dealId}/stage`, { stage: newStage })
  }, [])

  return { deals, stages: STAGES, moveCard }
}
```

---

## 6. Integração Google Calendar — Passo a Passo Técnico

### Passo 1 — Criar projeto no Google Cloud Console

1. Acesse console.cloud.google.com
2. Crie um projeto chamado "Lordam"
3. Ative a API: **Google Calendar API**
4. Em "Credenciais" → "Criar credenciais" → "ID do cliente OAuth 2.0"
5. Tipo: Aplicativo Web
6. URIs autorizados: `https://seudominio.com/api/v1/calendar/oauth/callback`
7. Salve o `client_id` e `client_secret` no `.env`

### Passo 2 — Fluxo OAuth no backend

```python
# Endpoint: GET /api/v1/calendar/oauth/authorize
@router.get("/oauth/authorize")
async def calendar_authorize(current_user = Depends(get_current_user)):
    flow = Flow.from_client_config(
        {
            "web": {
                "client_id": settings.GOOGLE_CLIENT_ID,
                "client_secret": settings.GOOGLE_CLIENT_SECRET,
                "auth_uri": "https://accounts.google.com/o/oauth2/auth",
                "token_uri": "https://oauth2.googleapis.com/token",
            }
        },
        scopes=SCOPES,
        redirect_uri=settings.GOOGLE_REDIRECT_URI,
    )
    auth_url, state = flow.authorization_url(
        access_type="offline",
        include_granted_scopes="true",
        prompt="consent",
    )
    # Salva state na sessão para validar no callback
    return {"auth_url": auth_url}

# Endpoint: POST /api/v1/calendar/oauth/callback
@router.post("/oauth/callback")
async def calendar_callback(
    code: str,
    state: str,
    db: AsyncSession = Depends(get_db),
    current_user = Depends(get_current_user),
):
    flow = Flow.from_client_config(...)
    flow.fetch_token(code=code)
    creds = flow.credentials

    # Salva tokens criptografados no banco (associados ao usuário)
    await save_user_google_tokens(db, current_user.id, {
        "access_token": creds.token,
        "refresh_token": creds.refresh_token,
    })
    return {"status": "connected"}
```

### Passo 3 — Criar evento automaticamente ao gerar contrato

```python
# Em contract_service.py, após criar o contrato:
async def after_contract_created(self, contract_id: str, deal: Deal):
    tokens = await get_user_google_tokens(self.db, deal.owner_id)
    if tokens:
        cal = GoogleCalendarService(tokens)
        result = cal.create_event(
            title=f"Lordam · {deal.service.name} — {deal.client.full_name}",
            description=f"Contrato gerado. Deal ID: {deal.id}",
            start_iso=deal.expected_close.isoformat() + "T09:00:00-03:00",
            end_iso=deal.expected_close.isoformat() + "T10:00:00-03:00",
            attendee_email=deal.client.email,
        )
        # Persiste o evento no banco
        await self.create_calendar_event(
            deal_id=deal.id,
            client_id=deal.client_id,
            google_event_id=result["google_event_id"],
            meet_link=result["meet_link"],
        )
```

---

## 7. Segurança e Boas Práticas

### Autenticação e Autorização

- Tokens JWT com expiração curta (60 min) + refresh token (30 dias)
- Senhas com bcrypt (cost factor 12)
- RBAC simples: `admin`, `agent`, `viewer`
- Middleware que verifica role em cada endpoint sensível

### Proteção de Dados

- Todos os tokens Google salvos **criptografados** no banco (Fernet)
- HTTPS obrigatório (Nginx com certificado Let's Encrypt)
- Variáveis sensíveis apenas via `.env` — nunca no código
- Rate limiting no Nginx: 100 req/min por IP

### Banco de Dados

- Conexões via pool assíncrono (`asyncpg`)
- Migrations gerenciadas pelo Alembic (nunca DDL manual)
- Backups automáticos diários com retenção de 30 dias
- Índices em todas as colunas usadas em filtros

### Logging e Monitoramento

- `structlog` para logs estruturados em JSON
- Sentry para rastreamento de erros em produção
- Prometheus + Grafana para métricas de performance

---

## 8. Docker Compose (Desenvolvimento)

```yaml
# docker-compose.yml
version: "3.9"

services:
  backend:
    build: ./backend
    ports: ["8000:8000"]
    environment:
      - DATABASE_URL=postgresql+asyncpg://lordam:lordam@db/lordam
      - REDIS_URL=redis://redis:6379/0
    volumes: ["./backend:/app"]
    depends_on: [db, redis]
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  worker:
    build: ./backend
    command: celery -A app.tasks.celery_app worker -l info
    environment:
      - REDIS_URL=redis://redis:6379/0
    depends_on: [db, redis]

  frontend:
    build: ./frontend
    ports: ["3000:3000"]
    volumes: ["./frontend:/app"]
    command: npm run dev

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: lordam
      POSTGRES_PASSWORD: lordam
      POSTGRES_DB: lordam
    volumes: ["pgdata:/var/lib/postgresql/data"]
    ports: ["5432:5432"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes:
  pgdata:
```

---

## 9. Bibliotecas Recomendadas

### Backend (Python)

| Biblioteca          | Versão   | Uso                           |
|---------------------|----------|-------------------------------|
| fastapi             | ^0.115   | Framework web async           |
| pydantic-settings   | ^2.5     | Configurações tipadas         |
| sqlalchemy          | ^2.0     | ORM assíncrono                |
| asyncpg             | ^0.30    | Driver PostgreSQL async       |
| alembic             | ^1.14    | Migrations                    |
| celery[redis]       | ^5.4     | Tarefas em background         |
| anthropic           | ^0.40    | Claude API                    |
| google-api-python-client | ^2.0 | Google Calendar API          |
| google-auth-oauthlib | ^1.0   | OAuth Google                  |
| weasyprint          | ^63      | Geração de PDF                |
| jinja2              | ^3.1     | Templates de contratos        |
| python-jose[cryptography] | ^3.3 | JWT                        |
| passlib[bcrypt]     | ^1.7     | Hashing de senhas             |
| structlog           | ^24.4    | Logs estruturados             |
| boto3               | ^1.35    | Armazenamento S3              |

### Frontend (TypeScript/React)

| Biblioteca          | Uso                                      |
|---------------------|------------------------------------------|
| next                | Framework full-stack                     |
| @tanstack/react-query | Data fetching e cache                 |
| react-hook-form     | Formulários                              |
| zod                 | Validação de schemas                     |
| @dnd-kit/core       | Drag-and-drop para o Kanban              |
| shadcn/ui           | Componentes de UI                        |
| recharts            | Gráficos e KPIs                          |
| axios               | HTTP client                              |
| next-auth           | Autenticação (incluindo Google OAuth)    |
| date-fns            | Manipulação de datas                     |

---

## 10. Próximos Passos de Implementação

1. **Semana 1-2:** Configurar ambiente Docker, criar migrations, endpoints de auth e clientes
2. **Semana 3-4:** Pipeline (deals/Kanban), serviços, integração Google OAuth
3. **Semana 5-6:** Módulo de contratos + integração Claude API + geração de PDF
4. **Semana 7-8:** Google Calendar automático, frontend completo, testes E2E
5. **Semana 9-10:** Segurança, monitoramento, deploy em produção (VPS ou ECS)

---

*Documento gerado automaticamente — Lordam Architecture v1.0*

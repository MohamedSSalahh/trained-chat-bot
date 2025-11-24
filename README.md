## Botora

Botora is a **multi-tenant Django 5 project** using `django-tenants`. It is structured so that each tenant has its own schema in a single PostgreSQL database, while sharing the same Django project codebase.

### Tech stack

- **Backend**: Django 5
- **Multi-tenancy**: `django-tenants`
- **API / Auth**: Django REST Framework, `djangorestframework-simplejwt`
- **Database**: PostgreSQL (via `psycopg2-binary`)

## Project Diagrams

### Architecture Overview

```mermaid
graph TB
    subgraph "Client Layer"
        U1[User 1<br/>tenant1.example.com]
        U2[User 2<br/>tenant2.example.com]
        U3[Admin<br/>admin.example.com]
    end
    
    subgraph "Django Application"
        MW[TenantMiddleware]
        URL[URL Router]
        VIEW[Views]
        ADMIN[Django Admin]
    end
    
    subgraph "PostgreSQL Database"
        subgraph "Public Schema"
            CLIENT[Client Model]
            DOMAIN[Domain Model]
        end
        subgraph "Tenant Schema 1"
            T1_DATA[Tenant 1 Data]
        end
        subgraph "Tenant Schema 2"
            T2_DATA[Tenant 2 Data]
        end
    end
    
    U1 --> MW
    U2 --> MW
    U3 --> MW
    MW --> URL
    URL --> VIEW
    URL --> ADMIN
    VIEW --> CLIENT
    VIEW --> DOMAIN
    VIEW --> T1_DATA
    VIEW --> T2_DATA
    ADMIN --> CLIENT
    ADMIN --> DOMAIN
    CLIENT --> T1_DATA
    CLIENT --> T2_DATA
    DOMAIN --> CLIENT
```

### Project Structure

```mermaid
graph TD
    ROOT[botora/]
    
    ROOT --> BOTORA[botora/<br/>Django Project]
    ROOT --> TENANTS[tenants/<br/>Tenant App]
    ROOT --> SETTINGS[settings/<br/>Config]
    ROOT --> BACKUP[backup/<br/>Legacy Code]
    ROOT --> MANAGE[manage.py]
    ROOT --> REQ[requirements.txt]
    ROOT --> DB[db.sqlite3]
    
    BOTORA --> SETTINGS_FILE[settings.py]
    BOTORA --> URLS[urls.py]
    BOTORA --> WSGI[wsgi.py]
    BOTORA --> ASGI[asgi.py]
    
    TENANTS --> T_MODELS[models.py<br/>Client & Domain]
    TENANTS --> T_ADMIN[admin.py]
    TENANTS --> T_VIEWS[views.py]
    TENANTS --> T_MIGRATIONS[migrations/]
    
    SETTINGS --> ENV[env.py<br/>Database & Secrets]
    
    BACKUP --> BLOG[blog/<br/>Legacy App]
    BACKUP --> BLOG_SITE[blog_site/<br/>Legacy Project]
```

### Database Schema Structure

```mermaid
erDiagram
    CLIENT ||--o{ DOMAIN : "has"
    CLIENT ||--|| TENANT_SCHEMA_1 : "owns"
    CLIENT ||--|| TENANT_SCHEMA_2 : "owns"
    CLIENT ||--|| TENANT_SCHEMA_N : "owns"
    
    CLIENT {
        int id PK
        string schema_name
        string name
        date paid_until
        boolean on_trial
        date created_on
    }
    
    DOMAIN {
        int id PK
        int tenant_id FK
        string domain
        boolean is_primary
    }
    
    TENANT_SCHEMA_1 {
        string "Tenant-specific tables"
        string "Isolated data"
    }
    
    TENANT_SCHEMA_2 {
        string "Tenant-specific tables"
        string "Isolated data"
    }
    
    TENANT_SCHEMA_N {
        string "Tenant-specific tables"
        string "Isolated data"
    }
```

### Workflow Algorithm

```mermaid
flowchart TD
    Start([HTTP Request Received]) --> ExtractDomain[Extract Domain from Request<br/>Host Header or URL]
    ExtractDomain --> CheckPublic{Is Public<br/>Schema Request?<br/>admin.example.com}
    
    CheckPublic -->|Yes| PublicSchema[Use Public Schema<br/>No Tenant Context]
    CheckPublic -->|No| QueryDomain[Query Domain Model<br/>in Public Schema<br/>SELECT * FROM domains<br/>WHERE domain = ?]
    
    QueryDomain --> DomainFound{Domain<br/>Found?}
    DomainFound -->|No| Error404[Return 404<br/>Tenant Not Found]
    DomainFound -->|Yes| GetClient[Get Associated Client<br/>tenant = domain.tenant]
    
    GetClient --> CheckActive{Tenant Active?<br/>paid_until >= today<br/>OR on_trial = true}
    CheckActive -->|No| Error403[Return 403<br/>Tenant Inactive/Expired]
    CheckActive -->|Yes| SetSchema[Set PostgreSQL<br/>Schema Search Path<br/>SET search_path TO<br/>tenant.schema_name]
    
    SetSchema --> SetContext[Set Tenant Context<br/>request.tenant = tenant<br/>connection.set_tenant]
    
    SetContext --> ProcessRequest[Process Request<br/>URL Router → View]
    
    ProcessRequest --> QueryTenantData[Query Tenant Schema<br/>All DB queries automatically<br/>use tenant schema]
    
    QueryTenantData --> GenerateResponse[Generate Response<br/>View returns data]
    
    GenerateResponse --> ClearContext[Clear Tenant Context<br/>Reset schema search path]
    
    ClearContext --> ReturnResponse[Return HTTP Response<br/>to Client]
    
    PublicSchema --> ProcessPublicRequest[Process Public Request<br/>Admin/Management]
    ProcessPublicRequest --> ReturnResponse
    
    Error404 --> ReturnResponse
    Error403 --> ReturnResponse
    
    ReturnResponse --> End([End])
    
    style Start fill:#e1f5ff
    style End fill:#e1f5ff
    style Error404 fill:#ffebee
    style Error403 fill:#ffebee
    style SetSchema fill:#c8e6c9
    style QueryTenantData fill:#c8e6c9
```

### Request Flow Sequence

```mermaid
sequenceDiagram
    participant User
    participant TenantMiddleware
    participant URLRouter
    participant View
    participant Database
    
    User->>TenantMiddleware: HTTP Request<br/>(tenant1.example.com)
    TenantMiddleware->>Database: Query Domain Model<br/>(public schema)
    Database-->>TenantMiddleware: Return Tenant Info
    TenantMiddleware->>Database: Set Schema Search Path<br/>(tenant1 schema)
    TenantMiddleware->>URLRouter: Forward Request
    URLRouter->>View: Route to View
    View->>Database: Query Tenant Data<br/>(tenant1 schema)
    Database-->>View: Return Tenant Data
    View-->>URLRouter: Response
    URLRouter-->>TenantMiddleware: Response
    TenantMiddleware-->>User: HTTP Response
```

### Project structure (simplified)

- **`botora/`**: Main Django project (settings, URLs, WSGI/ASGI).
- **`tenants/`**: Tenant model definitions and logic for `django-tenants`.
- **`settings/env.py`**: Environment-specific configuration (database, secret key, CORS, etc.).
- **`backup/`**: Old/sample project (`blog_site` and `blog` app) kept for reference.

### Getting started

1. **Clone the repo**

```bash
git clone https://github.com/MohamedSSalahh/trained-chat-bot.git
cd trained-chat-bot
```

2. **Create and activate a virtual environment (recommended)**

```bash
python -m venv .venv
.venv\Scripts\activate  # Windows
# source .venv/bin/activate  # Linux / macOS
```

3. **Install dependencies**

```bash
pip install -r requirements.txt
```

4. **Configure environment settings**

Edit `settings/env.py` with your real PostgreSQL settings and secret key:

- **`NAME` / `USER` / `PASSWORD` / `HOST` / `PORT`** for your database.
- **`SECRET_KEY_SETTINGS`** for Django.
- Optionally adjust `ALLOWED_HOSTS_SETTINGS` and CORS settings.

5. **Apply migrations**

```bash
python manage.py migrate_schemas
```

> If you prefer to run standard migrations during early development, you can use:
>
> ```bash
> python manage.py migrate
> ```

6. **Create a superuser**

```bash
python manage.py createsuperuser
```

7. **Run the development server**

```bash
python manage.py runserver
```

By default the project runs with `DEBUG = True` and `ALLOWED_HOSTS` taken from `settings/env.py`.

### Tenants

The project is configured to use:

- **`TENANT_MODEL = "tenants.Client"`**
- **`TENANT_DOMAIN_MODEL = "tenants.Domain"`**

Once migrations are applied, you can create tenants via the Django admin or with custom management commands (not yet included in this repository). Each tenant will use its own schema managed by `django-tenants`.

### Linting

The project includes **flake8** in `requirements.txt`. You can run it (once configured) with:

```bash
flake8
```

### Legacy / backup project

The `backup/` folder contains an older demo blog project (`blog_site` and `blog` app). It is not used by the main Botora project, but kept as reference or example code.

### Notes

- This repository currently uses a simple settings structure with `settings/env.py` checked in. In a real deployment you should move secrets (database password, secret key, etc.) into environment variables or a secrets manager.
- PostgreSQL is required because `django-tenants` relies on PostgreSQL schemas.



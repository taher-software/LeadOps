# LeadOps

## Project goal

An autonomous, AI-assisted commercial prospecting system that helps **Azibodin**:

1. **Identify** the right companies (targeting, ICP fit)
2. **Understand** their needs (enrichment, business intelligence)
3. **Initiate** relevant conversations (personalized outreach)
4. **Convert** qualified prospects into business opportunities

The platform is built from **specialized agents**, **API-driven integrations**, and **orchestrated workflows**. New prospecting capabilities and business intelligence features are added one at a time, without reworking what already exists.

## Scope

- **Backend only.** No frontend lives in this repository.
- All code lives under `backend/`.

## Tech stack

| Concern              | Technology            |
|----------------------|-----------------------|
| HTTP API             | FastAPI               |
| Validation / schemas | Pydantic (v2)         |
| Cloud platform       | Google Cloud (GCP)    |
| Async messaging      | Cloud Pub/Sub         |
| Database             | Firestore             |
| Scheduled triggers   | Cloud Scheduler       |

Do not add other infrastructure (SQL databases, Redis, Celery, other queues) without discussing it first.

## Repository layout

```
backend/
├── scripts/                 one-off and operational scripts (seeding, local setup, maintenance)
├── src/
│   └── app/
│       ├── async_jobs/      async job handlers triggered by Pub/Sub messages
│       ├── core/            settings/config, logging, shared dependencies, errors
│       ├── gcp/             Google Cloud clients (Firestore, Pub/Sub, ...)
│       ├── globals/
│       │   └── enum/        shared enums, including JobType
│       └── routers/         one folder per API resource
└── tests/
```

### Where things go

- **`routers/<resource>/`**: one folder per resource (for example `companies/`, `prospects/`, `campaigns/`). Each folder holds that resource's FastAPI router, its Pydantic request/response schemas, and its service logic. Keep route handlers thin and put business logic in the service.
- **`async_jobs/`**: one handler per `JobType`. Handlers take a validated Pydantic payload and must be **idempotent**, because Pub/Sub delivers at least once.
- **`gcp/`**: the only place that imports the Google Cloud SDKs. The rest of the app goes through these wrappers so they can be mocked in tests.
- **`core/`**: app-wide concerns. Configuration comes from environment variables through a Pydantic `BaseSettings` class. Never hard-code project IDs, topic names, or credentials.
- **`globals/enum/`**: enums shared across modules. `JobType` lists every async job the system can run.

## Architectural conventions

### Async job flow

```
Cloud Scheduler ──┐
                  ├──▶ Pub/Sub topic ──▶ push subscription ──▶ FastAPI endpoint ──▶ async_jobs/<handler>
API request ──────┘     (message has a JobType + payload)
```

- Every async message carries a `JobType` and a Pydantic-validated payload.
- Dispatch is driven by `JobType`. To add a job, add an enum value, write the handler, and register it. Do not edit existing handlers to add new behavior.
- Handlers must tolerate retries and duplicate deliveries.
- Cloud Scheduler never calls business logic directly. It publishes to Pub/Sub, or calls an endpoint that publishes.

### Agents

- Each agent (for example company discovery, enrichment, needs analysis, outreach, qualification) has one narrow responsibility.
- Agents talk to each other through jobs and Firestore state, not through direct calls, so they can be developed, retried, and scaled independently.
- External APIs (data providers, LLMs, email, CRM) sit behind an integration interface so providers can be swapped.

### Data (Firestore)

- Define collection names in one place, not as string literals spread across the code.
- Every stored document maps to a Pydantic model. Validate on write and on read.
- Include `created_at` and `updated_at` timestamps (UTC) on documents.

## Coding standards

- Python 3.12+, full type hints everywhere.
- Pydantic v2 for every input, output, message payload, and stored document.
- Use `async` endpoints and handlers where the underlying client supports it.
- Use structured logging. Never log secrets or full personal data.
- Keep modules small and focused, in line with the progressive, plug-in style of feature growth.

## Testing

- Tests live in `backend/tests/` and mirror the `src/app/` structure.
- Mock GCP through the `gcp/` wrappers, or use the Firestore and Pub/Sub emulators. Tests never hit real GCP resources.
- Every async job handler needs a test that covers a duplicate (redelivered) message.

## Working agreements

- The project is in its **scaffolding phase**. Only the directory structure exists so far, and the `.gitkeep` files only hold empty folders in place.
- Don't write implementation code until it's requested.
- Ask before adding new dependencies or GCP services.

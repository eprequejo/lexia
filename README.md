# LexPulse — Database Schema (MVP v1)

PostgreSQL (Cloud SQL) + Cloud Storage for documents.

## Overview

Everything hangs from `firm_id` — multitenancy by law firm.
One firm never sees another firm's data.
Documents (PDFs, photos) live in Cloud Storage, the DB only stores references.

---

## Tables

### firms
The law firms that buy the app.

| Column     | Type         | Notes                              |
|------------|--------------|-------------------------------------|
| id         | uuid PK      | auto-generated                     |
| name       | varchar(255) | firm name                          |
| cif        | varchar(20)  | tax ID                             |
| email      | varchar(255) |                                     |
| phone      | varchar(20)  |                                     |
| address    | text         |                                     |
| plan       | enum         | free / pro / enterprise            |
| created_at | timestamptz  |                                     |

---

### lawyers
Lawyers belonging to a firm. Also used for auth (they log in to manage cases).

| Column     | Type         | Notes                              |
|------------|--------------|-------------------------------------|
| id         | uuid PK      |                                     |
| firm_id    | uuid FK      | → firms.id                         |
| name       | varchar(255) |                                     |
| email      | varchar(255) | unique, used for login             |
| phone      | varchar(20)  |                                     |
| bar_number | varchar(50)  | num colegiado (e.g. ICAM 12.XXX)   |
| specialty  | varchar(100) | laboral, bancario, familia...      |
| role       | enum         | admin / lawyer                     |
| created_at | timestamptz  |                                     |

---

### clients
Clients of the firm. They also log in to see their dashboard.

| Column     | Type         | Notes                              |
|------------|--------------|-------------------------------------|
| id         | uuid PK      |                                     |
| firm_id    | uuid FK      | → firms.id                         |
| name       | varchar(255) |                                     |
| email      | varchar(255) | unique, used for login             |
| phone      | varchar(20)  |                                     |
| dni        | varchar(20)  | encrypted at rest                  |
| address    | text         |                                     |
| created_at | timestamptz  |                                     |

---

### cases
The core table. A client has a case with a lawyer.

| Column      | Type         | Notes                              |
|-------------|--------------|-------------------------------------|
| id          | uuid PK      |                                     |
| firm_id     | uuid FK      | → firms.id                         |
| client_id   | uuid FK      | → clients.id                       |
| lawyer_id   | uuid FK      | → lawyers.id (nullable at start)   |
| title       | varchar(255) |                                     |
| description | text         | client's own words                 |
| case_type   | enum         | laboral / bancario / inmobiliario / familia / mercantil / contratos / extranjeria / otro |
| status      | enum         | see status flow below              |
| price       | decimal      | closed price in EUR (nullable until quoted) |
| created_at  | timestamptz  |                                     |
| updated_at  | timestamptz  |                                     |
| resolved_at | timestamptz  | nullable                           |

#### Case statuses

```
new → ai_analysis → reviewed → in_progress → pending_docs → pending_signature → resolved → closed
                                    ↓
                                cancelled
```

- **new**: client just submitted the case
- **ai_analysis**: AI agent is analyzing (or has analyzed, pending lawyer review)
- **reviewed**: lawyer reviewed the AI analysis, case accepted
- **in_progress**: lawyer is working on it
- **pending_docs**: waiting for client to upload documents
- **pending_signature**: ready to sign (digital signature integration)
- **resolved**: case resolved, all done
- **closed**: archived
- **cancelled**: client or lawyer cancelled

---

### case_activity
Audit log. Every change to a case is recorded here.

| Column          | Type         | Notes                              |
|-----------------|--------------|-------------------------------------|
| id              | uuid PK      |                                     |
| case_id         | uuid FK      | → cases.id                         |
| actor_type      | enum         | lawyer / client / ai / system      |
| actor_id        | uuid         | FK to lawyers.id or clients.id     |
| action          | enum         | status_change / note / assignment / document_upload / message |
| detail          | text         | human-readable description         |
| previous_status | varchar(30)  | nullable (only for status_change)  |
| new_status      | varchar(30)  | nullable (only for status_change)  |
| created_at      | timestamptz  |                                     |

---

### documents
Files associated with a case. Actual files live in Cloud Storage.

| Column        | Type         | Notes                              |
|---------------|--------------|-------------------------------------|
| id            | uuid PK      |                                     |
| case_id       | uuid FK      | → cases.id                         |
| uploaded_by   | uuid         | FK to lawyers.id or clients.id     |
| uploader_type | enum         | lawyer / client                    |
| filename      | varchar(255) | original filename                  |
| storage_path  | text         | Cloud Storage URL / path           |
| file_type     | varchar(50)  | pdf, jpg, png, docx...             |
| file_size     | integer      | bytes                              |
| created_at    | timestamptz  |                                     |

---

### messages
Chat between client, lawyer, and AI within a case.

| Column      | Type         | Notes                              |
|-------------|--------------|-------------------------------------|
| id          | uuid PK      |                                     |
| case_id     | uuid FK      | → cases.id                         |
| sender_type | enum         | client / lawyer / ai               |
| sender_id   | uuid         | FK to lawyers.id or clients.id (null for ai) |
| content     | text         |                                     |
| created_at  | timestamptz  |                                     |

---

### ai_analyses
Structured AI analysis per case. Separate from chat messages.
The lawyer reviews this before the case moves forward.

| Column                 | Type         | Notes                              |
|------------------------|--------------|-------------------------------------|
| id                     | uuid PK      |                                     |
| case_id                | uuid FK      | → cases.id                         |
| summary                | text         | AI-generated case summary          |
| relevant_laws          | text         | laws/articles identified           |
| relevant_jurisprudence | text         | similar cases found                |
| suggested_strategy     | text         | AI's suggested approach            |
| disclaimer_accepted    | boolean      | client accepted the disclaimer     |
| reviewed_by_lawyer     | uuid FK      | → lawyers.id (nullable)           |
| reviewed_at            | timestamptz  | nullable                           |
| created_at             | timestamptz  |                                     |

---

## Key relationships

```
firms
  └── lawyers (1:N)
  └── clients (1:N)
  └── cases (1:N)
        ├── client_id → clients
        ├── lawyer_id → lawyers
        ├── case_activity (1:N)
        ├── documents (1:N)
        ├── messages (1:N)
        └── ai_analyses (1:N)
```

## Security notes

- DNI field encrypted at rest
- All tables filtered by firm_id (row-level security)
- Documents in Cloud Storage with signed URLs (time-limited access)
- Passwords hashed with bcrypt (auth table TBD — may use external auth provider)
- All timestamps in UTC# lexia

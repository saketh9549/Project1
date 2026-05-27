# ResumeAI — System Architecture & Stabilization Walkthrough

## System Architecture

```mermaid
graph TB
    subgraph Frontend["Frontend (React + Vite)"]
        UI["React SPA"]
        Router["React Router v6"]
        API_Client["api.js HTTP Client"]
        WS_Client["WebSocket Client"]
        Theme["ThemeContext"]
    end

    subgraph Backend["Backend (FastAPI)"]
        direction TB
        Main["main.py FastAPI App"]

        subgraph Routes["API Routes"]
            AuthR["auth.py"]
            ResumeR["resume.py"]
            AIR["ai.py"]
            JobsR["jobs.py"]
            InterviewR["interviews.py"]
            RewriterR["rewriter.py"]
            LiveJobsR["live_jobs.py"]
            RecruiterR["recruiter.py"]
            AnalyticsR["analytics.py"]
        end

        subgraph Middleware["Middleware"]
            AuthGuard["auth_guard.py"]
            RoleGuard["role_guard.py"]
            RateLimit["rate_limit.py"]
        end

        subgraph Services["Core Services"]
            ParseEngine["parsing_engine.py"]
            ScoreEngine["scoring_engine.py"]
            ExtractSvc["extraction_service.py"]
            AuthSvc["auth_service.py"]
        end

        subgraph AI["AI Layer"]
            GeminiSvc["gemini_service.py"]
            ResponseParser["response_parser.py"]
            VisionAnalysis["vision_analysis.py"]
            MultimodalSvc["multimodal_service.py"]
        end

        subgraph Storage["Storage Layer"]
            GridFS["gridfs_service.py"]
            UploadSvc["upload_service.py"]
            DownloadSvc["download_service.py"]
            PreviewSvc["preview_service.py"]
            MetadataSvc["file_metadata.py"]
        end

        subgraph Workers["Background Workers"]
            Celery["celery_worker.py"]
            Tasks["tasks.py"]
            RedisConfig["redis_config.py"]
        end
    end

    subgraph External["External Services"]
        MongoDB[("MongoDB")]
        Redis[("Redis")]
        GeminiAPI["Google Gemini API"]
        RemotiveAPI["Remotive Jobs API"]
    end

    UI --> Router
    Router --> API_Client
    API_Client -->|HTTP/REST| Main
    WS_Client -->|WebSocket| ResumeR

    Main --> Routes
    Routes --> Middleware
    Routes --> Services
    Routes --> AI
    Routes --> Storage

    Services --> MongoDB
    Storage --> MongoDB
    Workers --> MongoDB
    Workers --> Redis
    AI --> GeminiAPI
    LiveJobsR --> RemotiveAPI
    RateLimit --> Redis
```

---

## Data Flow

```mermaid
sequenceDiagram
    participant U as User Browser
    participant F as Frontend (React)
    participant B as Backend (FastAPI)
    participant DB as MongoDB
    participant GFS as GridFS
    participant AI as Gemini AI
    participant R as Redis/Celery

    Note over U,R: Resume Upload & Analysis Flow

    U->>F: Drop resume file
    F->>B: POST /api/resume/upload
    B->>GFS: Store binary file
    GFS-->>B: gridfs_file_id
    B->>DB: Create resume metadata (status: uploaded)
    B-->>F: { id, filename, status }

    F->>B: POST /api/resume/analyze/{id}
    alt Redis Available
        B->>R: Queue run_async_analysis_task
        R->>B: Task queued
        B-->>F: { status: queued }
        F->>B: WebSocket /ws/resume-status/{id}
        R-->>DB: Update progress stages
        Note over R,AI: Background: Extract → Parse → Score → Embed → AI Review → Vision
        R->>AI: Generate feedback
        AI-->>R: AI analysis result
        R-->>DB: Update status: completed
        B-->>F: WS: { status: completed, progress: 100 }
    else Redis Unavailable (Fallback)
        B->>B: Run analysis synchronously
        B->>AI: Generate feedback
        AI-->>B: AI analysis result
        B->>DB: Update all fields
        B-->>F: { status: completed }
    end
```

---

## API Endpoint Map

### Authentication (`/api/auth`)
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/signup` | ✗ | Create new user account |
| POST | `/login` | ✗ | Authenticate and get JWT tokens |
| PUT | `/change-password` | ✓ | Change password (requires current) |
| POST | `/refresh` | ✗ | Rotate refresh token |
| POST | `/forgot-password` | ✗ | Request password reset OTP |
| POST | `/reset-password` | ✗ | Reset password with OTP |
| POST | `/request-email-verification` | ✗ | Request email verification OTP |
| POST | `/verify-email` | ✗ | Verify email with OTP |

### Resume (`/api/resume`)
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/upload` | ✓ | Upload resume (GridFS + parse + score) |
| POST | `/upload-only` | ✓ | Upload without processing |
| POST | `/analyze/{id}` | ✓ | Trigger async analysis |
| GET | `/recent` | ✓ | List user's resumes |
| GET | `/detail/{id}` | ✓ | Get full resume data |
| GET | `/preview/{id}` | ✓ | Get PDF preview URL |
| GET | `/download/{id}` | ✓ | Download original file |
| DELETE | `/delete/{id}` | ✓ | Delete resume |
| WS | `/ws/resume-status/{id}` | ✗ | Real-time analysis progress |

### AI (`/api/ai`)
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/feedback` | ✓ | Generate AI recruiter feedback |
| GET | `/feedback/{id}` | ✓ | Get stored feedback |

### Jobs (`/api/jobs`)
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/match` | ✓ | Run AI job matching |
| GET | `/matches/{id}` | ✓ | Get stored matches |
| POST | `/match-custom-jd` | ✓ | Match against custom JD |
| POST | `/roadmap` | ✓ | Generate career roadmap |

### Interview Prep (`/api/interviews`)
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/generate` | ✓ | Generate interview questions |
| POST | `/evaluate` | ✓ | Evaluate mock interview answers |

### Resume Rewriter (`/api/rewriter`)
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/rewrite` | ✓ | AI-rewrite resume section |

### Live Jobs (`/api/live-jobs`)
| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/recommendations` | ✓ | Skill-matched remote jobs |

### Recruiter (`/api/recruiter`)
| Method | Endpoint | Auth | Role |
|--------|----------|------|------|
| GET | `/candidates` | ✓ | recruiter/admin |
| POST | `/shortlist/add` | ✓ | recruiter/admin |
| GET | `/shortlist` | ✓ | recruiter/admin |
| POST | `/jobs` | ✓ | recruiter/admin |
| GET | `/jobs` | ✓ | recruiter/admin |
| GET | `/analytics` | ✓ | recruiter/admin |

---

## Authentication Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant B as Backend
    participant DB as MongoDB

    Note over C,DB: Login Flow
    C->>B: POST /api/auth/login {email, password}
    B->>DB: Find user by email
    DB-->>B: User document
    B->>B: verify_password(input, stored_hash)
    B->>B: create_access_token(email, role)
    B->>B: create_refresh_token(email)
    B->>DB: Store refresh_token
    B-->>C: { access_token, refresh_token, user }

    Note over C,DB: Authenticated Request
    C->>B: GET /api/resume/recent [Authorization: Bearer <token>]
    B->>B: auth_guard: decode JWT, validate exp
    B->>DB: Query user resumes
    DB-->>B: Resume documents
    B-->>C: Resume list

    Note over C,DB: Token Refresh
    C->>B: POST /api/auth/refresh { refresh_token }
    B->>B: Decode refresh JWT
    B->>DB: Find stored refresh token
    B->>DB: Delete old token (rotation)
    B->>B: Generate new access + refresh tokens
    B->>DB: Store new refresh token
    B-->>C: { access_token, refresh_token }
```

---

## Stabilization Changes Made

### Backend Fixes Applied

| # | File | Fix | Category |
|---|------|-----|----------|
| 1 | [mongodb.py](file:///c:/Users/Lenovo/OneDrive%20-%20Shiv%20Nadar%20Institution%20of%20Eminence/Documents/saketh/shiv%20nadar%20university/Resume%20Analyser/backend/database/mongodb.py) | Added architecture documentation about sync pymongo usage | Architecture |
| 2 | [main.py](file:///c:/Users/Lenovo/OneDrive%20-%20Shiv%20Nadar%20Institution%20of%20Eminence/Documents/saketh/shiv%20nadar%20university/Resume%20Analyser/backend/main.py) | Suppressed google.generativeai FutureWarning | Deprecation |
| 3 | [vision_analysis.py](file:///c:/Users/Lenovo/OneDrive%20-%20Shiv%20Nadar%20Institution%20of%20Eminence/Documents/saketh/shiv%20nadar%20university/Resume%20Analyser/backend/multimodal/vision_analysis.py) | Suppressed FutureWarning at import-time | Deprecation |
| 4 | [interviews.py](file:///c:/Users/Lenovo/OneDrive%20-%20Shiv%20Nadar%20Institution%20of%20Eminence/Documents/saketh/shiv%20nadar%20university/Resume%20Analyser/backend/routes/interviews.py) | `get_event_loop()` → `get_running_loop()` (×2) | Deprecation |
| 5 | [rewriter.py](file:///c:/Users/Lenovo/OneDrive%20-%20Shiv%20Nadar%20Institution%20of%20Eminence/Documents/saketh/shiv%20nadar%20university/Resume%20Analyser/backend/routes/rewriter.py) | `get_event_loop()` → `get_running_loop()` | Deprecation |
| 6 | [live_jobs.py](file:///c:/Users/Lenovo/OneDrive%20-%20Shiv%20Nadar%20Institution%20of%20Eminence/Documents/saketh/shiv%20nadar%20university/Resume%20Analyser/backend/routes/live_jobs.py) | Fixed bare `except:` → `except Exception:` | Code Quality |
| 7 | [live_jobs.py](file:///c:/Users/Lenovo/OneDrive%20-%20Shiv%20Nadar%20Institution%20of%20Eminence/Documents/saketh/shiv%20nadar%20university/Resume%20Analyser/backend/routes/live_jobs.py) | Fixed sort field `date` → `upload_date` | Data Bug |
| 8 | [auth.py](file:///c:/Users/Lenovo/OneDrive%20-%20Shiv%20Nadar%20Institution%20of%20Eminence/Documents/saketh/shiv%20nadar%20university/Resume%20Analyser/backend/routes/auth.py) | `print()` → `logger.info()` for OTP codes | Security |
| 9 | [auth.py](file:///c:/Users/Lenovo/OneDrive%20-%20Shiv%20Nadar%20Institution%20of%20Eminence/Documents/saketh/shiv%20nadar%20university/Resume%20Analyser/backend/routes/auth.py) | Added 10-minute OTP expiry validation | Security |

### Frontend Fixes Applied

| # | File | Fix | Category |
|---|------|-----|----------|
| 10 | [api.js](file:///c:/Users/Lenovo/OneDrive%20-%20Shiv%20Nadar%20Institution%20of%20Eminence/Documents/saketh/shiv%20nadar%20university/Resume%20Analyser/frontend/src/services/api.js) | Downgraded 404 error logs to `console.info` | DX |
| 11 | [App.jsx](file:///c:/Users/Lenovo/OneDrive%20-%20Shiv%20Nadar%20Institution%20of%20Eminence/Documents/saketh/shiv%20nadar%20university/Resume%20Analyser/frontend/src/App.jsx) | Added `React.lazy()` code-splitting for 8 pages | Performance |
| 12 | Pages (`ResumeRewriter.jsx`, `JobMatching.jsx`, `InterviewPrep.jsx`, `LiveJobs.jsx`, `Profile.jsx`) | Propagated `getRecentUploads` error and showed `useToast` error banners | Error Handling |

### Test Cleanup

| # | File | Fix | Category |
|---|------|-----|----------|
| 13 | [test_enterprise.py](file:///c:/Users/Lenovo/OneDrive%20-%20Shiv%20Nadar%20Institution%20of%20Eminence/Documents/saketh/shiv%20nadar%20university/Resume%20Analyser/tests/test_enterprise.py) | Removed duplicate Mongo cleanup boilerplate | Maintenance |
| 14 | [test_multimodal_rag.py](file:///c:/Users/Lenovo/OneDrive%20-%20Shiv%20Nadar%20Institution%20of%20Eminence/Documents/saketh/shiv%20nadar%20university/Resume%20Analyser/tests/test_multimodal_rag.py) | Removed duplicate Mongo cleanup boilerplate | Maintenance |

---

## Verification Results

| Check | Result |
|-------|--------|
| Unit Tests (34/34) | ✅ All pass |
| Backend Import Chain | ✅ Clean (no warnings) |
| Frontend Production Build | ✅ Succeeds |
| Bundle Size Reduction | ✅ 999KB → 807KB core (-19%), pages split into separate chunks |

---

## Technical Debt (Documented)

| Item | Priority | Effort |
|------|----------|--------|
| Migrate all sync PyMongo to async Motor | High | Large (~15 files) |
| Migrate `google.generativeai` → `google.genai` | Medium | Medium (~5 files, API surface differs) |
| Add end-to-end integration tests | Medium | Medium |
| Add API rate limit tests | Low | Small |

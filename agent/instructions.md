Version	1.1 (draft)
Date	22 October 2026
Status	Prototype
Based on	PRD_Daily_Health_Companion.md (v1.0)
Related docs	README.md
Daily Health Companion is a self-management and organization aid. It does not diagnose conditions, interpret readings as a diagnosis, prescribe, or change treatment. This design has not had a HIPAA or other regulatory review and is not production-ready for real patient data until a qualified review is done.

1. Purpose and scope
This document describes how the system in the PRD is built: components, data flow, data model, security, directory structure, and deployment.

Deployment targets

Part	Platform
Frontend (React, TypeScript, Vite)	Vercel
Backend API (Node.js, Express, TypeScript)	Render Web Service
Database (PostgreSQL)	Render PostgreSQL
Authentication	Supabase Auth (Google and email), or the project's existing secure provider
AI	Optional provider, called only from the backend
2. PRD traceability
How each PRD area maps to the architecture.

PRD area	Where it lives
Daily health dashboard	Frontend pages/Dashboard, data from /logs and /reminders/due
Quick health logging (6 categories)	Frontend features/logs, backend routes/logs, health_logs table
Personal reminders (create, edit, snooze, complete, disable, delete)	Frontend features/reminders, backend routes/reminders, reminders and reminder_occurrences tables
Reminder sound and visual alerts	Frontend features/reminders/alerts (browser audio and notifications)
Reuse yesterday's schedule	Backend services/scheduleCopy, preview and confirm endpoints
Accessible personalization (themes, text size, simple language, interests)	Frontend features/preferences and theme/, backend /me/preferences, preferences table
Voice-friendly interaction	Frontend features/voice (browser speech APIs, with on-screen alternatives)
AI-assisted summaries	Backend services/ai and routes/assistant, frontend features/assistant
Confirmed assistant actions	Draft proposals returned by the API, saved only through normal validated endpoints after user approval
Account and cross-device access	Supabase Auth, backend auth middleware, PostgreSQL
Demo mode	Frontend data/localStore (browser storage) and demo assistant
Privacy controls (view and delete own records)	Frontend features/privacy, backend delete endpoints
Non-functional: accessibility, responsive, reliability, security, data integrity, maintainability, deployment	Sections 5 to 13
Later considerations in the PRD (caregiver access, data export, full account deletion workflow, more languages, device integrations) are listed in section 16 and are not part of the MVP design.

3. Design principles
Separate services. Vercel hosts the frontend. Render hosts the API and database. They share no secrets.
The server is the authority. The backend verifies identity and enforces ownership on every request.
Secrets stay on the server. No database URL, AI key, or signing secret in the browser, Git, or any VITE_ variable.
User ID comes from the verified token only.
Minimal health data. Collect only what is needed, send the least possible to the AI provider, and keep it out of logs.
Demo mode always works without an account, backend, or AI key.
Every AI-proposed change needs user confirmation.
Time is stored in UTC. Each reminder keeps its own IANA timezone.
Accessibility is a design requirement. WCAG 2.2 AA is the target, subject to testing.
4. System context

Loading
Component	Responsibility
Frontend (Vercel)	UI, themes, text size, simple language, voice, demo mode, reminder alerts, sign-in flow, API calls
Backend API (Render)	Token verification, validation, business rules, reminders, schedule copy, AI adapter, rate limiting
PostgreSQL (Render)	Profiles, preferences, health logs, reminders, reminder occurrences
Supabase Auth	Google and email sign-in, issues access tokens
AI provider	Summaries and suggestions. Called only by the backend
5. Technology choices
Layer	Choice	Notes
Frontend	React, TypeScript, Vite	Existing app. Changed as little as possible
Frontend hosting	Vercel	SPA rewrites via vercel.json
Backend	Node.js 20+, Express, TypeScript	Extend an existing backend only if the audit shows it is sound
Validation	Zod (or similar)	Every body, query, and param
Database	PostgreSQL 14+	Versioned migrations
DB access	Parameterized queries (query builder or ORM)	No string-built SQL
Auth	Supabase Auth	Backend verifies the access token
Security middleware	Helmet, CORS allow-list, rate limiter	
Voice	Browser Web Speech APIs	Where supported, always with an on-screen alternative
Testing	Vitest or Jest, Supertest	Auth and AI provider mocked
Backend hosting	Render Web Service and Render PostgreSQL	HTTPS by default
Exact libraries may change after the Phase 1 audit of the existing project.

6. Frontend architecture (Vercel)
6.1 Modes
Mode	When	Data location
Demo	Signed out, or no backend configured	Browser storage, clearly labeled sample data
Account	Signed in	Backend and PostgreSQL
A data-access layer hides the difference. UI components call one interface, and the layer routes to browser storage or the API. This keeps demo mode working and the UI unchanged. Preferences persist to the account when signed in, or locally in demo mode.

6.2 Responsibilities
Sign-in and sign-out through Supabase, and attaching the access token to API calls
Dashboard, quick-add logging, reminder management, summaries
Personalization: themes (Calm Blue, Nature Green, Warm Sunset, High Contrast), text sizes (standard, large, extra-large), simple-language mode, optional interests for friendly non-medical greetings
Accessibility: keyboard navigation, visible focus, semantic labels, readable contrast, responsive layouts, reduced-motion support
Voice-friendly input and spoken responses where the browser supports them
Reminder alerts: visible alert always, optional sound with test control and volume
Showing AI drafts for confirmation
Clear states for loading, offline, expired sign-in, and unavailable AI
6.3 Reminder alerts

Loading
Sound depends on browser permissions and the app being available. If the browser or device has closed the app, no sound plays. The design does not promise guaranteed alarms, and a visible alert is always shown with the sound.

6.4 Vercel configuration
Framework preset: Vite. Build command npm run build, output directory dist.
vercel.json rewrites all routes to index.html so client-side routing works on refresh.
VITE_* variables are read at build time. Changing them requires a redeploy.
Public values only: VITE_API_BASE_URL (the Render API URL), VITE_SUPABASE_URL, VITE_SUPABASE_ANON_KEY.
Preview deployments get their own URLs. See section 11.4 for CORS handling.
7. Backend architecture (Render)
7.1 Request pipeline
Request
  → Security middleware   (Helmet, CORS allow-list, body size limit, rate limit)
  → Auth middleware       (verify token, attach userId)
  → Validation            (schema check of body, query, params)
  → Route handler
  → Service layer         (rules: logs, reminders, schedule copy, AI)
  → Repository / DB layer (parameterized queries, always filtered by userId)
  → Response or centralized error handler
7.2 Cross-cutting behavior
Concern	Approach
Authentication	Verify the token on every protected route. Missing, invalid, or expired tokens return 401
Authorization	Every query includes user_id = <verified user>. Other users' records return 404
Validation	Reject malformed input with a consistent 400 response
Errors	One JSON error format. No stack traces, secrets, or internal details
Rate limiting	General limit, plus a stricter limit on the AI endpoint
Logging	Operational events only. No tokens, keys, passwords, prompts, or unnecessary health details
Config	Environment variables validated at startup. The app refuses to start if required ones are missing
Shutdown	Graceful shutdown closes the HTTP server and the DB pool
7.3 API surface
Method	Path	Purpose
GET	/health	Render health check (public, no sensitive data)
GET / PATCH	/api/v1/me	Profile
GET / PATCH	/api/v1/me/preferences	Theme, text size, simple language, interests, sound, volume
GET / POST	/api/v1/logs	List (filter by type and date, paginated) or create
GET / PATCH / DELETE	/api/v1/logs/:id	View, edit, delete a log
GET / POST	/api/v1/reminders	List or create reminders
GET / PATCH / DELETE	/api/v1/reminders/:id	View, edit, disable, delete a reminder
GET	/api/v1/reminders/due	Reminders due now
POST	/api/v1/reminders/:id/snooze	Snooze
POST	/api/v1/reminders/:id/complete	Mark done
GET	/api/v1/reminders/copy-yesterday/preview	Preview the copy
POST	/api/v1/reminders/copy-yesterday	Confirm and copy
GET	/api/v1/summaries	Factual summary (daily, weekly, or appointment) built from the user's records
POST	/api/v1/assistant	Ask the assistant (draft suggestions only)
POST	/api/v1/import/local	One-time import of local data, only after explicit user approval
8. Authentication and authorization

Loading
Rules:

The backend never trusts a user ID sent by the client.
Health information is never placed in Google profile fields.
Local browser data is never migrated to an account without explicit user approval.
Supabase redirect URLs must include the Vercel production URL and the local dev URL.
If the project already has a secure auth system, use it and skip Supabase.
Because the frontend and backend are on different domains, auth uses the Bearer token header, not cookies. This avoids cross-site cookie problems.
9. Data architecture
9.1 Entity relationships

Loading
9.2 Rules
category is one of medication, meal, sleep, activity, blood_pressure, glucose.
data is validated per category before saving. Values are stored as the user entered them, with their unit. They are never scored or diagnosed.
All timestamps are timestamptz in UTC.
Every user-data table has user_id with a foreign key and an index.
Deleting a profile cascades to all of that user's data.
A unique constraint on (reminder_id, occurrence_date) prevents duplicate occurrences.
Reminders are organizational aids only. They never suggest medication timing or dosage.
9.3 Migrations
Versioned, forward-only migrations run with npm run migrate. They are idempotent and safe on an empty database. Migrations run as a separate deploy step, not automatically on every server start.

10. Reminder, schedule, and AI design
10.1 Due-time calculation
Store time of day, start date, repeat rule, and IANA timezone per reminder.
Convert to a UTC due_at for each occurrence using the timezone database, so daylight saving shifts are correct.
Create or find the occurrence for the date when it becomes due.
10.2 No duplicate alerts
Each occurrence has last_alerted_at.
The due endpoint returns an occurrence only if it is due, not completed, not snoozed, and not yet alerted.
Selecting and marking last_alerted_at happen in one transaction, so refreshes, re-renders, and repeated checks cannot create a second alert.
Snooze sets snoozed_until and clears last_alerted_at, so the occurrence alerts once more at the new time.
10.3 Copy yesterday's schedule

Loading
Requires an explicit confirm: true.
Matching and unique constraints skip reminders that already exist today, so repeating the action is safe.
Completion status and health logs are never copied.
10.4 AI assistant

Loading
Rule (from the PRD)	How it is enforced
Demo mode without a key, AI mode when configured	Backend checks for the key. Frontend shows which mode is active
Server-side only	Key is a Render environment variable, never sent to the browser
Minimum data per request	Backend picks only the records relevant to the request
Factual summaries only	Daily, weekly, and appointment summaries use the user's own entries
No diagnosis, prescribing, or treatment changes	Safety system prompt, short disclaimer, guardrail tests
Urgent concerns	Response directs the user to local emergency services or a healthcare professional
No unconfirmed saves	Suggestions are drafts, saved only through the normal endpoints after user approval
Safe error states	Clear retry and unavailable messages with no secrets or stack traces
Privacy	Prompts and responses are not logged. UI explains what is shared before AI use
Provider swap	Provider sits behind an adapter interface
11. Security architecture
11.1 Measures
Area	Measure
Transport	HTTPS on Vercel and Render
Headers	Helmet defaults on the API
CORS	Allow-list read from environment variables
Input	Schema validation and a request size limit
SQL	Parameterized queries only
Access control	Token verification and user-scoped queries on every operation
Rate limiting	General limit and a stricter AI limit
Secrets	Server environment variables only. .env is git-ignored. .env.example has placeholders
Errors	Generic client messages. Details only in server logs, without sensitive data
Logging	No tokens, keys, passwords, prompts, or unnecessary health details
Dependencies	Pinned versions, npm audit, regular updates
Frontend bundle	Scanned for leaked secrets before deploy
11.2 Threats
Threat	Mitigation
A user reads or edits another user's data	User-scoped queries, 404 on foreign records, ownership tests
Stolen or forged token	Signature, expiry, and issuer verification
Secret leak through the frontend	No secrets in VITE_ variables, bundle scan
Prompt injection through user notes	Treat user text as data, limit AI to drafts, require confirmation, validate any suggested record
Abuse of the AI endpoint	Strict rate and size limits
Cross-site requests	CORS allow-list and Bearer-token auth
11.3 CORS with Vercel
FRONTEND_URL is the exact Vercel production URL, for example https://your-app.vercel.app or your custom domain.
LOCAL_FRONTEND_URL is http://localhost:5173.
Only these origins are allowed.
11.4 Vercel preview deployments
Each Vercel preview has a different URL, so it will be blocked by CORS. Options:

Default: test against production or local only.
Optional: add a specific preview URL to an extra allow-list variable (for example EXTRA_ALLOWED_ORIGINS) when needed. Avoid a broad wildcard such as *.vercel.app, because anyone can host a site on that domain.
12. Deployment architecture

Loading
Service	Platform	Key settings
Frontend	Vercel	Root directory frontend, Vite preset, VITE_* variables set at build time
Backend	Render Web Service	Root directory backend, health check /health, server secrets in the dashboard
Database	Render PostgreSQL	Internal connection string passed to the backend as DATABASE_URL
Deployment notes:

The frontend and backend deploy independently. Neither depends on the other's secrets.
A render.yaml Blueprint can define the API and database. Secret values are entered by the owner in the Render dashboard.
Run migrations as a pre-deploy step or a one-time shell command.
The Render free tier can sleep when idle, so the first request after a pause may be slow. The frontend should show a loading state for this.
Deploy order: database, backend, then frontend (it needs the API URL). After the frontend URL is known, set FRONTEND_URL on Render and add the URL to Supabase redirect settings.
Environment variables by platform
Variable	Platform	Public / Secret
DATABASE_URL	Render	Secret
NODE_ENV, PORT	Render	Public
FRONTEND_URL	Render	Public
LOCAL_FRONTEND_URL	Render (and local .env)	Public
EXTRA_ALLOWED_ORIGINS (optional)	Render	Public
SUPABASE_URL	Render and Vercel	Public
SUPABASE_JWT_SECRET or SUPABASE_JWKS_URL	Render	Secret / Public
AI_PROVIDER_API_KEY (optional)	Render	Secret
AI_MODEL (optional)	Render	Public
VITE_API_BASE_URL	Vercel	Public
VITE_SUPABASE_URL	Vercel	Public
VITE_SUPABASE_ANON_KEY	Vercel	Public
13. Full directory structure
This is the intended layout, as a monorepo. Adjust to the existing project after the Phase 1 audit.

daily-health-companion/
├── README.md
├── ARCHITECTURE.md
├── PRD.md
├── LICENSE
├── .gitignore                         # Includes .env, node_modules, dist
├── render.yaml                        # Render Blueprint: API + PostgreSQL
│
├── frontend/                          # Deployed on Vercel
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── vercel.json                    # SPA rewrites to index.html
│   ├── index.html
│   ├── .env.example                   # VITE_API_BASE_URL, VITE_SUPABASE_URL, VITE_SUPABASE_ANON_KEY
│   ├── public/
│   │   ├── favicon.ico
│   │   └── sounds/
│   │       └── reminder-gentle.mp3
│   └── src/
│       ├── main.tsx
│       ├── App.tsx
│       ├── config/
│       │   └── env.ts                 # Reads VITE_ variables
│       ├── lib/
│       │   ├── apiClient.ts           # fetch wrapper, adds Bearer token, handles errors
│       │   ├── supabaseClient.ts
│       │   └── time.ts                # Timezone helpers
│       ├── data/                      # Data-access layer (demo vs account)
│       │   ├── dataProvider.ts        # Chooses localStore or apiStore
│       │   ├── localStore.ts          # Demo mode, browser storage
│       │   └── apiStore.ts            # Account mode, calls backend
│       ├── auth/
│       │   ├── AuthProvider.tsx
│       │   ├── useAuth.ts
│       │   └── SignInPage.tsx
│       ├── theme/
│       │   ├── themes.ts              # Calm Blue, Nature Green, Warm Sunset, High Contrast
│       │   └── ThemeProvider.tsx      # Theme and text size (standard, large, extra-large)
│       ├── features/
│       │   ├── dashboard/
│       │   ├── logs/
│       │   │   ├── QuickAddLog.tsx
│       │   │   ├── LogList.tsx
│       │   │   └── forms/             # One form per category
│       │   ├── reminders/
│       │   │   ├── ReminderForm.tsx
│       │   │   ├── ReminderList.tsx
│       │   │   ├── CopyYesterdayDialog.tsx
│       │   │   └── alerts/
│       │   │       ├── useDueReminders.ts
│       │   │       ├── ReminderAlert.tsx      # Visual alert, snooze, mark done
│       │   │       └── sound.ts               # Sound, test control, volume
│       │   ├── assistant/
│       │   │   ├── AssistantPanel.tsx
│       │   │   ├── DraftConfirmation.tsx      # User approves before saving
│       │   │   └── demoAssistant.ts
│       │   ├── summaries/
│       │   ├── voice/
│       │   │   ├── useSpeechInput.ts
│       │   │   └── useSpeechOutput.ts
│       │   ├── preferences/
│       │   │   ├── SetupWizard.tsx        # Skippable onboarding
│       │   │   └── PreferencesPage.tsx
│       │   └── privacy/
│       │       ├── DataOverview.tsx       # View and delete own records
│       │       └── ImportLocalData.tsx    # Explicit approval required
│       ├── components/                    # Shared accessible UI components
│       ├── hooks/
│       ├── types/
│       └── tests/
│
├── backend/                           # Deployed on Render
│   ├── package.json
│   ├── tsconfig.json
│   ├── .env.example
│   ├── src/
│   │   ├── server.ts                  # Startup and graceful shutdown
│   │   ├── app.ts                     # Express setup and middleware order
│   │   ├── config/
│   │   │   └── env.ts                 # Validates environment at startup
│   │   ├── middleware/
│   │   │   ├── auth.ts                # Verifies token, sets userId
│   │   │   ├── cors.ts
│   │   │   ├── rateLimit.ts
│   │   │   ├── validate.ts
│   │   │   └── errorHandler.ts
│   │   ├── routes/
│   │   │   ├── health.ts
│   │   │   ├── me.ts
│   │   │   ├── logs.ts
│   │   │   ├── reminders.ts
│   │   │   ├── summaries.ts
│   │   │   ├── assistant.ts
│   │   │   └── import.ts
│   │   ├── schemas/                   # Zod schemas per route and log category
│   │   ├── services/
│   │   │   ├── logService.ts
│   │   │   ├── reminderService.ts
│   │   │   ├── occurrenceService.ts   # Due time, alert-once, snooze, complete
│   │   │   ├── scheduleCopyService.ts
│   │   │   ├── summaryService.ts
│   │   │   └── ai/
│   │   │       ├── aiService.ts
│   │   │       ├── providerAdapter.ts # Provider interface
│   │   │       ├── prompts.ts         # Safety system prompt
│   │   │       └── demoResponse.ts
│   │   ├── repositories/              # All SQL, always filtered by user_id
│   │   │   ├── profileRepo.ts
│   │   │   ├── preferencesRepo.ts
│   │   │   ├── logRepo.ts
│   │   │   ├── reminderRepo.ts
│   │   │   └── occurrenceRepo.ts
│   │   ├── db/
│   │   │   ├── pool.ts
│   │   │   ├── migrate.ts             # Migration runner
│   │   │   └── migrations/
│   │   │       ├── 001_create_profiles_preferences.sql
│   │   │       ├── 002_create_health_logs.sql
│   │   │       ├── 003_create_reminders.sql
│   │   │       └── 004_create_reminder_occurrences.sql
│   │   ├── utils/
│   │   │   ├── logger.ts              # Redacts sensitive fields
│   │   │   ├── timezone.ts
│   │   │   └── errors.ts
│   │   └── types/
│   └── tests/
│       ├── auth.test.ts
│       ├── ownership.test.ts          # User A cannot access user B's data
│       ├── validation.test.ts
│       ├── reminders.test.ts          # Timezone, repeat, daylight saving
│       ├── alerts.test.ts             # One alert per occurrence
│       ├── scheduleCopy.test.ts
│       ├── assistant.test.ts          # Demo mode, drafts only, no-diagnosis
│       └── helpers/
│
├── docs/
│   └── api.md                         # Documented API contracts
│
└── scripts/
    └── scan-secrets.sh                # Checks repo and frontend bundle for leaked secrets
Notes on the layout:

render.yaml lives at the repo root. The Render service uses backend as its root directory.
vercel.json lives inside frontend/. The Vercel project uses frontend as its root directory.
.env files are never committed. Only .env.example files are.
If the existing project is not a monorepo, keep its structure and place the backend in a new backend/ folder alongside it.
14. Observability and operations
/health returns a simple status with no sensitive data, used by Render health checks.
Structured logs record request ID, route, status, and duration, without health payloads.
Errors are logged with context but without secrets or personal health values.
Configure database backups and retention on the Render plan before any real use.
Track the PRD success metrics that need instrumentation (reminder creation success, duplicate-alert rate, AI error rate, sign-in success) in a privacy-preserving way, without health values.
15. Testing strategy
Level	Focus
Unit	Timezone and due-time calculation, repeat rules, validation schemas
Integration	Routes with a test database, auth middleware, error format
Security	Missing, invalid, and expired tokens, cross-user access attempts, CORS rejection
Reminders	One alert per occurrence, snooze, mark done, daylight saving edge cases
Copy schedule	Confirmation required, no duplicates, no completion status or logs copied
AI	Demo mode, drafts not auto-saved, no-diagnosis behavior (provider mocked)
Accessibility	Keyboard navigation, focus, contrast, text sizes, reduced motion, screen reader labels
Frontend	Theme and text-size persistence, demo mode without a backend
Build	Type check, lint, and build for frontend and backend
Secrets	Scan the repository and the frontend bundle
16. Key decisions and open questions
Decisions
Decision	Reason	Trade-off
Frontend on Vercel, backend on Render	Matches the owner's hosting choice, independent deploys	Two platforms, CORS required
Bearer-token auth, no cookies	Works across different domains	Token handling in the frontend
Polling for due reminders	Simple, no extra infrastructure	Alerts only while the app is open
Backend-side AI calls	Protects the key and limits data sent	Adds latency and server cost
Draft-then-confirm for AI actions	Prevents unapproved changes	One more step for the user
Local-to-account import needs consent	Respects user control (PRD)	Extra UI work
Open decisions (from the PRD)
Which AI provider and model, if AI mode is enabled?
Which authentication provider and email sign-in method will be configured?
Which languages should the first release support?
Should caregiver access follow the MVP, and what consent model is needed?
What data export and account deletion process is required before a broader release?
Later considerations
Caregiver access with explicit, user-granted permissions
Data export and a full account deletion workflow
More languages and expanded speech input and output
Web Push notifications through a service worker, so alerts can arrive when the app is closed
Integrations with health platforms or devices, only after privacy, security, and clinical review# Identity

You are a practical AI assistant who helps people get work done using the available tools. Give clear, direct, and well-structured answers. Match the user’s requested tone and format. Verify facts before replying, and state uncertainty or assumptions instead of guessing.
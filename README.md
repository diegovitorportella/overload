# 🏋️ OverLoad

> Full-stack workout log for **progressive overload**: log sets even without internet, track records and progress, and sync your data across devices.

![Status](https://img.shields.io/badge/status-planning-blue)
![Angular](https://img.shields.io/badge/Angular-21-dd0031)
![NestJS](https://img.shields.io/badge/NestJS-API-e0234e)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-data-336791)
![PWA](https://img.shields.io/badge/PWA-offline--first-5a0fc8)
![License](https://img.shields.io/badge/license-MIT-green)

---

## 📑 Table of Contents

1. [Name and Description](#1-name-and-description)
2. [Goal and Problem Solved](#2-goal-and-problem-solved)
3. [Target Audience](#3-target-audience)
4. [MVP Features](#4-mvp-features)
5. [Future Features](#5-future-features)
6. [Tech Stack and Rationale](#6-tech-stack-and-rationale)
7. [Architecture and Organization](#7-architecture-and-organization)
8. [Initial Folder Structure](#8-initial-folder-structure)
9. [Step-by-Step Development Plan](#9-step-by-step-development-plan)
10. [Best Practices and Code Standards](#10-best-practices-and-code-standards)
11. [Technical Challenges and Skills Developed](#11-technical-challenges-and-skills-developed)
12. [Future Evolution](#12-future-evolution)

---

## 1. Name and Description

**OverLoad** is a progressive web app (PWA) for logging strength workouts and tracking progress over time. The name comes from *progressive overload*: gradually increasing weight, reps or volume.

The project is **local-first with synchronization**:

- **Front-end (Angular PWA):** stores everything on the device and works without internet. It is the source of truth during a workout.
- **Back-end (NestJS + PostgreSQL):** authenticates the user, keeps a cloud copy of the data and syncs multiple devices.

## 2. Goal and Problem Solved

### Problem
Spreadsheets and notes are slow during a workout and do not reveal trends. Many apps require a constant connection (gyms often have poor signal) or charge for basic features.

### Goal
- **Fast logging with few taps per set**, never dependent on the network.
- **Clear** progress analysis: records, volume and estimated 1RM.
- **Reliable sync** between phone and computer, without losing data.

### Learning goal
Practice dynamic forms, PWA and local persistence on the front-end, and **authenticated API, relational database, synchronization and conflict resolution** on the back-end.

## 3. Target Audience

- **Strength training practitioners** (beginner to intermediate).
- **People who train on their own** and use more than one device.
- **Recruiters and developers**, as a complete offline-first product showcase.

## 4. MVP Features

### Front-end
| # | Feature | Description |
|---|---|---|
| 1 | **Exercise library** | Predefined list by muscle group + custom exercises. |
| 2 | **Workout routines** | Create "Workout A / B / C" with exercises and set targets. |
| 3 | **Workout logging** | Log sets (weight and reps), add or remove sets and exercises. |
| 4 | **Last workout reference** | Previous performance shown next to the input field. |
| 5 | **Rest timer** | Starts when a set is completed, with an alert when it ends. |
| 6 | **Personal records (PRs)** | Heaviest weight, highest volume and best estimated 1RM. |
| 7 | **History and charts** | List, calendar and progress per exercise. |
| 8 | **Fully offline** | Installable app; log and browse without internet. |
| 9 | **Sync indicator** | Shows "synced", "pending" or "error". |
| 10 | **Account and login** | Sign-up, login and automatic session refresh. |
| 11 | **Units and theme** | kg/lb, light/dark. |
| 12 | **Export and import** | File backup, in addition to the cloud. |

### Back-end
| # | Feature | Description |
|---|---|---|
| 13 | **Authentication** | Sign-up, login, short-lived access token and refresh token. |
| 14 | **User data API** | Exercises, routines, sessions and settings, always isolated per user. |
| 15 | **Sync endpoint** | Receives the device's pending changes and returns new changes since the last sync. |
| 16 | **Conflict resolution** | A defined, documented rule (last write wins, per record). |
| 17 | **Soft deletion** | Deleted items become markers, so removals propagate across devices. |
| 18 | **API documentation** | Browsable OpenAPI specification. |

**Out of the MVP:** social login, AI-generated plans, social features.

## 5. Future Features

- Automatic next-weight suggestions.
- Plateau detection and deload weeks.
- Plate and warm-up calculators.
- Body measurements (weight, circumferences).
- Supersets and set types (drop set, to failure).
- Weekly summary and frequency goals.
- Share workout summary as an image.
- Internationalization (pt-BR, en, es).

## 6. Tech Stack and Rationale

| Layer | Technology | Rationale |
|---|---|---|
| Front | **Angular (current stable) + strict TypeScript** | Robust reactive forms and opinionated structure. |
| Front | **Reactive Forms (`FormArray`)** | Nested logging: session → exercises → sets. |
| Front | **NgRx (Store, Effects, Entity, Selectors)** | Active session, history and library; stats as derived state. |
| Front | **IndexedDB (lightweight access library)** | Asynchronous, indexable local storage; powers offline. |
| Front | **Service Worker (Angular PWA)** | App caching and installation. |
| Front | **Chart.js** | Progress charts. |
| Front | **RxJS** | Timer, debounced auto-save, sync queue. |
| Back | **NestJS (TypeScript)** | Modular structure similar to Angular; validation, guards and documentation built in. |
| Back | **PostgreSQL + ORM with migrations** | Relational data (user → routine → session → set) and versioned schema. |
| Back | **JWT (access + refresh)** | Industry standard for a stateless API; strongly hashed passwords. |
| Back | **OpenAPI** | Documented contract. |
| Infra | **Docker Compose** | Reproducible database and API with one command. |
| Quality | **Jest (front and back) + Supertest** | Domain logic, API and sync tested. |
| Quality | **ESLint + Prettier + CI** | Consistency and automated feedback. |
| Shared | **`contracts` (types) and `domain` (calculations)** | Volume, 1RM and PR calculations become a pure package, used by the front-end (and by the back-end if needed). |

> **Decision:** sync uses **simple, explicit rules** (client-generated identifiers, timestamp/version, soft deletion), with no complex CRDT libraries. The goal is to understand the problem, not hide it.

## 7. Architecture and Organization

### 7.1 Overview

```
┌──────────────── FRONT-END (Angular PWA) ─────────────────┐
│ Pages/forms → NgRx → Repositories → IndexedDB             │
│                          │                                │
│                  Pending changes queue (outbox)           │
└──────────────────────────┬───────────────────────────────┘
                           │ HTTPS (when online)
                           ▼
┌──────────────── BACK-END (NestJS) ───────────────────────┐
│ Auth (JWT) → Sync controller → Sync service               │
│                                      │                    │
│                                 PostgreSQL                │
└───────────────────────────────────────────────────────────┘
```

### 7.2 How synchronization works
1. Every local create, edit or delete writes to IndexedDB **and** enters a pending queue.
2. Each record has a **client-generated identifier** (avoids duplicates on resend) and a **version/modification date**.
3. When online and logged in, the app sends the queue to the server in a batch.
4. The server applies changes **idempotently**, resolves conflicts with the defined rule (latest modification wins, per record) and returns what changed since the device's last sync.
5. The app applies what it received, clears the queue and updates the status indicator.
6. Deletions are **soft**, so other devices learn about removals.
7. Network failures only let the queue grow; nothing is lost.

### 7.3 Principles
- **Pure domain:** volume, 1RM and PRs in framework-independent functions, 100% testable.
- **Repository as a boundary on the front-end:** the store talks to repositories, never to IndexedDB directly.
- **The device rules during a workout:** the UI never waits for the server.
- **Per-user isolation on the back-end:** every query filters by the authenticated user; test this explicitly.
- **Thin controllers:** rules in services; data access in repositories.
- **Derived stats:** charts and PRs computed by selectors, not stored redundantly.
- **Mobile-first**, with large touch targets.

### 7.4 API contract (summary)

| Resource | Operations |
|---|---|
| Authentication | register, log in, refresh token, log out |
| Sync | send pending changes, get changes since a checkpoint |
| Profile | read and update settings |
| Export | download all user data |
| Health | status check |

### 7.5 Data model (conceptual)
**User** · **Exercise** · **Routine** · **Routine item** · **Session** · **Set** · **Settings** · **Refresh token**. All workout records carry: owner, version, modification date and deletion marker.

## 8. Initial Folder Structure

```
overload-log/
├── README.md
├── package.json                  # workspaces: frontend, backend, contracts, domain
├── docker-compose                # PostgreSQL + API
├── .github/                      # CI: lint, tests, build
├── contracts/                    # shared types and DTOs
├── domain/                       # pure functions: volume, 1RM, PRs, progression
├── backend/
│   ├── .env.example
│   ├── migrations/
│   └── src/
│       ├── main
│       ├── config/
│       ├── common/               # guards, error filters, interceptors
│       └── modules/
│           ├── auth/
│           ├── users/
│           ├── sync/             # receives pending changes, resolves conflicts, returns changes
│           └── export/
└── frontend/
    └── src/
        ├── manifest              # PWA name, icons and colors
        ├── assets/seed/          # predefined exercises
        ├── styles/
        └── app/
            ├── core/
            │   ├── persistence/  # IndexedDB + repositories
            │   ├── sync/         # pending queue + sync client
            │   ├── auth/
            │   └── services/     # theme, units, PWA install, backup
            ├── shared/
            ├── store/
            └── features/
                ├── auth/
                ├── exercises/
                ├── routines/
                ├── workout/      # active session, form, timer
                ├── history/
                ├── progress/
                └── settings/
```

## 9. Step-by-Step Development Plan

Estimate for part-time work: **11 to 14 weeks**. Tip: **the app works 100% locally until Phase 6**; the back-end comes in after the core product already works.

### Phase 0 — Foundation (4 to 5 days)
- [ ] Create the monorepo with workspaces (`frontend`, `backend`, `contracts`, `domain`).
- [ ] Configure ESLint, Prettier, Jest and CI.
- [ ] Define design tokens (mobile-first) and light/dark theme.

### Phase 1 — Domain (1 week)
- [ ] Define the models and contracts.
- [ ] Implement total volume, estimated 1RM and PR detection.
- [ ] Extensive tests with edge cases (zero, empty sets, ties).
- [ ] Document the chosen formulas and why.

### Phase 2 — Local persistence (1 week)
- [ ] Configure IndexedDB and repositories.
- [ ] Initial seed of exercises.
- [ ] Local schema versioning strategy.
- [ ] Include in every record: identifier, version, modification date and deletion marker.

### Phase 3 — State, exercises and routines (1.5 to 2 weeks)
- [ ] NgRx with initial hydration from local storage.
- [ ] Exercise library (search, filters, custom exercises).
- [ ] Routine editor (add, remove, reorder).

### Phase 4 — Workout logging (2 weeks) — **core of the app**
- [ ] Dynamic form (session → exercises → sets).
- [ ] Last workout reference for each exercise.
- [ ] Rest timer with sound or vibration alert.
- [ ] Continuous saving and session recovery after closing the app.
- [ ] Finish session: volume, PRs and summary.

### Phase 5 — History, progress and PWA (1.5 weeks)
- [ ] Workout list and calendar.
- [ ] Per-exercise charts and records screen.
- [ ] Service worker, manifest, installation and 100% offline test.
- [ ] Export and import with validation.

> ✅ **Milestone:** at this point you already have a complete, usable app, local only.

### Phase 6 — Back-end and authentication (1.5 to 2 weeks)
- [ ] Create the NestJS application, validated configuration and health route.
- [ ] Model the database and create migrations.
- [ ] Sign-up, login, password hashing, access and refresh tokens.
- [ ] Guards and rate limiting.
- [ ] OpenAPI and API tests.

### Phase 7 — Synchronization (2 to 2.5 weeks) — **biggest challenge**
- [ ] Idempotent sync endpoint, with batched changes.
- [ ] Conflict rule and soft deletion implemented and documented.
- [ ] Pending queue and sync client on the front-end.
- [ ] Status indicator and automatic resend when the connection returns.
- [ ] Scenario tests: two devices editing, long offline period, duplicate resend, concurrent deletion.
- [ ] Per-user isolation tests.

### Phase 8 — Polish and release (1 week)
- [ ] Review accessibility, performance and responsiveness.
- [ ] Package with Docker and deploy front and API to free hosting.
- [ ] Final README with screenshots, GIF and sync diagram.

## 10. Best Practices and Code Standards

**General**
- Strict TypeScript, no `any`; shared contracts.
- Small commits (*Conventional Commits*); code names in English.
- **Never** commit secrets; use `.env.example`.

**Domain**
- Pure functions, no side effects; every calculation tested.
- Always store in a base unit; convert only for display.

**Front-end**
- `OnPush`, standalone components, lazy loading.
- Typed forms; form construction in a dedicated factory.
- Persistence and sync as effects in Effects, never in components.
- Save incrementally: never lose a session in progress.
- Handle local write failures (full quota, private mode) with user feedback.

**Back-end**
- Validate all input with DTOs; standardized error responses.
- Versioned migrations; no manual database changes.
- Every query filtered by the authenticated user.
- Strongly hashed passwords; do not log sensitive data.
- Idempotent, transactional sync operations.
- Tests: service unit tests, API integration and sync scenarios.

**UX**
- Large touch targets, few taps per set.
- Accessibility: visible focus, contrast, labels.

## 11. Technical Challenges and Skills Developed

| Challenge | What you learn |
|---|---|
| Nested, dynamic form | `FormArray`, typed forms, validation |
| Domain calculations | Pure functions, TDD, edge cases |
| Local-first with IndexedDB | Async storage, repositories, local migrations |
| Offline PWA | Service Worker, caching, installation |
| **Sync and conflicts** | Idempotency, versioning, soft deletion, eventual consistency |
| Authentication with token refresh | JWT, password hashing, guards, interceptors |
| Multi-user isolation | Per-owner authorization, security tests |
| Relational modeling | Keys, indexes, migrations |
| Derived state for charts | Selectors and memoization |
| Containerization and CI | Docker Compose, automated pipeline |

## 12. Future Evolution

- **Progression suggestions** and plateau detection.
- **Social login** and password recovery.
- **Optimized incremental sync** (send only per-field differences).
- **Body measurements** and correlation with performance.
- **Notifications** as workout reminders.
- **End-to-end tests** (start → log → sync → see PR on another device).
- **Import** from other apps and spreadsheets.
- **Redis caching and queues**, if load justifies it.
- **Full internationalization**.

---

## 📝 Final Notes

- This document is a **planning guide**; adjust the scope to your pace.
- Build **the complete local app first**; sync is much easier on a stable base.
- After each completed phase, update the checkboxes and record what you learned.

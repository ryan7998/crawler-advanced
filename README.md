# CrawlerPro — Advanced Web Scraping Platform

**Canonical repo:** [github.com/ryan7998/crawler](https://github.com/ryan7998/crawler). Clone and open issues there.

A full-stack web application for crawling websites, extracting structured data with custom CSS selectors, and exporting results to Google Sheets or CSV. Features real-time progress updates, a Bull/Redis job queue, proxy usage analytics, and role-based authentication.

## Live Demo

[http://crawler.onthis.website/](http://crawler.onthis.website/)

---

## Feature Highlights

- **Multi-URL crawling** with custom CSS selector configurations
- **Real-time status updates** via Socket.IO — watch each URL complete live
- **3-step crawl wizard** — create or edit crawls at any step, no need to finish the wizard to save
- **Bull/Redis job queue** with queue status monitoring
- **Data export** to Google Sheets or CSV/Excel
- **Proxy usage analytics** — per-crawl and global statistics with cost analysis
- **Stuck crawl recovery** — worker restarts automatically reset `in-progress` crawls
- **Role-based auth** — `user`, `admin`, `superadmin`
- **Change detection** — compare crawl runs to identify changed content

---

## Tech Stack

### Frontend

| Technology | Version | Role |
|---|---|---|
| Vue 3 | `^3.4` | UI framework (Composition API / `<script setup>`) |
| Pinia | `^2.2` | State management |
| Vue Router | `^4` | Client-side routing |
| Tailwind CSS | `^3.4` | Styling — only CSS framework used |
| Axios | `^1.7` | HTTP client |
| Socket.IO Client | `^4.7` | Real-time crawl progress |
| Vite | `^5.4` | Build tool and dev server |

> **Note:** Vuetify has been fully removed. All UI is implemented with Tailwind CSS utility classes.

### Backend

| Technology | Version | Role |
|---|---|---|
| Node.js + Express | v18+ | REST API server and Bull worker |
| MongoDB + Mongoose | — | Data persistence and aggregation |
| Bull + Redis | — | Job queue for crawl tasks |
| Socket.IO | `^4.7` | Real-time crawl progress broadcasting |
| Playwright | — | Headless browser page loading |
| Cheerio | — | HTML parsing and selector extraction |
| Joi | — | Server-side request validation |
| bcryptjs | — | Password hashing |
| jsonwebtoken | — | JWT authentication |

---

## Project Structure

```
crawler/
├── client/                       # Vue 3 frontend
│   ├── src/
│   │   ├── App.vue
│   │   ├── main.js
│   │   ├── router.js
│   │   ├── index.css             # Tailwind directives
│   │   ├── components/
│   │   │   ├── features/crawl/   # Domain-specific crawl UI
│   │   │   ├── forms/            # Standalone form components
│   │   │   ├── layout/           # App shell (Navbar, ModalWrapper, StatsWrapper…)
│   │   │   ├── modals/           # All modal dialogs
│   │   │   ├── pages/            # Route-level page components
│   │   │   └── ui/               # Generic UI primitives (table, status pill, slide-over…)
│   │   ├── composables/          # Reusable Composition API logic
│   │   ├── constants/            # Shared constant objects
│   │   ├── stores/               # Pinia stores (crawl, auth, statsBar)
│   │   └── utils/                # Pure helper functions
│   ├── FRONTEND.md               # Frontend developer documentation
│   └── package.json
│
└── server/                       # Node.js backend (two processes)
    ├── src/
    │   ├── app.js                # Express API server (:3001)
    │   ├── worker.js             # Bull worker + Socket.IO (:3002)
    │   ├── classes/              # Playwright Seed + ErrorHandler
    │   ├── controllers/          # All route handler functions
    │   ├── middleware/           # authenticate, asyncHandler, errorHandler
    │   ├── models/               # Mongoose schemas (Crawl, CrawlData, User, ProxyUsage, Selectors)
    │   ├── queues/               # getCrawlQueue — cached Bull queue factory
    │   ├── routes/               # authRoutes, crawlerRoutes, exportRoutes, oauth2Routes
    │   ├── scripts/              # One-off migration & admin scripts
    │   ├── seeders/              # Per-domain CSS selector seed data
    │   ├── services/             # AuthService, ChangeDetectionService, ProxyUsageService…
    │   └── utils/                # responseUtils, validationUtils, aggregationPipelines…
    ├── utils/                    # Legacy root-level utilities (helperFunctions, socket)
    └── BACKEND.md                # Backend developer documentation
```

---

## Prerequisites

- Node.js v18+
- MongoDB (local or Atlas)
- Redis (local or managed)
- npm

---

## Quick Start

### 1. Clone and install

```bash
git clone https://github.com/ryan7998/crawler.git
cd crawler

# Server
cd server && npm install

# Client
cd ../client && npm install
```

### 2. Environment variables

**`server/.env`**

```env
# Required
MONGO_URI=mongodb://localhost:27017/crawler_db
JWT_SECRET=your_jwt_secret_change_in_production

# Optional (defaults shown)
PORT=3001                # API server
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
JWT_EXPIRES_IN=7d
NODE_ENV=development
ORIGIN=http://localhost:5173
CRAWL_CONCURRENCY=3

# Proxy (Oxylabs) — optional
OXYLAB_SERVER=
OXYLAB_USERNAME=
OXYLAB_PASSWORD=

# Google Sheets OAuth2 — optional
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_REDIRECT_URI=
```

**`client/.env`**

```env
VITE_BASE_URL=http://localhost:3001
VITE_SOCKET_URL=http://localhost:3002
```

### 3. Run in development

```bash
# Terminal 1 — API server
cd server && npm run dev

# Terminal 2 — Worker process
cd server && npm run worker

# Terminal 3 — Vue dev server
cd client && npm run dev
```

| Service | URL | Notes |
|---|---|---|
| Frontend | http://localhost:5173 | Vite dev server |
| API | http://localhost:3001 | Express REST API |
| Worker / Socket | http://localhost:3002 | Bull worker + Socket.IO |

> The API server and worker must both be running for crawls to execute. The API enqueues jobs; the worker processes them.

---

## API Reference

### Auth (`/api/auth`)

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `POST` | `/api/auth/register` | — | Register a new admin user |
| `POST` | `/api/auth/login` | — | Login, returns JWT |
| `POST` | `/api/auth/logout` | ✓ | Logout |
| `GET` | `/api/auth/verify` | ✓ | Verify token |
| `GET` | `/api/auth/profile` | ✓ | Get own profile |
| `PUT` | `/api/auth/profile` | ✓ | Update own profile |
| `PUT` | `/api/auth/change-password` | ✓ | Change password |
| `GET` | `/api/auth/users` | ✓ superadmin | List all users |

### Crawls (`/api`)

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/getallcrawlers` | List crawls (paginated, searchable) |
| `GET` | `/api/getcrawler/:id` | Get single crawl with aggregated data |
| `POST` | `/api/createcrawler` | Create crawl |
| `PUT` | `/api/updatecrawl/:id` | Update crawl |
| `DELETE` | `/api/deletecrawl/:id` | Delete crawl and all its data |
| `POST` | `/api/startcrawl` | Queue crawl job(s) for specific URLs |
| `POST` | `/api/runallcrawls` | Queue all enabled crawls |
| `DELETE` | `/api/deletecrawldata/:id` | Clear all scraped data for a crawl |
| `DELETE` | `/api/deletecrawldata/:id/urls` | Clear data for specific URLs |
| `DELETE` | `/api/clearqueue/:crawlId` | Clear Bull queue for a crawl |
| `GET` | `/api/queuestatus/:crawlId` | Queue job counts (active/waiting/done) |
| `GET` | `/api/crawls/:id/proxy-stats` | Per-crawl proxy usage stats |
| `GET` | `/api/proxy-stats/global` | System-wide proxy usage stats |
| `GET` | `/api/proxy-stats/cost-analysis` | Cost analysis with date range |

### Export (`/api/export`)

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/export/google-sheets/:crawlId` | Export one crawl to Google Sheets |
| `POST` | `/api/export/google-sheets/multiple` | Export multiple crawls to one sheet |
| `POST` | `/api/export/google-sheets/global` | Global crawl export |
| `GET` | `/api/export/changes/:crawlId` | Change analysis between crawl runs |
| `GET` | `/api/export/csv/:crawlId` | Download change data as CSV |

---

## Architecture Overview

### Frontend

```
App.vue
 ├── NotificationWrapper   (toast provider)
 │    ├── Navbar
 │    ├── StatsWrapper      (auth-only — wraps router-view with a bottom stats bar)
 │    │    └── StatsBottomBar  → GlobalStatsBar | CrawlDetailsStatsBar
 │    ├── <router-view>     (unauthenticated fallback)
 │    └── ModalWrapper      (renders all global modals, reads crawlStore flags)
```

**Modal system**: All modals are rendered centrally in `ModalWrapper`. Components open modals by calling `crawlStore.open*Modal()`. Saves are handled in `ModalWrapper`, which updates the store and fires `crawlStore.triggerRefresh()` to notify any watching component to re-fetch.

**Real-time**: `CrawlDetailsView` connects to a Socket.IO room keyed by `crawlId`. Incoming `crawlLog` events update `liveStatusDictionary` and append to `aggregatedData` without a round-trip API call.

### Backend / Worker

The Express API and the Bull worker run as **two separate Node.js processes**:

- **API server** (`PORT=3001`) — CRUD, auth, export, proxy stats
- **Worker** (`PORT=3002`) — Bull job processing, Playwright page loads, Socket.IO events

On startup the worker:
1. Connects to MongoDB
2. Scans for any `in-progress` crawls (stuck from a previous crash) and resets them to `pending`
3. Registers a Bull processor for each resumed crawl
4. Listens for new `/processor/:crawlId` calls to activate per-crawl job processing

**Per-crawl rate-limiting** — each crawl gets its own throttle instance (1 Playwright request / 2 s) to avoid overloading target sites. Bull itself is capped at 5 jobs / 3 s as a secondary limit.

---

## Developer Documentation

### Frontend

See **[`client/FRONTEND.md`](client/FRONTEND.md)** for:

- Full component catalogue with props/emits
- All composables documented with usage examples
- Pinia store API reference
- Styling conventions and copy-paste patterns
- Data flow diagrams
- Checklist for adding new features

### Backend

See **[`server/BACKEND.md`](server/BACKEND.md)** for:

- Full route reference with auth requirements
- Controller patterns (`asyncHandler`, `sendSuccess`)
- Model schemas with all fields and indexes
- Service API reference (AuthService, ChangeDetectionService, ProxyUsageService…)
- Queue and worker internals
- Error handling patterns
- Authentication & authorization guide
- Crawl lifecycle data flow
- Checklist for adding new endpoints, models, and migrations

---

## Contributing

1. Fork the repo
2. Create a feature branch: `git checkout -b feature/my-feature`
3. Follow the conventions in [`client/FRONTEND.md`](client/FRONTEND.md) for frontend changes
4. Follow the conventions in [`server/BACKEND.md`](server/BACKEND.md) for backend changes
5. Open a pull request with a description of what changed and why

---

## License

ISC

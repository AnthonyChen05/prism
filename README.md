# Prism

**A modular page system for vibe-coded web apps.**

Drop a folder in. Get a page. That's it.

Prism is a self-hosted backend that automatically discovers and serves any module you create — no routing config, no wiring, no framework boilerplate. It's built for people who want to vibe-code their own tools: personal dashboards, project trackers, habit apps, kanban boards, CRMs, or anything else you'd normally pay $15/month for.

---

## The idea

Tools like Monday.com, Notion, Linear, and Airtable are powerful — but they're also bloated, expensive, and built for someone else's workflow. With Prism, you own the stack. You write the UI exactly how you want it, drop it in a folder, and it's live.

Want a custom project tracker? Write it. A personal CRM? Write it. A habit tracker that looks exactly the way you want? Write it. Vibe code the whole thing in an afternoon and run it yourself.

---

## How modules work

Every page is a **module** — a folder inside `backend/src/modules/` with an `index.ts` and a `public/` folder.

```
src/modules/
├── dashboard/          → served at /dashboard (landing page by default)
│   ├── index.ts        → backend routes
│   └── public/
│       ├── index.html
│       ├── app.js
│       └── style.css
├── kanban/             → served at /kanban  (drop it in, it just works)
│   ├── index.ts
│   └── public/
│       ├── index.html
│       └── app.js
└── crm/                → served at /crm
    ├── index.ts
    └── public/
        └── index.html
```

The server auto-discovers every module on startup. No registration, no config changes — just drop a folder in and restart.

---

## Creating a module

A module needs one file: `src/modules/<name>/index.ts`

```typescript
import type { AppModule } from '../../shared/types/module'

const MyModule: AppModule = {
  name: 'my-module',
  version: '1.0.0',

  async register(server, services, prefix) {
    // prefix = "/my-module" — derived from folder name automatically

    // Serve your HTML page
    server.get(prefix, { config: { public: true } } as never, async (_req, reply) => {
      reply.type('text/html').send('<h1>Hello from my module</h1>')
    })

    // Add API routes scoped to your prefix
    server.get(`${prefix}/api/data`, { config: { public: true } } as never, async () => {
      return { items: await services.db.yourModel.findMany() }
    })
  }
}

export default MyModule
```

Add a `public/` folder next to it and your static files (HTML, CSS, JS) are automatically served at `/<name>-assets/`.

In your HTML, use `{{ASSETS}}` as a placeholder for the asset path — it gets replaced at serve time:

```html
<link rel="stylesheet" href="{{ASSETS}}/style.css" />
<script src="{{ASSETS}}/app.js"></script>
```

In your JS, use `window.location.pathname` as the API base so everything routes correctly regardless of folder name:

```js
const API = window.location.pathname.replace(/\/$/, '')
const data = await fetch(API + '/api/data').then(r => r.json())
```

---

## What's included

### The dashboard module

The built-in dashboard is a personal home page with:

- **Quick Links** — bookmark your most-used sites
- **To-Do List** — simple persistent tasks
- **Google Calendar** — embed your calendar with one URL
- **RSS Feeds** — follow any RSS/Atom feed
- **Custom background** — upload your own image (stored locally, no server needed)

It's also a good reference for how to build your own module.

### Core services

Every module gets access to the full service layer:

```typescript
interface CoreServices {
  db: PrismaClient        // PostgreSQL via Prisma — query anything
  time: TimeService       // Timezone-aware date/time (Luxon)
  notify: NotificationService  // Send real-time notifications
  timer: TimerService     // Schedule actions to fire after a delay
  scheduler: Scheduler    // Raw BullMQ job scheduling
  events: EventBus        // Pub/sub between modules
}
```

Modules can talk to each other via `services.events`. One module emits, another listens — zero coupling.

### Timer actions

Schedule anything to happen after a delay:

```typescript
// Send a notification in 24 hours
await services.timer.after('daily-reminder', 86_400_000, {
  type: 'notify',
  payload: { userId: 'user-1', title: 'Daily check-in', body: 'How are your tasks looking?' }
})

// Emit an event to other modules in 5 seconds
await services.timer.after('sync-trigger', 5000, {
  type: 'event',
  event: 'crm:sync',
  payload: { source: 'scheduler' }
})
```

---

## Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 20 + TypeScript |
| Framework | Fastify |
| Database | PostgreSQL 16 via Prisma ORM |
| Queue / Jobs | BullMQ (Redis) |
| Realtime | Socket.io |
| Auth | JWT |
| Container | Docker + Docker Compose |

---

## Getting started

### With Docker (recommended)

```bash
git clone https://github.com/AnthonyChen05/prism.git
cd prism/backend

cp .env.example .env

docker compose up -d

# First time only — run migrations
docker compose exec api npx prisma migrate dev --name init

# Open the dashboard
open http://localhost:3000
```

### Local dev (no Docker)

You'll need PostgreSQL and Redis running locally.

```bash
cd backend
npm install
cp .env.example .env
# Update DATABASE_URL and REDIS_HOST in .env

npx prisma generate
npx prisma migrate dev --name init

npm run dev
```

---

## Configuration

| Variable | Default | Description |
|---|---|---|
| `PORT` | `3000` | Server port |
| `LANDING_MODULE` | `dashboard` | Which module to redirect to from `/` |
| `DATABASE_URL` | *(see .env.example)* | PostgreSQL connection string |
| `REDIS_HOST` | `redis` | Redis hostname |
| `JWT_SECRET` | *(change this)* | JWT signing secret |

To change the landing page, set `LANDING_MODULE` to any module folder name:

```env
LANDING_MODULE=kanban
```

---

## Ideas for modules to build

These are the kinds of tools people pay monthly subscriptions for. With Prism you can vibe code your own version in a weekend:

| Tool | What it replaces |
|---|---|
| Kanban board | Trello / Linear |
| Project tracker | Monday.com / Asana |
| Personal CRM | HubSpot (personal tier) |
| Habit tracker | Streaks / Habitica |
| Reading list | Pocket / Instapaper |
| Budget tracker | Mint / YNAB |
| Note-taking | Notion pages |
| Time tracker | Toggl |
| Link board | Linktree |
| Status page | Statuspage.io |

Every one of these is a folder with an HTML file and some API routes.

---

## Project structure

```
prism/
├── backend/
│   ├── src/
│   │   ├── core/
│   │   │   ├── server.ts           # Server bootstrap, static mounts, auth, Socket.io
│   │   │   ├── plugin-loader.ts    # Auto-discovers modules from src/modules/
│   │   │   └── services/           # db, time, notify, timer, scheduler, events
│   │   ├── modules/
│   │   │   ├── dashboard/          # Built-in personal dashboard
│   │   │   ├── time/               # Time + timer API
│   │   │   └── notifications/      # Notification history + Socket.io push
│   │   └── shared/
│   │       └── types/module.ts     # AppModule, CoreServices, TimerAction types
│   ├── prisma/schema.prisma
│   ├── docker-compose.yml
│   ├── Dockerfile
│   └── .env.example
└── README.md
```

---

## License

MIT — do whatever you want with it.

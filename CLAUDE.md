# Cowork V1 — Claude Code Implementation Prompt

> **You are Claude Code building Cowork V1**, an omnichannel customer-conversation platform for SMBs.
> Read this file at the start of every session. Refer back to it whenever you're unsure.
> Everything here is locked. If something isn't in this doc, ask before deciding.

---

## 0. The Project in 60 Seconds

**Cowork V1 is an omnichannel conversation platform for SMBs.** It unifies WhatsApp + Instagram + Messenger DMs into one inbox, replies automatically using fixed pre-approved responses (no AI hallucination), books appointments in a built-in calendar, and saves every customer in a built-in CRM.

**The 4 sacred pillars** (every feature maps to one of these):
1. **Unified Inbox** — WhatsApp + Instagram + Messenger in one view (TikTok is V1.1).
2. **AI Chatbot** — fixed-reply only. The bot picks from the FAQ Library; it cannot generate text.
3. **Calendar & Booking** — native booking calendar with slot management + reminders.
4. **Mini CRM** — every message auto-populates a Contact record with tags, history, status.

**Launch:** Sep 1, 2026 (private beta with 5 design partners). Public launch ~Oct 1, 2026.
**Team:** Belal (engineer), Mahmoud (GTM), Ahmed (investor), Claude Code (you).
**Customers:** SMBs globally — clinics, retail, education, automotive, real estate, professional services, travel.
**Pricing:** Starter $15 (inbox + CRM) · Growth $59 (+ AI chatbot + calendar) · Agency $199 (+ scale + branding) · Enterprise Custom.

---

## 1. Locked Tech Stack — Do Not Re-evaluate

| Layer | Choice | Why |
|---|---|---|
| Framework | Next.js 14 App Router + TypeScript | Server Components + API routes in one repo. |
| Styling | Tailwind CSS + shadcn/ui | Copy-paste components. |
| Database | PostgreSQL on Neon | Free tier, branching, point-in-time backups. |
| ORM | Prisma | Typed schema. |
| Auth | Clerk (with Organizations) | One Org = one Workspace. |
| AI | Anthropic Claude API (`claude-sonnet-4-6`) | Intent classification. |
| WhatsApp | Meta Cloud API (BYO WABA model, customer pays Meta directly) | — |
| Instagram | Instagram Graph API | — |
| Messenger | Messenger Platform | — |
| Email | Resend | Transactional + reminders. |
| Storage | Cloudflare R2 (S3-compatible) | Free egress. For WA attachments. |
| Jobs / Queues | Inngest | Cron + retries + queues + dead-letter. |
| Rate limiting | Upstash Ratelimit | Free tier. Apply to webhooks. |
| Real-time | SWR polling 5–10s on open thread (V1) → Pusher in V1.1 | Cheap, ships now. |
| Payments | Stripe (+ Tap for Gulf later) | — |
| Observability | PostHog (product analytics) + Sentry (errors) + BetterStack (logs) | — |
| Hosting | Vercel | — |
| Font | Inter or Geist (pick Geist) | One font only. |
| Icons | lucide-react | One icon set. |

**Do not add new dependencies without checking with Belal.** Adding a dep is a budget decision.

---

## 2. Repo Structure

```
cw-platform/
├── app/                          # Next.js App Router
│   ├── (auth)/                   # Auth pages (signup, login)
│   ├── (marketing)/              # Public pages (root layout differs)
│   ├── (app)/                    # Authenticated app (uses dashboard layout)
│   │   ├── layout.tsx
│   │   ├── inbox/
│   │   ├── contacts/
│   │   ├── calendar/
│   │   ├── chatbot/              # FAQ Library + Setup Wizard
│   │   ├── automations/          # Custom Follow-Up Rules
│   │   ├── analytics/
│   │   └── settings/
│   ├── api/
│   │   ├── webhooks/
│   │   │   ├── whatsapp/route.ts
│   │   │   ├── instagram/route.ts
│   │   │   ├── messenger/route.ts
│   │   │   └── stripe/route.ts
│   │   ├── cron/
│   │   │   ├── reminders/route.ts
│   │   │   ├── rules/route.ts
│   │   │   └── slot-generator/route.ts
│   │   └── trpc/                 # if we use tRPC; otherwise plain route handlers
│   └── layout.tsx
├── components/
│   ├── ui/                       # shadcn — DO NOT MODIFY, copy-paste from shadcn
│   ├── inbox/
│   ├── crm/
│   ├── calendar/
│   ├── chatbot/
│   └── shared/
├── lib/
│   ├── db.ts                     # Prisma client singleton
│   ├── auth.ts                   # Clerk helpers, WorkspaceContext
│   ├── plan/                     # Plan feature gates
│   │   ├── features.ts           # PlanFeatures matrix
│   │   └── gate.ts               # PlanFeatureGate middleware
│   ├── channels/                 # Channel connectors
│   │   ├── types.ts              # ChannelConnector interface
│   │   ├── whatsapp.ts
│   │   ├── instagram.ts
│   │   ├── messenger.ts
│   │   └── registry.ts           # Map<Channel, ChannelConnector>
│   ├── ai/
│   │   ├── classifyIntent.ts     # Claude API + cheap pre-classifier
│   │   ├── preClassifier.ts      # regex / keyword first-pass
│   │   └── costGuard.ts          # per-workspace LLM cost cap
│   ├── ingest/
│   │   └── ingestMessage.ts      # auto-Contact, auto-Conversation, store Message
│   ├── outbound/
│   │   └── sendMessage.ts        # Inngest-queued send with retry
│   ├── audit/
│   │   └── log.ts                # AuditLog helper
│   ├── rules/
│   │   └── engine.ts             # Custom Follow-Up Rules evaluator
│   ├── billing/
│   │   ├── stripe.ts             # Stripe SDK + webhook verification
│   │   └── plans.ts              # Plan + pricing constants
│   ├── storage/
│   │   └── r2.ts                 # signed-URL helper for attachments
│   ├── ratelimit/
│   │   └── client.ts             # Upstash Ratelimit setup
│   └── utils/                    # tz, dates, formatters
├── prisma/
│   ├── schema.prisma             # See §5 below
│   ├── migrations/
│   └── seed.ts                   # Demo data + vertical presets
├── content/
│   └── verticals/                # Vertical preset libraries
│       ├── clinic.ts
│       ├── retail.ts
│       ├── education.ts
│       ├── auto.ts
│       ├── realestate.ts
│       ├── prof-services.ts
│       └── travel.ts
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/                      # Playwright
├── inngest/
│   ├── client.ts
│   └── functions/                # Each Inngest function in a file
├── public/
├── middleware.ts                 # Clerk + WorkspaceContext + rate limiting
├── .env.example
├── CLAUDE.md                     # This file
└── README.md
```

---

## 3. Code Conventions — Non-Negotiable

### TypeScript
- `tsconfig.json` runs in **strict** mode. No `any`, no `@ts-ignore` without an inline `// FIXME(belal):` comment and a ticket.
- Prefer `type` aliases over `interface` unless extending.
- Use `zod` for runtime validation of all external inputs (webhooks, API routes, forms).
- Function names: `verbNoun` (`classifyIntent`, `ingestMessage`, `sendWhatsApp`).
- Files: `kebab-case.ts` for non-component files, `PascalCase.tsx` for components.

### React / Next.js
- **Server Components by default.** Use `'use client'` only when interactivity is needed (forms, hover states, real-time UIs).
- Data fetching in Server Components via async/await directly with Prisma (server-side). No `useEffect` for initial data.
- Mutations via **Server Actions** (App Router) unless we need REST shape.
- Forms via `react-hook-form` + `zod`. Always.
- All input components from `components/ui/` (shadcn). Never roll our own.

### Styling
- Tailwind only. No CSS modules, no styled-components.
- Use the design tokens defined in `tailwind.config.ts` (colors, spacing, radii, font sizes). Don't hardcode.
- Dark mode from Day 1 — Tailwind's `dark:` prefix on every color class.

### Database
- **Every read from a workspace-scoped table MUST include `where: { workspaceId }`.** No exceptions. The RLS policy is the safety net, not the primary defense.
- Use `select` on Prisma queries to fetch only what you need. Avoid `include` on large relations.
- Migrations: `pnpm prisma migrate dev --name <short-description>` — names are kebab-case + verb-noun.
- Indexes documented in `schema.prisma` with `@@index([...])`. Don't add ad-hoc indexes via raw SQL.

### Error handling
- API routes return `{ ok: false, error: { code, message } }` on failure. Never bubble raw exceptions.
- Server Components: throw inside the component for `error.tsx` boundary to catch.
- Never expose Prisma errors to the client. Sanitize.
- **No PII in error messages.** No phone, email, message body in error logs. Hash or redact.

### Logging
- App logs via BetterStack (`@logtail/next`). Use structured logs: `log.info({ workspaceId, action: 'message.ingested' }, 'msg')`.
- Sentry catches exceptions automatically. Tag every exception with `workspaceId` if known.
- Never `console.log` in production code. Use the logger.

### Testing
- **Unit tests** (Vitest) for `lib/` pure functions (no DB, no I/O). Required.
- **Integration tests** for API routes and Inngest functions, using a Neon test branch. Required for any flow touching multiple tables.
- **Playwright e2e** for critical user journeys (Setup Wizard, Inbox → reply, Booking flow). Required before merging to `main`.
- Coverage target: 60% on `lib/`, e2e covers happy paths.

### Comments
- **Don't comment what the code does — comment why it's done a non-obvious way.**
- Public functions / exported types: TSDoc comments with `@param` + `@returns`.

### Git
- Branch naming: `feat/inbox-list`, `fix/wa-webhook-retry`, `chore/upgrade-prisma`.
- Conventional commits in messages. PR titles = the user-facing change.
- Every PR: passes `pnpm typecheck`, `pnpm lint`, `pnpm test`, and a Vercel preview deploy.
- Never push directly to `main`. PRs only.

---

## 4. Multi-Tenancy & Security Rules — Sacred

### The Workspace model
- One **Clerk Organization** = one **Workspace**.
- Every workspace-scoped row has a `workspaceId` column.
- `middleware.ts` resolves `workspaceId` from the Clerk session and attaches it to a request-scoped context.
- Every Prisma query in app code reads `workspaceId` from that context — never from URL params or user input.

### Postgres Row-Level Security (RLS) — defense in depth
- RLS is enabled on every workspace-scoped table.
- The middleware sets `SET app.workspace_id = '<id>'` on every request via Prisma's `$executeRawUnsafe`.
- RLS policy: `USING (workspace_id::text = current_setting('app.workspace_id', true))`
- This means even if a query forgets `where workspaceId`, Postgres still filters it.

### Authentication
- Clerk handles signup, login, session, JWT.
- The middleware reads `auth().orgId` and resolves it to our `workspaceId`.
- No custom session handling, no JWT signing on our side.
- Auth-protected routes: anything under `(app)/`. Public: marketing, auth pages, webhooks, cron.

### Webhooks (Meta, Stripe)
- **Meta**: verify the `X-Hub-Signature-256` header on every POST. Reject if mismatch.
- **Stripe**: verify the `Stripe-Signature` header. Reject if mismatch.
- All webhook payloads logged to `WebhookDelivery` table BEFORE processing. Idempotency key = `(channel, externalId)`. If the row already exists with `processedAt`, return 200 immediately without re-processing.

### Plan-feature gating — server-side
- Define plan features in `lib/plan/features.ts` as a const matrix:
  ```ts
  export const PLAN_FEATURES = {
    STARTER:    { chatbot: false, calendar: false, channels: ['WHATSAPP'], maxUsers: 2,  contactsLimit: 500 },
    GROWTH:     { chatbot: true,  calendar: true,  channels: ['WHATSAPP','INSTAGRAM','MESSENGER'], maxUsers: 5,  contactsLimit: 2000 },
    AGENCY:     { chatbot: true,  calendar: true,  channels: ['WHATSAPP','INSTAGRAM','MESSENGER'], maxUsers: 10, contactsLimit: 5000, whiteLabel: 'option' },
    ENTERPRISE: { chatbot: true,  calendar: true,  channels: ['WHATSAPP','INSTAGRAM','MESSENGER'], maxUsers: Infinity, contactsLimit: Infinity, whiteLabel: 'resale' },
  } as const;
  ```
- Use `requireFeature(workspace, 'chatbot')` middleware on any route that touches a gated feature. Returns 402 with upgrade-link payload if blocked.
- **Server-enforce everything.** UI gating is decoration; backend is truth.

### PII handling
- Never log message bodies, phone numbers, emails, or customer names to Sentry / BetterStack at info level. Use a `redact()` helper.
- Signed URLs for attachments expire in 15 minutes.
- DB backups encrypted (Neon handles this).
- GDPR data-export endpoint required before launch: `GET /api/workspace/export` returns a zip of all the workspace's data.

### Rate limiting
- Webhooks: 100 req/min per source IP via Upstash.
- Auth endpoints: 10 req/min per IP.
- Public booking pages (V1.1): 30 req/min per IP.

---

## 5. Master Prisma Schema

```prisma
// prisma/schema.prisma

generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [pgcrypto]
}

// ─── Workspace & Users ────────────────────────────────────────────

model Workspace {
  id           String   @id @default(cuid())
  clerkOrgId   String   @unique             // 1 Clerk Org = 1 Workspace
  name         String
  vertical     Vertical @default(OTHER)     // clinic | retail | education | auto | realestate | prof_services | travel | other
  timezone     String   @default("UTC")     // IANA, e.g. "Africa/Cairo"
  primaryLang  String   @default("en")      // BCP-47
  setupComplete Boolean @default(false)
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  members        WorkspaceMember[]
  subscription   Subscription?
  contacts       Contact[]
  conversations  Conversation[]
  messages       Message[]
  replies        Reply[]
  tags           Tag[]
  notes          Note[]
  bookingSlots   BookingSlot[]
  appointments   Appointment[]
  rules          Rule[]
  auditLogs      AuditLog[]
  quickReplies   QuickReply[]
  workingHours   WorkingHours[]
  classifications ClassificationLog[]

  @@index([clerkOrgId])
}

model WorkspaceMember {
  id          String    @id @default(cuid())
  workspaceId String
  clerkUserId String
  role        Role      @default(MEMBER)    // OWNER | ADMIN | MEMBER | VIEWER
  invitedAt   DateTime  @default(now())
  acceptedAt  DateTime?

  workspace   Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@unique([workspaceId, clerkUserId])
  @@index([workspaceId])
  @@index([clerkUserId])
}

enum Role { OWNER ADMIN MEMBER VIEWER }
enum Vertical { CLINIC RETAIL EDUCATION AUTO REALESTATE PROF_SERVICES TRAVEL OTHER }

// ─── Subscription & Quota ─────────────────────────────────────────

model Subscription {
  id                   String    @id @default(cuid())
  workspaceId          String    @unique
  stripeCustomerId     String?
  stripeSubscriptionId String?
  plan                 Plan      @default(STARTER)
  status               SubStatus @default(TRIALING)
  trialEndsAt          DateTime?
  currentPeriodStart   DateTime
  currentPeriodEnd     DateTime
  contactsLimit        Int
  contactsUsed         Int       @default(0)
  extraContactPacks    Int       @default(0)
  cancelAtPeriodEnd    Boolean   @default(false)
  updatedAt            DateTime  @updatedAt

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@index([stripeCustomerId])
  @@index([currentPeriodEnd])
}

enum Plan { STARTER GROWTH AGENCY ENTERPRISE }
enum SubStatus { TRIALING ACTIVE PAST_DUE CANCELED INCOMPLETE }

// ─── Contacts & Channel Links ─────────────────────────────────────

model Contact {
  id          String        @id @default(cuid())
  workspaceId String
  name        String?
  phone       String?       // WA primary key
  email       String?
  status      ContactStatus @default(NEW)
  source      String?       // "Instagram Ad", "Walk-in", etc.
  language    String?       // BCP-47
  deletedAt   DateTime?
  createdAt   DateTime      @default(now())
  updatedAt   DateTime      @updatedAt

  workspace      Workspace      @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  channelLinks   ChannelLink[]
  conversations  Conversation[]
  appointments   Appointment[]
  tags           ContactTag[]

  @@index([workspaceId, phone])
  @@index([workspaceId, status])
  @@index([workspaceId, updatedAt(sort: Desc)])
  @@index([workspaceId, deletedAt])
}

enum ContactStatus { NEW ACTIVE BOOKED CHURNED }

model ChannelLink {
  id          String  @id @default(cuid())
  contactId   String
  workspaceId String
  channel     Channel
  externalId  String  // phone for WA, IG handle for IG, page-scoped-id for Messenger
  primary     Boolean @default(false)

  contact Contact @relation(fields: [contactId], references: [id], onDelete: Cascade)

  @@unique([workspaceId, channel, externalId])
  @@index([contactId])
}

enum Channel { WHATSAPP INSTAGRAM MESSENGER TIKTOK }

// ─── Conversations & Messages ─────────────────────────────────────

model Conversation {
  id          String     @id @default(cuid())
  workspaceId String
  contactId   String
  channel     Channel
  status      ConvStatus @default(OPEN)
  lastMessageAt DateTime @default(now())
  unreadCount Int        @default(0)
  assignedToClerkUserId String?
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  contact   Contact   @relation(fields: [contactId], references: [id], onDelete: Cascade)
  messages  Message[]
  notes     Note[]

  @@index([workspaceId, status, lastMessageAt(sort: Desc)])
  @@index([contactId, updatedAt(sort: Desc)])
}

enum ConvStatus { OPEN HANDED_OFF CLOSED }

model Message {
  id             String     @id @default(cuid())
  workspaceId    String
  conversationId String
  direction      Direction
  body           String
  channel        Channel
  intent         String?    // matched intent from classifier
  sentByAI       Boolean    @default(false)
  externalId     String?    // Meta's message id (for outbound dedup)
  sentAt         DateTime   @default(now())

  workspace    Workspace          @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  conversation Conversation       @relation(fields: [conversationId], references: [id], onDelete: Cascade)
  attachments  MessageAttachment[]

  @@index([conversationId, sentAt(sort: Desc)])
  @@index([workspaceId, sentAt(sort: Desc)])
}

enum Direction { INBOUND OUTBOUND }

model MessageAttachment {
  id          String   @id @default(cuid())
  messageId   String
  workspaceId String
  url         String   // R2 signed URL or path
  mimeType    String
  filename    String?
  sizeBytes   Int
  createdAt   DateTime @default(now())

  message Message @relation(fields: [messageId], references: [id], onDelete: Cascade)

  @@index([messageId])
  @@index([workspaceId])
}

// ─── FAQ Library (Fixed-Reply) ────────────────────────────────────

model Reply {
  id          String   @id @default(cuid())
  workspaceId String
  intent      String   // e.g. "ASK_PRICE", "ASK_HOURS", "BOOK", "ASK_LOCATION"
  bodies      Json     // { en: "...", es: "...", ar: "..." } - language-keyed map
  enabled     Boolean  @default(true)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@unique([workspaceId, intent])
  @@index([workspaceId, enabled])
}

// ─── Tags ─────────────────────────────────────────────────────────

model Tag {
  id          String       @id @default(cuid())
  workspaceId String
  label       String
  color       String?      // hex, optional
  createdAt   DateTime     @default(now())

  workspace Workspace    @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  contacts  ContactTag[]

  @@unique([workspaceId, label])
  @@index([workspaceId])
}

model ContactTag {
  contactId String
  tagId     String
  taggedAt  DateTime @default(now())

  contact Contact @relation(fields: [contactId], references: [id], onDelete: Cascade)
  tag     Tag     @relation(fields: [tagId], references: [id], onDelete: Cascade)

  @@id([contactId, tagId])
  @@index([tagId])
}

// ─── Notes ────────────────────────────────────────────────────────

model Note {
  id             String   @id @default(cuid())
  workspaceId    String
  conversationId String
  authorClerkUserId String
  body           String
  createdAt      DateTime @default(now())

  workspace    Workspace    @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  conversation Conversation @relation(fields: [conversationId], references: [id], onDelete: Cascade)

  @@index([conversationId, createdAt(sort: Desc)])
}

// ─── Quick Replies (canned responses) ─────────────────────────────

model QuickReply {
  id          String   @id @default(cuid())
  workspaceId String
  shortcut    String   // e.g. "/price"
  body        String
  createdAt   DateTime @default(now())

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@unique([workspaceId, shortcut])
  @@index([workspaceId])
}

// ─── Working Hours ────────────────────────────────────────────────

model WorkingHours {
  id          String   @id @default(cuid())
  workspaceId String
  weekday     Int      // 0 = Sun, 6 = Sat
  openTime    String   // "09:00"
  closeTime   String   // "18:00"
  enabled     Boolean  @default(true)

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@unique([workspaceId, weekday])
  @@index([workspaceId])
}

// ─── Calendar & Booking ───────────────────────────────────────────

model BookingSlot {
  id          String     @id @default(cuid())
  workspaceId String
  startAt     DateTime   // stored in UTC
  endAt       DateTime
  status      SlotStatus @default(AVAILABLE)
  createdAt   DateTime   @default(now())

  workspace    Workspace     @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  appointments Appointment[]

  @@index([workspaceId, status, startAt])
  @@index([workspaceId, startAt])
}

enum SlotStatus { AVAILABLE BOOKED BLOCKED }

model Appointment {
  id           String     @id @default(cuid())
  workspaceId  String
  contactId    String
  slotId       String
  status       ApptStatus @default(CONFIRMED)
  reminderSent Boolean    @default(false)
  notes        String?
  createdAt    DateTime   @default(now())
  updatedAt    DateTime   @updatedAt

  workspace Workspace    @relation(fields: [workspaceId], references: [id], onDelete: Cascade)
  contact   Contact      @relation(fields: [contactId], references: [id], onDelete: Cascade)
  slot      BookingSlot  @relation(fields: [slotId], references: [id])

  @@index([workspaceId, status])
  @@index([contactId])
  @@index([slotId])
  @@index([reminderSent, status])    // for reminders cron
}

enum ApptStatus { CONFIRMED RESCHEDULED NO_SHOW COMPLETED CANCELED }

// ─── Custom Follow-Up Rules ───────────────────────────────────────

model Rule {
  id          String    @id @default(cuid())
  workspaceId String
  name        String
  trigger     Json      // { type: "NO_BOOKING_WITHIN", hours: 24, intent: "ASK_PRICE" }
  delayMinutes Int      @default(0)
  replyId     String?   // FK to Reply (the message to send)
  enabled     Boolean   @default(true)
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@index([workspaceId, enabled])
}

// ─── Audit Log ────────────────────────────────────────────────────

model AuditLog {
  id          String   @id @default(cuid())
  workspaceId String
  actorClerkUserId String?    // null for system
  action      String          // "appointment.created", "rule.toggled", etc.
  targetType  String?         // "Contact", "Appointment", ...
  targetId    String?
  metadata    Json?
  ipAddress   String?
  userAgent   String?
  createdAt   DateTime @default(now())

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@index([workspaceId, createdAt(sort: Desc)])
  @@index([targetType, targetId])
}

// ─── Webhook Idempotency ──────────────────────────────────────────

model WebhookDelivery {
  id           String    @id @default(cuid())
  channel      Channel
  externalId   String    // Meta message id / Stripe event id
  payload      Json
  receivedAt   DateTime  @default(now())
  processedAt  DateTime?
  errorMessage String?
  retryCount   Int       @default(0)

  @@unique([channel, externalId])
  @@index([processedAt])
}

// ─── AI Classification Log (cost + quality tracking) ──────────────

model ClassificationLog {
  id           String   @id @default(cuid())
  workspaceId  String
  messageId    String
  intent       String?
  confidence   String   // "high" | "low" | "preclassified"
  promptTokens Int      @default(0)
  outputTokens Int      @default(0)
  cachedHit    Boolean  @default(false)
  createdAt    DateTime @default(now())

  workspace Workspace @relation(fields: [workspaceId], references: [id], onDelete: Cascade)

  @@index([workspaceId, createdAt(sort: Desc)])
}
```

**After applying:** run the RLS-enabling SQL script in `prisma/migrations/_post/enable-rls.sql` (Belal writes this Week 1).

---

## 6. Environment Variables

```bash
# Database
DATABASE_URL="postgresql://..."             # Neon connection string (with pooler)
DIRECT_URL="postgresql://..."               # Neon direct (for migrations)

# Auth (Clerk)
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="pk_..."
CLERK_SECRET_KEY="sk_..."
CLERK_WEBHOOK_SECRET="whsec_..."             # for user/org webhooks → sync to WorkspaceMember

# AI
ANTHROPIC_API_KEY="sk-ant-..."
CLAUDE_MODEL="claude-sonnet-4-6"

# Meta (WhatsApp + Instagram + Messenger)
META_APP_ID="..."
META_APP_SECRET="..."                        # for webhook signature verification
META_VERIFY_TOKEN="<we-pick-a-random-string>"

# Email
RESEND_API_KEY="re_..."
RESEND_FROM="Cowork <hello@cowork.app>"

# Storage
R2_ACCESS_KEY_ID="..."
R2_SECRET_ACCESS_KEY="..."
R2_BUCKET="cowork-attachments"
R2_ACCOUNT_ID="..."
R2_PUBLIC_URL="https://files.cowork.app"     # custom domain pointing at R2

# Payments
STRIPE_SECRET_KEY="sk_..."
STRIPE_WEBHOOK_SECRET="whsec_..."
STRIPE_PRICE_STARTER_MONTHLY="price_..."
STRIPE_PRICE_GROWTH_MONTHLY="price_..."
STRIPE_PRICE_AGENCY_MONTHLY="price_..."
STRIPE_PRICE_STARTER_YEARLY="price_..."
# ... etc

# Inngest
INNGEST_EVENT_KEY="..."
INNGEST_SIGNING_KEY="..."

# Rate limiting
UPSTASH_REDIS_REST_URL="https://..."
UPSTASH_REDIS_REST_TOKEN="..."

# Observability
NEXT_PUBLIC_POSTHOG_KEY="phc_..."
NEXT_PUBLIC_POSTHOG_HOST="https://app.posthog.com"
SENTRY_DSN="https://..."
LOGTAIL_SOURCE_TOKEN="..."

# Cron
CRON_SECRET="<we-pick-a-random-string>"
```

Copy `.env.example` with placeholders into the repo. Never commit `.env`.

---

## 7. Key Code Patterns — Copy These Exactly

### 7.1 Workspace Context middleware

```typescript
// middleware.ts
import { authMiddleware } from '@clerk/nextjs';
import { NextResponse } from 'next/server';

export default authMiddleware({
  publicRoutes: [
    '/',
    '/pricing',
    '/login(.*)',
    '/signup(.*)',
    '/api/webhooks/(.*)',
    '/api/cron/(.*)',
    '/api/health',
  ],
  async afterAuth(auth, req) {
    if (!auth.userId) return NextResponse.next();

    // Resolve workspaceId from Clerk Organization
    const workspaceId = await resolveWorkspaceId(auth.orgId);
    if (!workspaceId && !req.nextUrl.pathname.startsWith('/setup')) {
      return NextResponse.redirect(new URL('/setup', req.url));
    }

    const res = NextResponse.next();
    if (workspaceId) {
      res.headers.set('x-workspace-id', workspaceId);
    }
    return res;
  },
});

export const config = {
  matcher: ['/((?!.*\\..*|_next).*)', '/', '/(api|trpc)(.*)'],
};
```

```typescript
// lib/auth.ts
import { auth } from '@clerk/nextjs/server';
import { db } from './db';

export async function getWorkspaceContext() {
  const { orgId, userId } = auth();
  if (!orgId || !userId) throw new UnauthorizedError();

  const workspace = await db.workspace.findUnique({
    where: { clerkOrgId: orgId },
    include: { subscription: true, members: { where: { clerkUserId: userId } } },
  });
  if (!workspace) throw new NotFoundError('workspace');
  if (!workspace.members.length) throw new ForbiddenError();

  return {
    workspace,
    userId,
    role: workspace.members[0].role,
    subscription: workspace.subscription,
  };
}
```

### 7.2 Channel Connector Interface

```typescript
// lib/channels/types.ts
import { Channel } from '@prisma/client';

export type InboundMessage = {
  channel: Channel;
  externalId: string;     // Meta msg id — idempotency key
  fromExternalId: string; // phone for WA, IG handle for IG
  fromName?: string;
  body: string;
  attachments: { url: string; mimeType: string }[];
  timestamp: Date;
  workspacePhoneId?: string; // which of our WABA numbers received it
};

export interface ChannelConnector {
  readonly channel: Channel;
  verifyWebhookSignature(rawBody: string, signature: string): boolean;
  parseInbound(payload: unknown): InboundMessage[];
  send(args: {
    workspaceId: string;
    to: string;           // external recipient id
    body: string;
    replyToMessageId?: string;
  }): Promise<{ externalId: string }>;
}
```

```typescript
// lib/channels/whatsapp.ts (skeleton — Claude Code fills in)
import crypto from 'crypto';
import type { ChannelConnector, InboundMessage } from './types';

export const whatsappConnector: ChannelConnector = {
  channel: 'WHATSAPP',

  verifyWebhookSignature(rawBody, signature) {
    const expected = crypto
      .createHmac('sha256', process.env.META_APP_SECRET!)
      .update(rawBody)
      .digest('hex');
    return crypto.timingSafeEqual(
      Buffer.from(`sha256=${expected}`),
      Buffer.from(signature),
    );
  },

  parseInbound(payload: any): InboundMessage[] {
    const out: InboundMessage[] = [];
    for (const entry of payload?.entry ?? []) {
      for (const change of entry.changes ?? []) {
        for (const msg of change?.value?.messages ?? []) {
          out.push({
            channel: 'WHATSAPP',
            externalId: msg.id,
            fromExternalId: msg.from,
            fromName: change.value?.contacts?.[0]?.profile?.name,
            body: msg.text?.body ?? '',
            attachments: [], // TODO Week 2: handle image/media
            timestamp: new Date(Number(msg.timestamp) * 1000),
            workspacePhoneId: change.value?.metadata?.phone_number_id,
          });
        }
      }
    }
    return out;
  },

  async send({ workspaceId, to, body }) {
    // Resolve workspace's WA access token + phone-number-id from DB
    const config = await getWhatsAppConfig(workspaceId);
    const res = await fetch(
      `https://graph.facebook.com/v20.0/${config.phoneNumberId}/messages`,
      {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${config.accessToken}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          messaging_product: 'whatsapp',
          to,
          text: { body },
        }),
      },
    );
    if (!res.ok) throw new ChannelSendError('WA send failed', await res.text());
    const data = await res.json();
    return { externalId: data.messages[0].id };
  },
};
```

### 7.3 Webhook route with idempotency

```typescript
// app/api/webhooks/whatsapp/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { whatsappConnector } from '@/lib/channels/whatsapp';
import { db } from '@/lib/db';
import { inngest } from '@/inngest/client';

export async function GET(req: NextRequest) {
  const p = req.nextUrl.searchParams;
  if (
    p.get('hub.mode') === 'subscribe' &&
    p.get('hub.verify_token') === process.env.META_VERIFY_TOKEN
  ) {
    return new NextResponse(p.get('hub.challenge'), { status: 200 });
  }
  return new NextResponse('forbidden', { status: 403 });
}

export async function POST(req: NextRequest) {
  const rawBody = await req.text();
  const signature = req.headers.get('x-hub-signature-256') ?? '';

  if (!whatsappConnector.verifyWebhookSignature(rawBody, signature)) {
    return new NextResponse('invalid signature', { status: 401 });
  }

  const payload = JSON.parse(rawBody);
  const inbounds = whatsappConnector.parseInbound(payload);

  for (const msg of inbounds) {
    // Idempotency: have we seen this externalId before?
    try {
      await db.webhookDelivery.create({
        data: {
          channel: 'WHATSAPP',
          externalId: msg.externalId,
          payload: msg as any,
        },
      });
    } catch (e: any) {
      // Unique constraint violation = already received. Skip.
      if (e.code === 'P2002') continue;
      throw e;
    }

    // Fire async ingestion via Inngest (don't block the webhook response)
    await inngest.send({
      name: 'message.inbound',
      data: { channel: 'WHATSAPP', externalId: msg.externalId },
    });
  }

  return NextResponse.json({ ok: true });
}
```

### 7.4 Intent Classifier (with cheap pre-classifier)

```typescript
// lib/ai/preClassifier.ts
const KEYWORD_INTENTS: Record<string, string[]> = {
  ASK_HOURS:    ['hours', 'open', 'closed', 'when do you', 'opening', 'closing'],
  ASK_LOCATION: ['where', 'address', 'location', 'directions', 'map'],
  ASK_PRICE:    ['price', 'cost', 'how much', 'pricing', 'rates', 'fee'],
  BOOK:         ['book', 'appointment', 'schedule', 'available', 'slot', 'time'],
  HANDOFF:      ['human', 'real person', 'agent', 'speak to someone'],
};

export function preClassify(message: string): string | null {
  const lower = message.toLowerCase();
  for (const [intent, keywords] of Object.entries(KEYWORD_INTENTS)) {
    if (keywords.some(kw => lower.includes(kw))) return intent;
  }
  return null; // unknown → fall through to Claude
}
```

```typescript
// lib/ai/classifyIntent.ts
import Anthropic from '@anthropic-ai/sdk';
import { preClassify } from './preClassifier';
import { checkCostCap } from './costGuard';
import { db } from '@/lib/db';

const claude = new Anthropic();

export async function classifyIntent(args: {
  workspaceId: string;
  messageId: string;
  body: string;
  availableIntents: string[];
}): Promise<{ intent: string; confidence: 'high' | 'low' | 'preclassified' }> {

  // Layer 1: cheap pre-classifier
  const pre = preClassify(args.body);
  if (pre && args.availableIntents.includes(pre)) {
    await logClassification({
      ...args, intent: pre, confidence: 'preclassified', cachedHit: true,
    });
    return { intent: pre, confidence: 'preclassified' };
  }

  // Layer 2: cost-cap gate
  if (!(await checkCostCap(args.workspaceId))) {
    await logClassification({
      ...args, intent: 'HANDOFF', confidence: 'low', cachedHit: false,
    });
    return { intent: 'HANDOFF', confidence: 'low' };
  }

  // Layer 3: Claude
  const result = await claude.messages.create({
    model: process.env.CLAUDE_MODEL ?? 'claude-sonnet-4-6',
    max_tokens: 64,
    system: `You classify customer DMs into ONE of these intents: ${args.availableIntents.join(', ')}.
Reply with ONLY the intent name, in capitals, no punctuation.
If none of the intents match clearly, reply: HANDOFF.
The message may be in any language.`,
    messages: [{ role: 'user', content: args.body }],
  });

  const text = result.content[0].type === 'text'
    ? result.content[0].text.trim().toUpperCase()
    : 'HANDOFF';
  const matched = args.availableIntents.includes(text) ? text : 'HANDOFF';

  await logClassification({
    workspaceId: args.workspaceId,
    messageId: args.messageId,
    intent: matched,
    confidence: matched === 'HANDOFF' ? 'low' : 'high',
    cachedHit: false,
    promptTokens: result.usage.input_tokens,
    outputTokens: result.usage.output_tokens,
  });

  return {
    intent: matched,
    confidence: matched === 'HANDOFF' ? 'low' : 'high',
  };
}

async function logClassification(d: {
  workspaceId: string; messageId: string; intent: string;
  confidence: 'high' | 'low' | 'preclassified'; cachedHit: boolean;
  promptTokens?: number; outputTokens?: number;
}) {
  await db.classificationLog.create({ data: {
    workspaceId: d.workspaceId,
    messageId: d.messageId,
    intent: d.intent,
    confidence: d.confidence,
    cachedHit: d.cachedHit,
    promptTokens: d.promptTokens ?? 0,
    outputTokens: d.outputTokens ?? 0,
  }});
}
```

### 7.5 Plan-feature gate

```typescript
// lib/plan/features.ts
export const PLAN_FEATURES = {
  STARTER: {
    chatbot: false,
    calendar: false,
    channels: ['WHATSAPP'] as const,
    maxUsers: 2,
    contactsLimit: 500,
    whiteLabel: 'none' as const,
  },
  GROWTH: {
    chatbot: true,
    calendar: true,
    channels: ['WHATSAPP', 'INSTAGRAM', 'MESSENGER'] as const,
    maxUsers: 5,
    contactsLimit: 2000,
    whiteLabel: 'none' as const,
  },
  AGENCY: {
    chatbot: true,
    calendar: true,
    channels: ['WHATSAPP', 'INSTAGRAM', 'MESSENGER'] as const,
    maxUsers: 10,
    contactsLimit: 5000,
    whiteLabel: 'option' as const,
  },
  ENTERPRISE: {
    chatbot: true,
    calendar: true,
    channels: ['WHATSAPP', 'INSTAGRAM', 'MESSENGER'] as const,
    maxUsers: Infinity,
    contactsLimit: Infinity,
    whiteLabel: 'resale' as const,
  },
} as const;

export type FeatureName = keyof (typeof PLAN_FEATURES)['STARTER'];

export function getFeatures(plan: keyof typeof PLAN_FEATURES) {
  return PLAN_FEATURES[plan];
}
```

```typescript
// lib/plan/gate.ts
import { NextResponse } from 'next/server';
import { getWorkspaceContext } from '@/lib/auth';
import { PLAN_FEATURES, type FeatureName } from './features';

export async function requireFeature(feature: FeatureName) {
  const { workspace, subscription } = await getWorkspaceContext();
  const plan = subscription?.plan ?? 'STARTER';
  const features = PLAN_FEATURES[plan];

  if (!features[feature]) {
    throw new PlanGateError(plan, feature);
  }
  return { workspace, plan, features };
}

export class PlanGateError extends Error {
  constructor(public plan: string, public feature: FeatureName) {
    super(`Feature '${feature}' not available on plan '${plan}'`);
  }
}

// Use in API route:
// try { await requireFeature('chatbot'); } catch (e) {
//   if (e instanceof PlanGateError) return NextResponse.json({
//     ok: false, error: { code: 'PLAN_GATE', plan: e.plan, feature: e.feature, upgradeUrl: '/billing' }
//   }, { status: 402 });
//   throw e;
// }
```

### 7.6 Audit logger

```typescript
// lib/audit/log.ts
import { db } from '@/lib/db';
import { auth } from '@clerk/nextjs/server';
import { headers } from 'next/headers';

export async function audit(args: {
  workspaceId: string;
  action: string;
  targetType?: string;
  targetId?: string;
  metadata?: Record<string, unknown>;
}) {
  const { userId } = auth();
  const h = headers();
  await db.auditLog.create({
    data: {
      workspaceId: args.workspaceId,
      actorClerkUserId: userId ?? null,
      action: args.action,
      targetType: args.targetType,
      targetId: args.targetId,
      metadata: args.metadata as any,
      ipAddress: h.get('x-forwarded-for') ?? null,
      userAgent: h.get('user-agent') ?? null,
    },
  });
}
```

### 7.7 Inngest reminder cron

```typescript
// inngest/functions/send-reminders.ts
import { inngest } from '../client';
import { db } from '@/lib/db';
import { sendWhatsApp } from '@/lib/channels/whatsapp';

export const sendReminders = inngest.createFunction(
  { id: 'send-reminders' },
  { cron: '*/10 * * * *' }, // every 10 min
  async ({ step }) => {
    const due = await step.run('find-due', () =>
      db.appointment.findMany({
        where: {
          reminderSent: false,
          status: 'CONFIRMED',
          slot: { startAt: { lte: new Date(Date.now() + 24 * 60 * 60 * 1000) } },
        },
        include: { contact: true, slot: true, workspace: true },
      }),
    );

    for (const appt of due) {
      await step.run(`send-${appt.id}`, async () => {
        await sendWhatsApp({
          workspaceId: appt.workspaceId,
          to: appt.contact.phone!,
          body: `Reminder: your appointment at ${appt.workspace.name} is on ${formatDate(appt.slot.startAt, appt.workspace.timezone)}.`,
        });
        await db.appointment.update({
          where: { id: appt.id },
          data: { reminderSent: true },
        });
        await db.auditLog.create({ data: {
          workspaceId: appt.workspaceId,
          action: 'reminder.sent',
          targetType: 'Appointment',
          targetId: appt.id,
        }});
      });
    }
    return { sentCount: due.length };
  },
);
```

### 7.8 Server Action (mutation pattern)

```typescript
// app/(app)/inbox/[conversationId]/_actions.ts
'use server';

import { z } from 'zod';
import { getWorkspaceContext } from '@/lib/auth';
import { db } from '@/lib/db';
import { sendMessage } from '@/lib/outbound/sendMessage';
import { audit } from '@/lib/audit/log';
import { revalidatePath } from 'next/cache';

const SendReplySchema = z.object({
  conversationId: z.string(),
  body: z.string().min(1).max(4096),
});

export async function sendReply(input: z.infer<typeof SendReplySchema>) {
  const parsed = SendReplySchema.parse(input);
  const { workspace, userId } = await getWorkspaceContext();

  const conv = await db.conversation.findFirst({
    where: { id: parsed.conversationId, workspaceId: workspace.id },
    include: { contact: { include: { channelLinks: true } } },
  });
  if (!conv) throw new Error('Conversation not found');

  await sendMessage({
    workspaceId: workspace.id,
    conversationId: conv.id,
    channel: conv.channel,
    to: conv.contact.channelLinks.find(c => c.channel === conv.channel)!.externalId,
    body: parsed.body,
    sentByAI: false,
  });

  await audit({
    workspaceId: workspace.id,
    action: 'message.sent',
    targetType: 'Conversation',
    targetId: conv.id,
  });

  revalidatePath(`/inbox/${parsed.conversationId}`);
}
```

---

## 8. The 11-Week Execution Plan

Each week below has: **Objectives → Deliverables → Acceptance Criteria → Testing → Risks**.

When starting Week N: paste **Section 0–7 of this doc** as the system prompt, then paste the Week N block from §9 below as the task prompt.

---

## 9. Per-Week Detailed Prompts

### Week 1 — Foundations (Jun 15–21)

**Objectives**
1. File all Meta + TikTok approvals (Belal does this manually on Day 1 morning — not your task, but assume credentials will be available by Week 3).
2. Bootstrap the repo with the locked stack.
3. Apply the full Prisma schema (§5) to Neon.
4. Enable Postgres RLS on all workspace-scoped tables.
5. Wire Clerk auth + Organizations → Workspace mapping.

**Deliverables**
- `cw-platform/` repo created via `npx create-next-app@latest --typescript --tailwind --app --src-dir=false`.
- `pnpm` workspace, locked file committed.
- shadcn-init done. Components installed: `button, input, textarea, select, dialog, sheet, dropdown-menu, tabs, toast, card, avatar, badge, skeleton, tooltip`.
- `prisma init` + the full schema from §5 applied via `pnpm prisma migrate dev --name init`.
- `prisma/migrations/_post/enable-rls.sql` script written and applied (RLS policies for every workspace-scoped table).
- Clerk integration: signup → on-create Clerk Org → on-org-create `Workspace` + `WorkspaceMember(role: OWNER)` records via a Clerk webhook handler at `/api/webhooks/clerk/route.ts`.
- `middleware.ts` resolving workspaceId per request and injecting via header + Postgres `SET app.workspace_id`.
- `lib/auth.ts` with `getWorkspaceContext()`.
- `.env.example` with every variable from §6.
- A `/` (marketing) and `/dashboard` (auth-gated, empty for now) route.
- README explaining setup steps.

**Acceptance Criteria**
- ✅ A fresh signup → Clerk Org created → Workspace row exists → user lands on `/dashboard` showing "Welcome, {workspace.name}".
- ✅ Two test workspaces created (A and B). User from A runs `SELECT * FROM contact` via Prisma → returns only A's contacts. Even with raw SQL `WHERE workspace_id = 'b'` it returns empty (RLS active).
- ✅ `pnpm typecheck` clean. `pnpm lint` clean. `pnpm test` (placeholder) passes.
- ✅ First Vercel preview deploys successfully.

**Testing**
- Vitest unit test: `lib/auth.ts` resolves the right Workspace given a Clerk org id.
- Integration test: RLS isolation between 2 workspaces.
- Manual smoke: signup → dashboard end-to-end.

**Risks**
- Clerk Organizations webhook delivery delay → user lands on /dashboard before Workspace exists. Mitigation: middleware redirects to a "Setting up your workspace…" splash that polls until ready (max 10s).
- RLS misconfig → no data visible. Mitigation: clear test in `tests/integration/rls.test.ts`.

---

### Week 2 — WhatsApp End-to-End + Idempotency + Inbound Pipeline (Jun 22–28)

**Objectives**
1. Real WA messages flow into the DB.
2. Outbound WA messages send via Inngest (with retry).
3. Webhook idempotency in place.

**Deliverables**
- `lib/channels/types.ts` (ChannelConnector interface from §7.2).
- `lib/channels/whatsapp.ts` (full implementation per §7.2 + §7.3 skeleton).
- `app/api/webhooks/whatsapp/route.ts` (GET verify + POST ingest, idempotent via WebhookDelivery, dispatches Inngest event).
- `inngest/functions/process-inbound.ts` — handles the `message.inbound` event, calls `ingestMessage()`.
- `lib/ingest/ingestMessage.ts` — find-or-create Contact + ChannelLink, find-or-create Conversation, insert Message, increment Conversation.unreadCount + update lastMessageAt.
- `lib/outbound/sendMessage.ts` — wraps the connector's `send()` in an Inngest function with 3 retries + exponential backoff + dead-letter.
- A Workspace settings page that lets the owner paste their WA `accessToken`, `phoneNumberId`, `businessAccountId` (encrypted at rest using `pgcrypto`).
- BetterStack logger setup.

**Acceptance Criteria**
- ✅ Connect a real WA sandbox / test number → send "hi" from a real phone → Message row appears in DB tagged to a new Contact + Conversation.
- ✅ Same webhook POST replayed (curl with same payload) → no duplicate Message row.
- ✅ Trigger an outbound send → message delivered to the test phone. Force a 5xx with a mocked Meta response → Inngest retries 3× → moves to dead-letter on 4th failure.
- ✅ Sentry / BetterStack catch a thrown error with no PII in the error message.

**Testing**
- Integration test: post a real WA webhook payload sample → assert Contact + Conversation + Message created exactly once.
- Integration test: idempotency — same payload posted 3× → 1 Message row.
- Integration test: outbound send with mocked 503 → 3 retries logged in Inngest history.

**Risks**
- Meta WA approval not landed → use sandbox test phone Meta provides.
- ngrok / Vercel preview URL mismatch with the webhook URL Meta has → document the redeploy-and-re-register step in README.

---

### Week 3 — Unified Inbox UI + Real-time + Channel Connectors (Jun 29–Jul 5)

**Objectives**
1. The defining screen of the product works.
2. SWR polling makes the inbox feel live.
3. IG + Messenger connectors plug in via the interface.

**Deliverables**
- `/app/(app)/inbox/page.tsx` — three-pane layout: filters rail (left, ~80px) · conversation list (~360px) · open conversation (rest). Mobile: single-pane stack with drill-down.
- `components/inbox/ConversationListItem.tsx` — Contact name + latest snippet + channel icon(s) + unread badge + relative time.
- `components/inbox/MessageBubble.tsx` — channel-aware bubble (color/border per channel), avatar, sentAt, AI badge if `sentByAI`.
- `components/inbox/ReplyBox.tsx` — textarea, send button, attachment placeholder (V1.1: real attachments), quick-reply autocomplete (Week 8: full).
- `components/inbox/ContactCard.tsx` — right rail, collapsible.
- SWR polling: open conversation re-fetches messages every 5s.
- `lib/channels/instagram.ts` + `lib/channels/messenger.ts` (skeletons that follow the same interface, behind feature flag if Meta approval pending).
- `app/api/webhooks/instagram/route.ts` + `messenger/route.ts` — same idempotency pattern.
- `lib/channels/registry.ts` — `Map<Channel, ChannelConnector>` central dispatch.

**Acceptance Criteria**
- ✅ Real DM on WA → conversation list updates within 5s with no F5.
- ✅ Click conversation → thread loads with skeleton → real messages appear → reply box focused.
- ✅ Send a reply → optimistic UI update → confirmed when delivery completes.
- ✅ If IG approval is live: same contact DMs from IG → appears as additional ChannelLink on the same Contact (matched by `fromName` + manual merge UI in V1.1).
- ✅ Inbox renders cleanly on mobile (375px width) with drill-down navigation.

**Testing**
- Playwright e2e: signup → connect WA → simulate inbound (DB seed) → open inbox → reply → message in DB.
- Component test: ConversationListItem renders unread badge correctly.

**Risks**
- IG / Messenger App Review still pending → ship WA-only with feature flag for the other channels.
- Polling load on Neon → cap concurrent polls per workspace at 1 (debounce on focus change).

---

### Week 4 — AI Chatbot Engine (Jul 6–12)

**Objectives**
1. The bot replies on its own to inbound messages.
2. Cost-cap and pre-classifier in place.

**Deliverables**
- `lib/ai/preClassifier.ts` (§7.4).
- `lib/ai/classifyIntent.ts` (§7.4).
- `lib/ai/costGuard.ts` — `checkCostCap(workspaceId)` returns true if we're within the workspace's monthly LLM-call budget. Reads from `ClassificationLog` aggregation.
- `inngest/functions/process-inbound.ts` updated: after `ingestMessage()`, fire intent classification → if intent matched + chatbot feature enabled → send the matching `Reply` via the connector. If HANDOFF → mark Conversation `HANDED_OFF`, notify the workspace.
- Reply selection logic: pick the `Reply.bodies[workspace.primaryLang]` if exists, else `bodies.en`, else first available.
- Handoff notification: in-app badge on the Conversation + optional email to all workspace OWNERs (configurable).

**Acceptance Criteria**
- ✅ Test workspace with 10 FAQ replies → send 10 inbound test messages → at least 8 get the correct auto-reply, 2 trigger HANDOFF.
- ✅ Pre-classifier catches "what are your hours" without hitting Claude (ClassificationLog row with `cachedHit: true`).
- ✅ Cost-cap triggered (mock by setting cap to 1) → next message marked HANDOFF, no Claude call.
- ✅ ClassificationLog populated with token counts for every Claude call.

**Testing**
- Eval set: 50 hand-curated test messages × 5 verticals → assert >= 85% correct intent matching.
- Integration: full inbound → classify → send-reply happy path.
- Cost-guard unit test.

**Risks**
- Claude misclassifies subtle intents. Mitigation: HANDOFF threshold + Week 8 surfaces a "Bot got this wrong" button for quick library refinement.

---

### Week 5 — Setup Wizard + FAQ Library Editor + Vertical Presets (Jul 13–19)

**THE highest-leverage week of V1. Spend all of it here.**

**Objectives**
1. Non-technical owner can go from signup → live bot in under 20 minutes.
2. Vertical presets do the heavy lifting on FAQ content.

**Deliverables**
- `content/verticals/` — 7 preset libraries (clinic, retail, education, auto, realestate, prof-services, travel). Each exports:
  ```ts
  export const clinicPreset = {
    workingHoursDefault: [...],
    services: [{ name: 'Consultation', durationMinutes: 30, priceFrom: '...' }, ...],
    suggestedTags: ['VIP', 'IG Lead', 'Returning'],
    faqs: [
      { intent: 'ASK_PRICE',    bodies: { en: '...', es: '...' } },
      { intent: 'ASK_HOURS',    bodies: { en: '...' } },
      { intent: 'ASK_LOCATION', bodies: { en: '...' } },
      { intent: 'BOOK',         bodies: { en: 'Sure! Here are our next available slots:' } },
      // ... 12–15 entries total
    ],
  } as const;
  ```
- `/app/setup/page.tsx` — multi-step Setup Wizard. 5 steps:
  1. Business name + vertical + primary language
  2. Working hours (defaults autoloaded from vertical preset, editable)
  3. Services (autoloaded, editable; add/remove rows)
  4. FAQ Library review (12–15 pre-filled replies; owner can edit each in-place)
  5. Connect WhatsApp (paste credentials OR "I'll do this later" → defer to /settings)
- Progress bar across the top. "Skip step" + "Use defaults" buttons on every step.
- On final submit: persist everything, set `Workspace.setupComplete = true`, redirect to /inbox with the demo conversation seeded.
- FAQ Library editor at `/app/(app)/chatbot/page.tsx` — list view of all replies, click to edit each.

**Acceptance Criteria**
- ✅ A non-engineer (Mahmoud) completes the wizard end-to-end for a fake clinic in **under 20 minutes** with zero help.
- ✅ All 5 steps validate via Zod. Server-side persistence is atomic (all-or-nothing).
- ✅ Wizard re-entered later from Settings; values pre-populated.
- ✅ A seeded demo conversation is visible in the new Workspace's inbox on first visit.

**Testing**
- Playwright e2e: walk through all 5 steps, assert Workspace.setupComplete + 12+ Reply rows + WorkingHours rows + services persisted.
- User-test session with 1 friendly business before end of week.

**Risks**
- Owner gets bored mid-wizard. Mitigation: defaults everywhere, can finish in 2 minutes if you trust the presets.

---

### Week 6 — Calendar + Booking + Reminders (Jul 20–26)

**Objectives**
1. End-to-end booking works via WA interactive list.
2. Reminders fire on schedule.
3. (Belal in parallel: Bizee LLC filed Jul 15; Stripe + Mercury accounts opened.)

**Deliverables**
- Working-hours editor at `/app/(app)/settings/hours` (also editable from Setup Wizard).
- Slot generator: nightly Inngest function generates AVAILABLE BookingSlot rows for the next 30 days based on WorkingHours.
- BOOK intent handler in `process-inbound`: when matched, query free slots → send WA interactive-list message with next 5 slots → wait for selection.
- WA interactive reply handler: parse the slot pick → create Appointment + mark slot `BOOKED` → send confirmation message → send email to workspace owner via Resend.
- Calendar UI at `/app/(app)/calendar` — week view with appointments overlaid on working hours. Click appointment → edit/cancel modal.
- Reminder cron (`inngest/functions/send-reminders.ts` from §7.7) — fires hourly, sends pre-appointment reminders.
- Timezone-aware throughout: all DB stores UTC; all UI renders in `workspace.timezone`.

**Acceptance Criteria**
- ✅ End-to-end: DM "I want to book" → bot replies with interactive list of 5 slots → tap one → appointment created → confirmation message arrives → owner email arrives via Resend → reminder fires X hours before (default 24h).
- ✅ Cancel appointment from staff UI → slot returns to AVAILABLE.
- ✅ Reminder doesn't fire twice for the same appointment (reminderSent flag).
- ✅ Timezone: a Cairo workspace and a Toronto workspace each see appointments in their local time.

**Testing**
- Playwright e2e: full booking flow with seeded WA inbound.
- Integration test: slot generator creates the right number of slots per week.
- Integration test: reminder cron processes due appointments only once.

**Risks**
- WA interactive-list message format quirks (Meta requires specific JSON shape). Mitigation: develop against Meta's documentation; have a fallback to plain-text "Reply 1, 2, 3, 4, or 5".

---

### Week 7 — Mini CRM + Tags + Auto-tagging + Custom Follow-Up Rules (Jul 27–Aug 2)

**Objectives**
1. The CRM pillar is real.
2. Custom Follow-Up Rules engine works.

**Deliverables**
- `/app/(app)/contacts/page.tsx` — paginated, sortable, searchable Contacts list.
- `/app/(app)/contacts/[id]/page.tsx` — Contact detail with header (name, channels, tags, status) and tabs: Timeline (all messages + appointments + tag changes + rule firings, chronological) · Notes · Appointments.
- Tag editor inline on Contact detail (add/remove tags).
- Auto-tag rules: when a Contact is created via IG webhook → auto-tag "IG-Lead". When source = "Instagram Ad" → tag "Paid-IG-Lead". Configurable later (V1.1); hardcode 3–5 rules in V1.
- Rule Builder UI at `/app/(app)/automations/page.tsx`:
  - Trigger picker: 5 types (NO_BOOKING_WITHIN, INTENT_MATCHED, APPT_NO_SHOW, TAG_ADDED, MESSAGE_COUNT_REACHED)
  - Parameters per trigger (hours, intent name, tag name, count)
  - Delay (minutes)
  - Reply (picked from FAQ Library)
  - Enable / disable toggle
- `inngest/functions/evaluate-rules.ts` — runs every 15 min. For each enabled Rule across all workspaces:
  - Find conversations matching the trigger
  - Filter out conversations that already received this rule's reply (audit log check)
  - Send the reply via the connector
  - Log to AuditLog: `rule.fired`

**Acceptance Criteria**
- ✅ Set up rule "if customer asked price and didn't book within 24h, send win-back reply X" → manually create test conversation matching the trigger → rule fires within 16 minutes.
- ✅ Contact detail page shows all messages across all channels, all appointments, all tag changes, all rule firings.
- ✅ Inline tag editor: add a tag → reflects in DB and other Contacts views without F5.

**Testing**
- Integration: rule trigger evaluator. Mock the conversation state, assert the rule fires (or doesn't) correctly.
- Playwright e2e: Contact detail page navigation.

**Risks**
- Rule evaluation N+1 query problem if many enabled rules × many conversations. Mitigation: paginate by workspace; index `Rule(workspaceId, enabled)` and `Conversation(workspaceId, status, updatedAt)`.

---

### Week 8 — Notes + Quick Replies + Business Hours + Analytics (Aug 3–9)

**Objectives**
1. Operational polish.
2. Analytics dashboard.
3. **Feature freeze at end of week.**

**Deliverables**
- Private Notes on Conversations — UI in ConversationCard (right rail). Author + timestamp.
- Quick Reply Templates editor at `/app/(app)/settings/quick-replies` — list, add, edit, delete. Shortcuts (`/price`).
- Quick Reply autocomplete in `ReplyBox` — when agent types `/`, dropdown of matching shortcuts; Enter pastes the body.
- Business Hours auto-reply: in `process-inbound`, before classifying intent, check current time vs `WorkingHours` for the workspace. If outside hours, send the "out-of-office" message (configurable per workspace) and skip the classifier.
- Analytics dashboard at `/app/(app)/analytics/page.tsx`:
  - 6 cards above the fold: Messages handled · Bookings made · Conversion rate (bookings / unique contacts) · Avg response time · Handoffs · Top-asked intent
  - 1 line chart: messages over time (last 30 days)
  - Date-range toggle (7d / 30d / 90d)

**Acceptance Criteria**
- ✅ Out-of-hours message arrives → bot sends OOO reply, doesn't classify.
- ✅ `/price` in ReplyBox shows autocomplete; Enter pastes the body.
- ✅ Note added to a conversation appears in the conversation history + Contact timeline.
- ✅ Analytics dashboard loads in under 1s with 10k messages in the test DB.
- ✅ All 22 V1 features check off (run through §5 of strategic-review.html).

**Testing**
- Playwright e2e: every screen renders, no console errors, no failed network requests.
- Manual UX walkthrough by both founders.

**Risks**
- Analytics queries slow at scale. Mitigation: materialized views in Postgres OR cache results in Vercel KV for 5 min.

---

### Week 9 — Stripe Billing + Plan Gating + Trial + Demo Data (Aug 10–16)

**Objectives**
1. The business model works in code.
2. Server-side plan gating is bulletproof.

**Deliverables**
- Stripe products + prices created (Starter/Growth/Agency monthly + yearly).
- `lib/billing/stripe.ts` — Stripe SDK wrapper.
- Stripe checkout flow: from `/billing` page → choose plan → Stripe Checkout → return to `/billing/success`.
- Stripe webhook handler at `/api/webhooks/stripe/route.ts` — handles `checkout.session.completed`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`. Idempotent via WebhookDelivery.
- `Subscription` row maintained in sync with Stripe state.
- 14-day trial logic: new Workspace → Subscription with `status: TRIALING`, `trialEndsAt: now + 14 days`, `plan: GROWTH` (give them the full thing during trial).
- Trial-end cron: 24h before trial expires → email "Your trial ends tomorrow". On expire → if no payment method → `status: PAST_DUE`, downgrade `plan: STARTER`.
- `PlanFeatureGate` middleware applied to every relevant API route and Server Action.
- Daily reconciliation cron: pulls fresh subscription state from Stripe for all workspaces with `stripeSubscriptionId`, repairs drift.
- Demo data seeding: on `Workspace` creation, seed:
  - 1 demo Contact ("Sara Demo")
  - 1 demo Conversation with 3 inbound + 2 outbound messages showing the bot in action
  - Marker in metadata so we can later prune/hide demo data when the workspace gets its first real conversation.

**Acceptance Criteria**
- ✅ Sign up → land in inbox with the demo conversation visible.
- ✅ Real Stripe checkout completes (test mode) → Subscription updated in DB → UI reflects the new plan.
- ✅ Starter account `curl POST /api/chatbot/...` → 402 Plan Gate response with upgrade link.
- ✅ Trial expires (force `trialEndsAt` to past) → Subscription moved to PAST_DUE → email sent.
- ✅ Stripe webhook replayed (test) → idempotent (no double-update).

**Testing**
- Stripe webhook replay test.
- Plan-gate curl tests for each tier.
- Trial-end cron integration test.

**Risks**
- Stripe webhook out-of-order delivery. Mitigation: always trust the `created` timestamp + Stripe's `event.data.object` state, not the implied delta.

---

### Week 10 — Stabilize · Security · Marketing site · Code hygiene (Aug 17–23)

**Objectives**
1. Production-ready.
2. Every security checklist item closed.

**Deliverables**
- All Sentry alerts wired (404 spike, 5xx spike, slow query).
- PostHog tracking on every key flow: signup, setup_wizard_step_completed, message_received, message_sent, booking_created, rule_fired, trial_started, plan_upgraded, etc.
- Marketing site live — separate Next.js project in a `marketing/` workspace OR Framer site at the apex domain. App lives at `app.cowork.app`.
- Custom domain on Vercel with SSL.
- Every item in the Security Checklist below ticked:
  - ☐ Postgres RLS enabled on all workspace-scoped tables
  - ☐ Meta webhook signature verification on every channel route
  - ☐ Stripe webhook signature verification
  - ☐ Rate limit on auth endpoints + webhooks (Upstash)
  - ☐ Signed URLs for attachments (15-min expiry)
  - ☐ All secrets in Vercel env, not in repo
  - ☐ Sentry filtered for PII
  - ☐ GDPR data-export endpoint at `/api/workspace/export` working
  - ☐ Privacy policy + Terms live
- README + ops runbook complete: how to rotate Meta token, re-issue Anthropic key, roll DB credentials, manually retry a stuck webhook.
- Type-check + lint clean.
- Bug squash: P0/P1 all closed.

**Acceptance Criteria**
- ✅ Belal + Mahmoud onboard their own fake business and use it for 3 consecutive days with zero P0/P1 errors in Sentry.
- ✅ All security checklist boxes ticked.
- ✅ Marketing site live with the 4 pillars front and center + pricing page.

**Risks**
- Last-minute bugs from real usage. Mitigation: **no new features this week.**

---

### Week 11 — Private-beta Dry-run + Final Polish (Aug 24–31)

**Objectives**
1. One real (friendly) customer is live in production with no help from Belal.

**Deliverables**
- Onboard 1–2 friendly businesses through the Setup Wizard with no engineer intervention. Watch them. Fix every UX papercut surfaced.
- Final cron sanity check — 48 hours of clean Sentry / Inngest history.
- Privacy Policy + Terms of Service live and linked from signup + footer.
- GDPR data-export endpoint live: `GET /api/workspace/export` returns a zip of all the workspace's data as JSON files.
- Walk every item in the launch checklist.

**Acceptance Criteria**
- ✅ 1 friendly business completed Setup Wizard with zero help in under 20 min.
- ✅ 48h of clean cron runs (no unhandled errors in Sentry).
- ✅ Every box in the §10 launch checklist ☑.
- ✅ If any box is ☐ on Aug 31 → push launch by 1 week. Don't ship broken.

---

## 10. Acceptance Criteria & Testing Standards

### Definition of Done for any feature
1. TypeScript compiles cleanly with `pnpm typecheck`.
2. ESLint passes with `pnpm lint`.
3. Unit tests written for any pure function in `lib/`.
4. Integration tests written for any flow touching the DB or multiple files.
5. Playwright e2e covers the happy path of any user-facing feature.
6. PR description includes screenshots/video for any UI change.
7. Sentry / PostHog instrumented for the new flow.
8. AuditLog row written for any state-changing user action.
9. Mobile (375px) render checked manually.
10. Reviewed by Belal before merging to `main`.

### Test coverage targets
- `lib/` unit tests: 60%+ statement coverage.
- API routes: integration tests for the 80% happy path + the most likely failure mode.
- Critical e2e flows (must pass before each release):
  1. Signup → Setup Wizard → land on inbox with demo conversation
  2. Real WA inbound → bot replies → message appears in inbox UI
  3. Customer DMs "book" → bot replies with slots → customer picks → appointment created → reminder fires
  4. Owner adds a Custom Follow-Up Rule → trigger condition met → rule fires within 16 min
  5. Stripe checkout → plan upgraded → gated feature unlocked

---

## 11. Anti-Patterns — Refuse to Do These

If asked to do any of the following, push back. These are scope discipline rules.

| Ask | Refuse with |
|---|---|
| "Add a small extra feature not in §5 MVP table" | "That's V1.1 — adding to backlog." |
| "Make the chatbot generate its own replies" | "The fixed-reply mechanic is sacred — it's our differentiator." |
| "Skip the channel-connector interface for this one channel" | "Every channel goes through the connector. Otherwise the abstraction breaks." |
| "Add a new dependency without justification" | "We freeze the dep list. Use what we already have unless there's a real reason." |
| "Roll our own auth / billing / webhook signing" | "Use Clerk / Stripe / their libraries respectively." |
| "Skip writing the test for this PR" | "DoD requires tests. Write a quick one." |
| "Comment out the Postgres RLS check 'just for this query'" | "RLS stays on. Refactor the query to be tenant-aware." |
| "Hardcode a workspace ID for testing in production code" | "Use the test workspace seeded in `prisma/seed.ts`." |
| "Bypass plan gating to demo a feature for one customer" | "Run a manual upgrade on their workspace instead." |

---

## 12. How to Use This Prompt

### Starting a new Claude Code session
1. Open Claude Code in the `cw-platform/` repo.
2. Make sure `CLAUDE.md` is in the repo root (Claude Code auto-loads it).
3. State the task in plain English: e.g. *"Implement Week 3 deliverables per §9 of CLAUDE.md."*
4. Claude Code will reference the conventions, schema, and patterns in this file.

### When something isn't in this file
- If a decision can be made without affecting the 4 pillars or the architecture, make it and document it in a PR comment.
- If it changes the architecture, schema, or core patterns, **stop and ask Belal**. Don't guess.

### When the user (Belal) asks for a feature
- Check if it's in the V1 MVP table (§5 of `strategic-review.html`).
- If yes → proceed.
- If no → push back: *"That's V1.1 per the strategic review. Adding to backlog. Want me to proceed with V1 work instead?"*

### Communication style
- Be direct. No hedging.
- When unsure, ask 1 specific question, then proceed.
- Don't paraphrase the spec back at the user — assume they read it.
- When a sprint is done, summarize: what shipped, what tested green, what risks remain.

---

**End of CLAUDE.md.**

Last updated: Jun 14, 2026. Author: Co-founder strategic review pass.
Read this file at the start of every session.

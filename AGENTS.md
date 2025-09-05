# AGENTS.md — Baby Logger

Simple, minimal, black‑and‑white baby activity logger for two users (you and your partner).

## 0) Principles

- **Pages orchestrate** use‑cases. Components are **dumb** and composable.
- **Server Actions** (and Route Handlers for reads) are the only code that talks to the data layer. Data access is abstracted behind a **Repository** interface so we can swap **in‑memory → Firestore** later.
- **React Query** for client‑side reads/invalidation; mutations via Server Actions.
- **Shadcn UI + Tailwind** for a clean, black‑&‑white UI.
- **Auditability & privacy first**: every write stores `createdBy`, `createdAt`, `updatedAt`; sessions are cookie‑based (httpOnly, signed).

---

## 1) Architecture (high level)

```
App Router pages (RSC) ─▶ Server Actions (mutations)
                   │
                   └▶ Route Handlers /api (reads for React Query)

Server Actions & /api ─▶ Repository interfaces (Auth, Baby, Events, EventTypes)
                     └▶ InMemoryRepository (Phase 1)
                     └▶ FirestoreRepository (Phase 2)
```

**Data flow:**

- List screens fetch via React Query from `/api/...` (which call Repos).
- Create/Update/Delete use **Server Actions** which call Repos, then `revalidatePath()` and/or emit cache invalidation keys (React Query).

---

## 2) Domain model (TypeScript types)

```ts
export type UserEmail = string; // unique identifier for users
export type BabyId = string;
export type EventTypeId = string;
export type EventId = string;

export type User = {
  email: UserEmail;
  passwordHash: string; // bcrypt or argon2
  displayName?: string;
  createdAt: string; // ISO
  updatedAt: string; // ISO
};

export type Baby = {
  id: BabyId; // e.g., "laura"
  name: string; // e.g., "Laura"
  parents: UserEmail[]; // allowed users
  createdAt: string;
  updatedAt: string;
};

export type EventType = {
  id: EventTypeId;
  babyId: BabyId;
  name: string; // e.g., "Mamada", "Fralda", "Sono"
  active: boolean;
  order: number; // for UI sorting
  createdBy: UserEmail;
  createdAt: string;
  updatedAt: string;
};

export type BabyEvent = {
  id: EventId;
  babyId: BabyId;
  typeId: EventTypeId;
  // IMPORTANT: timestamp is NOT editable; set on creation.
  happenedAt: string; // ISO set by server at creation
  note?: string;
  createdBy: UserEmail;
  createdAt: string;
  updatedAt: string;
};
```

---

## 3) Data layer abstraction (Repositories)

```ts
export interface AuthRepository {
  getUserByEmail(email: string): Promise<User | null>;
  createUser(
    user: Omit<User, "createdAt" | "updatedAt" | "passwordHash"> & {
      password: string;
    }
  ): Promise<User>;
  verifyPassword(email: string, password: string): Promise<boolean>;
  updatePassword(email: string, newPassword: string): Promise<void>;
}

export interface BabyRepository {
  getDefaultBabyFor(email: string): Promise<Baby | null>; // single-baby app => returns one
}

export interface EventTypeRepository {
  list(babyId: BabyId): Promise<EventType[]>;
  create(input: {
    babyId: BabyId;
    name: string;
    createdBy: UserEmail;
  }): Promise<EventType>;
  update(
    id: EventTypeId,
    patch: Partial<Pick<EventType, "name" | "active" | "order">>
  ): Promise<EventType>;
  remove(id: EventTypeId): Promise<void>;
}

export interface EventRepository {
  list(
    babyId: BabyId,
    opts?: { limit?: number; cursor?: string }
  ): Promise<{ items: BabyEvent[]; nextCursor?: string }>;
  get(id: EventId): Promise<BabyEvent | null>;
  create(input: {
    babyId: BabyId;
    typeId: EventTypeId;
    note?: string;
    createdBy: UserEmail;
  }): Promise<BabyEvent>;
  update(
    id: EventId,
    patch: Partial<Pick<BabyEvent, "typeId" | "note">>
  ): Promise<BabyEvent>;
  remove(id: EventId): Promise<void>;
}
```

### In‑memory implementations (Phase 1)

- Use `Map<string, T>` per entity with simple auto‑id (`crypto.randomUUID()`).
- `happenedAt` is set to `new Date().toISOString()` on `create()` and never changed.
- Seed two users (your emails) and one baby (e.g., `id: 'laura'`).

### Firestore mapping (Phase 2)

**Collections**

- `users/{email}` → `{ email, passwordHash, displayName, createdAt, updatedAt }`
- `babies/{babyId}` → `{ name, parents[], createdAt, updatedAt }`
- `babies/{babyId}/eventTypes/{eventTypeId}` → `{ name, active, order, createdBy, createdAt, updatedAt }`
- `babies/{babyId}/events/{eventId}` → `{ typeId, happenedAt, note, createdBy, createdAt, updatedAt }`

**Indexes**

- `babies/{babyId}/events` order by `happenedAt` desc (default query)

**Server timestamps**: use `serverTimestamp()` for created/updated; still store `happenedAt` on create only.

---

## 4) Auth design (simple, cookie session)

- `POST /login` server action receives `{ email, password }`.
- Validate via `AuthRepository.verifyPassword`.
- On success, set an **httpOnly, Secure, signed** cookie `session` with `{ email }` for 30 days.
- `middleware.ts` enforces auth on `/app/**` and redirects to `/login` if missing/invalid.
- `POST /logout` server action clears the cookie.
- Passwords stored as **bcrypt** hashes (even for in‑memory).

**Security**

- Use `jose` to sign the cookie payload (HS256, secret in `AUTH_SECRET`).
- No third‑party providers; 2 users only.

---

## 5) Server Actions (mutations)

```ts
// auth
export const signIn = async (formData: FormData) => {
  /* set cookie; redirect('/app') */
};
export const signOut = async () => {
  /* clear cookie; redirect('/login') */
};

// event types
export const createEventType = async (babyId: BabyId, name: string) => {
  /* repo.create; revalidatePath('/app/types') */
};
export const updateEventType = async (
  id: EventTypeId,
  patch: Partial<{ name: string; active: boolean; order: number }>
) => {
  /* ... */
};
export const deleteEventType = async (id: EventTypeId) => {
  /* ... */
};

// events
export const createEvent = async (
  babyId: BabyId,
  typeId: EventTypeId,
  note?: string
) => {
  /* repo.create; revalidatePath('/app') */
};
export const updateEvent = async (
  id: EventId,
  patch: Partial<{ typeId: EventTypeId; note?: string }>
) => {
  /* ... */
};
export const deleteEvent = async (id: EventId) => {
  /* ... */
};
```

**Notes**

- All actions read `email` from server session (cookie) to set `createdBy`.
- After each mutation, call `revalidatePath()` to refresh RSC trees.

---

## 6) Route Handlers (reads for React Query)

- `/api/event-types?babyId=...` → `EventType[]`
- `/api/events?babyId=...&cursor=...&limit=...` → `{ items, nextCursor }`
- `/api/me` → current user (email, displayName)

Each handler:

- Validates auth from cookie.
- Calls the appropriate Repository.
- Returns JSON.

---

## 7) UI (pages & components)

### Pages

- `/login` — email/password form (centered card). On success, redirect `/app`.
- `/app` — list of events for the default baby. Header with baby name + `+` button.
- `/app/types` — manage Event Types (list + inline edit + toggle active + reorder).

### Components (dumb)

- `Header` — shows baby name; `Sign out` button.
- `EventList` — receives `events` and `onSelect(id)`.
- `EventItem` — minimal row (type name • time • note excerpt).
- `EventModal` — add/edit; when editing, `happenedAt` is shown but **disabled**.
- `TypeList` — list + actions; uses `Dialog` for add/edit.
- `AddFab` — floating `+` button (bottom‑right).

**Style**: black text on white; use Tailwind + shadcn `Card`, `Dialog`, `Button`, `Input`, `Select`, `Table`, `Textarea`.

---

## 8) UX rules

- **Timestamp** set by server on creation; read‑only everywhere.
- **Type** chosen from active types; sorted by `order`.
- **Note** optional, multi‑line; show first line in list.
- **Empty state**: clear calls to action, e.g., "Nenhum evento ainda — clique em + para registrar o primeiro".
- **Errors**: show inline shadcn `Toast` or `FormMessage` with short messages.

---

## 9) React Query usage

- Provide `QueryClientProvider` at `app/providers.tsx`.
- Reads use `/api/...` with `fetch` wrappers.
- After Server Action mutations, **invalidate** `events`/`eventTypes` queries with a client helper (`actionSuccess$` event or refetch on modal close).

---

## 10) File/folder layout (App Router)

```
app/
  layout.tsx
  providers.tsx             // QueryClientProvider
  login/
    page.tsx
    actions.ts              // signIn, signOut
  app/
    page.tsx                // events list (RSC shell + client list)
    types/page.tsx          // event types management
    actions.ts              // create/update/delete event(s) & types
api/
  events/route.ts           // GET list
  event-types/route.ts      // GET list
  me/route.ts               // GET current user

lib/
  auth/session.ts           // sign/verify cookie with jose
  auth/requireUser.ts       // helper for server actions & routes
  repo/
    types.ts                // domain types & interfaces
    memory/
      auth.repo.ts
      baby.repo.ts
      events.repo.ts
      eventTypes.repo.ts
      seed.ts
    firestore/
      auth.repo.ts          // TODO (Phase 2)
      baby.repo.ts          // TODO (Phase 2)
      events.repo.ts        // TODO (Phase 2)
      eventTypes.repo.ts    // TODO (Phase 2)
  utils/time.ts             // toISO(), now(), tz helpers if needed

components/
  header.tsx
  add-fab.tsx
  events/
    event-list.tsx
    event-item.tsx
    event-modal.tsx
  types/
    type-list.tsx
    type-modal.tsx

middleware.ts               // auth gate for /app/**
```

---

## 11) Minimal visual spec (B/W)

- **Typography**: system font; `text-gray-900` on `bg-white`.
- **Buttons**: solid black (`bg-black text-white`) and outline (`border-black text-black`).
- **Links**: underline on hover.
- **Cards**: `rounded-2xl`, subtle shadow, generous padding.
- **Dialogs**: centered, max‑w‑md, full‑width inputs, dense labels.

---

## 12) Testing

- **Unit**: repositories (memory) — CRUD, invariants (`happenedAt` immutable).
- **Integration**: route handlers auth checks, server actions side‑effects (revalidate called).
- **E2E (later)**: basic login → add event → edit → delete.

---

## 13) Migration plan (memory → Firestore)

1. Implement Firestore repos adhering to the same interfaces.
2. Add `.env` for Firebase creds (client & admin SDK as needed).
3. Toggle implementation via a single env flag `DATA_BACKEND=memory|firestore`.
4. Add import task to upsert seed data (event types).

---

## 14) Environment

```
AUTH_SECRET=...           # for cookie signing (jose HS256)
ALLOWED_USERS=alice@example.com,bob@example.com
# FIREBASE_... (Phase 2)
DATA_BACKEND=memory
```

---

# Implementation Tasks (for Codex)

> **Goal:** build incrementally, never breaking auth or the write path; keep components dumb.

## Phase 0 — Scaffold & plumbing

1. **Create project skeleton** (Next.js App Router, TS, Tailwind, shadcn).

   - Files: `app/layout.tsx`, `tailwind.config.ts`, shadcn init.
   - AC: app runs; base styles applied; shadcn installed.

2. **Add Query Client provider**.

   - Files: `app/providers.tsx`, wrap in `layout.tsx`.
   - AC: `react-query` devtools optional (off by default).

## Phase 1 — In‑memory data layer

3. **Define domain types & repository interfaces.**

   - Files: `lib/repo/types.ts`.
   - AC: Types compile; interfaces exported.

4. **Implement in‑memory repositories + seed.**

   - Files: `lib/repo/memory/*.ts`, `lib/repo/memory/seed.ts`.
   - Seed: two users (emails from `ALLOWED_USERS` with password `changeme`), baby `laura`, default event types.
   - AC: CRUD works in unit tests.

5. **Repo factory by env flag.**

   - Files: `lib/repo/index.ts` (`getRepos()` returning concrete impls).
   - AC: switchable via `DATA_BACKEND`.

## Phase 2 — Auth (cookie session)

6. **Session utils (sign/verify).**

   - Files: `lib/auth/session.ts` (uses `jose`).
   - AC: `createSession(email)` → cookie; `getUserFromSession()` parses/validates.

7. **Server actions: `signIn`, `signOut`.**

   - Files: `app/login/actions.ts`.
   - AC: correct cookie set/cleared; bad creds show error.

8. **Login page.**

   - Files: `app/login/page.tsx`.
   - AC: centered card; email/password; calls `signIn`; redirects `/app` on success.

9. **Middleware gate.**

   - Files: `middleware.ts`.
   - AC: unauthenticated users hitting `/app/**` get 302 to `/login`.

## Phase 3 — Reads via Route Handlers + React Query

10. **`/api/me`** returns current user.

    - Files: `app/api/me/route.ts`.
    - AC: 200 with `{ email }` when logged; 401 otherwise.

11. **`/api/event-types`** list by baby.

    - Files: `app/api/event-types/route.ts`.
    - AC: returns sorted active types.

12. **`/api/events`** list by baby with pagination.

    - Files: `app/api/events/route.ts`.
    - AC: returns `{ items, nextCursor }` ordered by `happenedAt` desc.

## Phase 4 — Mutations via Server Actions

13. **Event Types actions**: `createEventType`, `updateEventType`, `deleteEventType`.

    - Files: `app/app/actions.ts`.
    - AC: Writes succeed; `revalidatePath('/app/types')` called.

14. **Events actions**: `createEvent`, `updateEvent`, `deleteEvent`.

    - Files: `app/app/actions.ts`.
    - AC: Writes succeed; `revalidatePath('/app')` called; `happenedAt` immutable.

## Phase 5 — Pages & dumb components

15. **Header** with baby name and sign‑out.

    - Files: `components/header.tsx`.
    - AC: Black button styles; sign‑out posts server action.

16. **Events page `/app`** (RSC shell + client list).

    - Files: `app/app/page.tsx`, `components/events/*`, `components/add-fab.tsx`.
    - AC: React Query fetches; `+` opens modal; create/edit uses server actions; list updates.

17. **Types page `/app/types`**.

    - Files: `app/app/types/page.tsx`, `components/types/*`.
    - AC: Create/edit/toggle/remove types; order field editable.

## Phase 6 — Polish & UX

18. **Empty states, toasts, loading skeletons.**

    - Files: relevant components.
    - AC: Smooth UX; consistent black‑&‑white aesthetic.

19. **Tests (unit/integration).**

    - Files: `__tests__` for repos and handlers.
    - AC: CI green.

## Phase 7 — Firestore (optional, when ready)

20. **Firestore config & repos.**

    - Files: `lib/repo/firestore/*.ts`.
    - AC: Same interface; all tests pass with `DATA_BACKEND=firestore`.

21. **Env & deployment.**

    - Files: `.env.example`, `README` note.
    - AC: Runs on Vercel with Firestore.

---

## Acceptance Criteria (global)

- `happenedAt` is set at create time and is never editable.
- Only the two configured users can log in; every event stores `createdBy`.
- **No component** directly calls a database SDK; only Actions or `/api` call repos.
- UI is minimal, black‑and‑white, accessible.
- Swapping data backend requires **no UI** changes.

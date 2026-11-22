# CLAUDE.md

Guidance for Claude Code in this repo. Read it before every task. These rules take precedence over default habits.

## Project

**Dogear** (working name; repo `book-bros`) is a community webapp for people who read **physical** books.

- **Phase 1 (build now):** profiles, book discovery, personal collections, book buddies, 1:1 chat, block/report.
- **Phase 2 (do NOT build):** lending/borrowing. Add no lending UI, routes, tables or columns. Phase 1 schema must still leave room for it (see [Lending-ready schema](#lending-ready-schema)).
- The name will change. Keep it in one constant (`src/lib/site.ts`) and never hard-code "Dogear" anywhere else.

## Stack

- Next.js (App Router) + TypeScript (`strict`) + Tailwind CSS, deployed on **Vercel Hobby**.
- **Supabase Free**: Auth (Google OAuth only), Postgres with row-level security, Realtime. Use `@supabase/ssr` + `@supabase/supabase-js`.
- Book data: **Open Library** (primary), **Google Books** (fallback for ISBNs Open Library lacks).

## Non-negotiable rules

1. **Zero cost.** Use free tiers only. Don't add any service, SaaS or package that needs an account or billing without asking first. We run no server of our own: everything is Next.js on Vercel plus Supabase.
2. **RLS is the security boundary. Never trust the client.** All data access goes through Supabase. Every table has RLS enabled with explicit policies. Client-side checks are for UX only. The database must reject anything a user isn't allowed to do, even when they call the Supabase API directly with their own JWT.
3. **Secrets live in `.env.local`** (git-ignored). Keep `.env.example` in sync: add a key there in the same commit that first reads it.
4. **Mobile-first and responsive.**
5. **Commit after each working feature** (see [Workflow](#workflow)).

Free-tier facts that shape the design:
- Vercel Hobby is for non-commercial use only, and its image optimisation quota is small, so we don't use it.
- The Supabase Free database is 500 MB, and the project pauses after about a week of inactivity. Store references, not blobs. Don't put cover images in Storage.
- Supabase Realtime caps concurrent connections. Open channels only while needed.

## Supabase and security

- There are three clients in `src/lib/supabase/`:
  - `client.ts`: browser, publishable key.
  - `server.ts`: Server Components, Server Actions and Route Handlers. Uses the publishable key plus the user's cookies, so RLS applies.
  - `admin.ts`: secret key, starts with `import 'server-only'`.
- **The secret key bypasses RLS.** Only `admin.ts` may use it, and only to write the shared book catalog (`books`) with data the server fetched itself from Open Library or Google Books. Never use it for user-owned data. Never pass it client-supplied fields other than an identifier to look up.
- **The secret key never reaches the browser.** It never gets a `NEXT_PUBLIC_` prefix, is never imported by client code (`server-only` turns that into a build error), and is never logged, returned in a response or passed as a prop. `.env.example` holds only an empty placeholder for it.
- On the server, authorise with `supabase.auth.getUser()` or `getClaims()`, never `getSession()`, because the session cookie is unverified.
- Session refresh goes in `proxy.ts` (`middleware.ts` on Next ≤ 15), following the `@supabase/ssr` guide.
- Google OAuth sign-in:
  - Call `signInWithOAuth({ provider: 'google', options: { redirectTo: `${origin}/auth/callback` } })`. The route `src/app/auth/callback/route.ts` then calls `exchangeCodeForSession`.
  - The Google client ID and secret live in the Supabase dashboard, not in our env.
  - Add localhost and the Vercel preview URL wildcard to Supabase's redirect allow-list.
- Schema changes are made only as SQL migrations in `supabase/migrations/` (Supabase CLI), committed to git. Never edit the schema by hand in the dashboard.
- A migration that creates a table also enables RLS on it and defines its policies, in the same file.
- Policy conventions:
  - Write one policy per operation.
  - `update` policies need both `using` and `with check`.
  - Write `(select auth.uid())`, not bare `auth.uid()`, so it is evaluated once per query.
  - Index the columns that policies filter on.
  - Helper functions used in policies are `security definer` with `set search_path = ''`, and are not exposed as RPCs unless that is intended.
- Enforce invariants in the database: `check` constraints (lengths, allowed values), `not null`, unique constraints, and foreign keys with a deliberately chosen `on delete`. Also validate input in Server Actions, but the database is the backstop.
- After each migration, regenerate types with `npx supabase gen types typescript --linked > src/types/database.ts`. Don't use `any` for rows.
- After schema changes, check the dashboard's Security Advisor, and test new policies as a second user and as a signed-out visitor.

## Visibility and privacy

- **Signed-in users** (`authenticated` role) can see other users' profiles and collections, except where either user has blocked the other.
- **Signed-out visitors** (`anon` role) can see only book pages (`/books/[id]`), each with its total owner count, plus the landing and sign-in pages. Every other route redirects to sign-in.
- The redirect is only UX. RLS is the enforcement:
  - Write every user-data policy `to authenticated`, so `anon` gets no rows from profiles, copies, buddies, conversations or messages.
  - `books` is readable `to anon, authenticated`.
  - Owner counts come from a `security definer` function (for example `public.book_owner_counts(book_ids uuid[])`) granted to `anon`. It returns only `(book_id, owner_count)`, counting distinct owners. Never grant `anon` select on `copies`, and never reveal who the owners are.
- **Location:** profiles may hold optional `area` and `city` text. Never store or show anything more precise: no coordinates, street addresses, postcodes or distances.
- Each `auth.users` row gets one `profiles` row, created by a trigger on signup. Email never goes in a readable table or the UI.

## Book data

- All external book API calls go through `src/lib/books/` on the server, never from the browser, so we can cache, set headers and keep the Google key private. The one exception is cover `<img>` tags, which load straight from the provider's CDN.
- **Open Library first:**
  - Search: `https://openlibrary.org/search.json?q=…&fields=…&limit=…`. Always pass `fields`.
  - ISBN lookup: `/isbn/{isbn}.json`. Works: `/works/{id}.json`.
  - Send `User-Agent: Dogear/<version> (<OPEN_LIBRARY_CONTACT>)`, as Open Library requests.
- **Google Books fallback,** only when Open Library has no record for an ISBN: `https://www.googleapis.com/books/v1/volumes?q=isbn:{isbn}`, adding `&key=` if a key is set. Thumbnails come back as `http://`; rewrite them to `https://`.
- Give every request a timeout (`AbortSignal.timeout(5000)`) and cache it (`next: { revalidate: 86400 }`, or `'use cache'` if Cache Components is enabled). Open Library is sometimes slow or down, so degrade gracefully.
- Normalise ISBN-10 to ISBN-13 before storing or comparing.
- **`books` is our cached catalog:**
  - Primary key: an internal UUID. Also unique and nullable: `isbn13`, `ol_work_id`, `ol_edition_id`, `google_volume_id`.
  - Everyone can read it. Writes go only through `admin.ts`.
  - Rows are created lazily, when someone first adds the book. We don't mirror the catalog.
- **Covers:**
  - Use plain `<img>`, never `next/image`.
  - Prefer `https://covers.openlibrary.org/b/id/{cover_i}-M.jpg`: lookups by ISBN are rate-limited per IP (about 100 per 5 minutes), but lookups by cover ID or OLID aren't.
  - Append `?default=false` so a missing cover returns a 404, and show a placeholder `onError`.
  - Always set `width`/`height` (or an aspect ratio), `alt` and `loading="lazy"`.
  - As a safety net, set `images: { unoptimized: true }` in `next.config`, and turn off the ESLint rule `@next/next/no-img-element` with a comment pointing here.

## Book buddies

- Buddies are mutual: one user sends a request, the other accepts or declines.
- Store one row per pair in canonical order (`check (user_a < user_b)`), with a unique key on the pair plus `status` and `requested_by`. This stops duplicates and A→B / B→A races.
- Only the recipient can accept. Either user can remove the link.

## Chat

- **Chat is between accepted book buddies only.** Enforce it in RLS with a `security definer` helper (for example, `private.are_buddies(a, b)`):
  - creating a conversation requires the two users to be accepted buddies,
  - sending a message requires that they still are, and that neither has blocked the other.

  Participants can still read the history of their own conversations after the buddy link ends.
- **Persist, then deliver.** A message is a Postgres row, inserted through RLS. Realtime then delivers it.
- Default delivery: Postgres Changes on `messages`, filtered by `conversation_id=eq.{id}`. RLS select policies decide who receives each event. Add the table to the `supabase_realtime` publication in its migration.
- If load grows, move to private Broadcast channels authorised by RLS on `realtime.messages`. Never switch to broadcast-only, unpersisted messages.
- Open one channel per open conversation and call `removeChannel` on unmount. Load history with keyset pagination on `(created_at, id)`.
- The schema keeps conversations generic (`conversations` + `conversation_participants`), with no buddy foreign key. The buddies-only rule lives in the policies, so phase 2 can loosen it for loan conversations without a migration of existing data.
- Messages are text only, with a length `check` constraint. Render them as plain text and never use `dangerouslySetInnerHTML`.

## Block and report

- **Blocking is enforced in RLS, not the UI.** If either user has blocked the other:
  - no new conversations or messages between them,
  - no buddy requests,
  - neither appears in the other's discovery or buddy lists.
- Use one `security definer` helper (for example, `private.is_blocked(a, b)`) in every relevant policy. It must be `security definer` because the blocked user can't read the blocker's rows.
- Blocking someone also deletes any buddy link between the pair (via a trigger), which ends their chat too.
- Users can read and delete only their own `blocks` rows.
- Reports:
  - Users can insert reports (`with check (reporter_id = (select auth.uid()))`) and read only their own.
  - Nobody else reads reports through the API. Moderation happens in the Supabase dashboard for now.
  - A report snapshots the reported text, because messages can be deleted.

## Lending-ready schema

Phase 2 must be able to add lending with new tables, without reworking phase 1 tables:
- **Owned physical copies are their own rows** (`copies`: UUID, `owner_id`, `book_id`, optional condition and notes), separate from reading status and lists. Future loans will reference `copies.id`. Don't collapse "I own this" into a boolean.
- Optional `area` and `city` on profiles (see [Visibility and privacy](#visibility-and-privacy)).
- Generic conversations (see [Chat](#chat)).
- Model states as text with `check` constraints, or as extendable enums, not as piles of booleans.
- Not now: no `loans` table, no `is_lendable` flag, no lending routes or UI.

## UI

- Mobile-first: unprefixed Tailwind classes target phones; add `sm:`/`md:`/`lg:` for larger screens. Check every screen at 360 px wide.
- Touch targets are at least 44 px, and nothing works only on hover. Full-height layouts (such as chat) use `dvh` and respect `env(safe-area-inset-*)`.
- Use Server Components by default. Add `'use client'` only where interaction needs it (forms, chat, realtime).
- Accessibility: semantic HTML, a label on every input, alt text on covers, visible focus.

## Environment variables

```bash
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=  # publishable (or legacy anon) key, safe in the browser, RLS applies
SUPABASE_SECRET_KEY=                   # secret (or legacy service_role) key, server only, bypasses RLS: admin.ts only
GOOGLE_BOOKS_API_KEY=                  # optional, server only
OPEN_LIBRARY_CONTACT=                  # email or URL for the Open Library User-Agent
```

These keys live in `.env.example` with empty values; copy it to `.env.local` and fill it in. Only `NEXT_PUBLIC_*` variables reach the browser, so nothing secret may use that prefix. Mirror the variables in the Vercel project settings for Production and Preview.

`create-next-app` writes a `.gitignore` that ignores `.env*`, which also hides `.env.example`. When scaffolding, add `!.env.example` to it.

## Workflow

- The app isn't scaffolded yet. Planned npm scripts: `dev`, `build`, `lint`, `typecheck` (`tsc --noEmit`). Update this section once they exist.
- Local Supabase is optional: `npx supabase start` needs Docker. Otherwise use the hosted free project.
- **"Working" means all of these:**
  - `lint`, `typecheck` and `build` pass.
  - The feature works at phone width.
  - New RLS policies have been checked as a second user and as a signed-out visitor.

  Then commit: one feature per commit, with its migration in the same commit.
- Never commit `.env.local` or any key.

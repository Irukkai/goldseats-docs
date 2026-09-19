# TypeScript and React Conventions (`goldseats-web`)

Applies to everything in `goldseats-web`: Next.js 15 App Router, TypeScript strict mode,
Tailwind CSS, react-three-fiber. Enforced by ESLint + Prettier in CI; anything not
mechanically enforceable is a review item under [`code-review.md`](code-review.md).

---

## Tooling

- **Package manager: npm.** Not pnpm, not yarn. `package-lock.json` is committed and
  CI uses `npm ci`.
- **ESLint** with `next/core-web-vitals`, `@typescript-eslint` (type-aware),
  `eslint-plugin-jsx-a11y`.
- **Prettier** for all formatting. No formatting arguments in review. 100-column print
  width, single quotes, semicolons, trailing commas.
- **TypeScript strict.** `strict: true`, plus `noUncheckedIndexedAccess` and
  `noImplicitOverride`. These stay on.

```bash
npm run lint       # eslint . --max-warnings 0
npm run typecheck  # tsc --noEmit
npm run test       # vitest run
npm run build      # next build
```

Run all four before pushing. CI runs exactly these.

---

## Project layout

```
src/
  app/                        # App Router. Routes only — no business logic.
    (marketing)/page.tsx      # migrated landing page
    (app)/page.tsx            # home: now-playing + upcoming
    (app)/films/[filmId]/page.tsx
    (app)/showtimes/[showtimeId]/seats/page.tsx
    api/                      # route handlers — only for BFF-ish needs, never a second backend
    layout.tsx
  components/
    ui/                       # design-system primitives: Button, Card, Skeleton, Select
    films/                    # FilmCard, FilmGrid, FilmTimeline, FormatRanking
    seats/                    # SeatMap2D, SeatMapLegend, SeatScoreBadge
    seats-3d/                 # react-three-fiber: AuditoriumScene, SeatCamera, ScreenMesh
    chat/                     # ChatPanel, ChatMessage, ToolCallTrace
  lib/
    api/                      # generated + hand-written goldseats-api client
    format/                   # date, runtime, currency, language-label formatters
    hooks/
  types/
    api.ts                    # types generated from the API's OpenAPI schema
```

Rules:

- `app/` files fetch data and compose components. Logic lives in `lib/`, markup in
  `components/`.
- Components are **server components by default**. Add `'use client'` only when you need
  state, effects, or browser APIs. Every `'use client'` in a PR gets a look in review.
- Never import from `components/seats-3d/` at the top level of a server component — it
  pulls three.js into the initial bundle. Use `next/dynamic` with `ssr: false`.

---

## Naming

| Thing | Convention | Example |
| --- | --- | --- |
| Component file | `PascalCase.tsx` | `SeatMap2D.tsx` |
| Non-component file | `kebab-case.ts` | `format-showtime.ts` |
| Component | `PascalCase` | `FilmTimeline` |
| Hook | `useCamelCase` | `useSeatRecommendations` |
| Type / interface | `PascalCase`, no `I` prefix | `SeatRecommendation` |
| Constant | `SCREAMING_SNAKE_CASE` | `MAX_PARTY_SIZE` |
| Boolean | `is` / `has` / `can` prefix | `isSoldOut`, `canReserveInPerson` |

Domain vocabulary is fixed by [`../product/glossary.md`](../product/glossary.md). A seat
score is a `score`, never a `rating`. An auditorium is an `auditorium`, never a `screen`
or a `room`. Consistency with the API's field names matters more than local elegance.

---

## Types

Never hand-write API response types. Generate them from the API's OpenAPI schema:

```bash
npm run generate:api-types   # openapi-typescript against http://localhost:8000/openapi.json
```

`any` is banned by ESLint. For genuinely unknown input, use `unknown` and narrow:

```ts
// Good — narrow at the boundary, trust the type after.
function parseSeatLayout(raw: unknown): SeatLayout {
  const result = seatLayoutSchema.safeParse(raw);
  if (!result.success) {
    throw new SeatLayoutError(`Invalid seat_layout: ${result.error.message}`);
  }
  return result.data;
}
```

Prefer discriminated unions over optional-field soup. This models the booking fork in
[M5](../product/milestones/M5.md) honestly:

```ts
// Good
type BookingOutcome =
  | { kind: 'online'; bookingUrl: string; provider: 'cineplex' }
  | { kind: 'reserved_in_person'; reservationId: string; holdsUntil: string };

// Bad — every consumer has to re-derive which combination is legal
type BookingOutcome = {
  kind: string;
  bookingUrl?: string;
  reservationId?: string;
  holdsUntil?: string;
};
```

Use `as const` for literal unions that mirror API enums:

```ts
export const PARTY_SCENARIOS = ['solo', 'pair', 'small_group', 'large_group', 'family'] as const;
export type PartyScenario = (typeof PARTY_SCENARIOS)[number];
```

Avoid `as` casts. If you need one, add a one-line comment saying what invariant makes it
safe.

---

## Components

Function declarations, named exports, props typed inline or as a local `Props` type:

```tsx
type SeatScoreBadgeProps = {
  score: number; // 0–100, as returned by POST /v1/recommendations
  isGolden: boolean;
};

export function SeatScoreBadge({ score, isGolden }: SeatScoreBadgeProps) {
  return (
    <span
      className={clsx(
        'rounded-full px-2 py-0.5 text-xs font-semibold tabular-nums',
        isGolden ? 'bg-gold-500 text-ink-950' : 'bg-ink-800 text-ink-200',
      )}
      aria-label={`Seat score ${score} out of 100`}
    >
      {score}
    </span>
  );
}
```

- No default exports except where Next.js requires them (`page.tsx`, `layout.tsx`,
  `error.tsx`, `loading.tsx`).
- One exported component per file. Small private subcomponents in the same file are fine.
- Props are read-only. Never mutate a prop or an object reached through one.
- Derive, don't sync. If a value can be computed from props, compute it — don't mirror it
  into `useState` and keep it in step with an effect.
- `useEffect` is for synchronising with something outside React. Data fetching belongs in
  a server component or a route handler, not an effect.

---

## Data fetching

Server components fetch directly, with explicit cache semantics:

```tsx
// src/app/(app)/page.tsx
export default async function HomePage({
  searchParams,
}: {
  searchParams: Promise<{ title?: string; language?: string; releasedAfter?: string }>;
}) {
  const filters = await searchParams;
  const [nowPlaying, upcoming] = await Promise.all([
    getNowPlaying(filters),
    getUpcoming(filters),
  ]);

  return (
    <>
      <FilmGrid heading="Now playing" films={nowPlaying} />
      <FilmGrid heading="Coming soon" films={upcoming} />
    </>
  );
}
```

```ts
// src/lib/api/films.ts
export async function getNowPlaying(filters: FilmFilters): Promise<FilmSummary[]> {
  const res = await fetch(apiUrl('/v1/films/now-playing', filters), {
    // Catalog data changes at most hourly; see ../engineering/performance-budgets.md
    next: { revalidate: 900, tags: ['films:now-playing'] },
  });
  if (!res.ok) throw new ApiError(res.status, await res.text());
  return (await res.json()).data;
}
```

Caching rules, aligned with [`performance-budgets.md`](performance-budgets.md):

| Data | Strategy |
| --- | --- |
| Film catalog, film detail | `revalidate: 900` with a cache tag |
| Theatre and seat layout | `revalidate: 86400` — layouts change almost never |
| Showtimes | `revalidate: 300` |
| Seat availability, recommendations | `cache: 'no-store'` — never cache a seat map |
| Anything user-specific (feed, watchlist) | `cache: 'no-store'` |

Every route that awaits data ships a `loading.tsx` skeleton and an `error.tsx`. A blank
screen during a fetch is a bug.

---

## Styling

Tailwind only. No CSS modules, no styled-components, no inline `style` except for values
that are genuinely dynamic (a seat's computed `left`/`top` on the 2D map, for instance).

Design tokens come from the landing page palette and live in `tailwind.config.ts`:

```ts
theme: {
  extend: {
    colors: {
      ink: { 950: '#0a0a0f', 900: '#12121a', 800: '#1c1c28', 200: '#c9c9d4' },
      gold: { 500: '#FFD700', 600: '#e6c200', 400: '#ffe34d' },
    },
    fontFamily: {
      sans: ['var(--font-inter)', 'system-ui', 'sans-serif'],
      display: ['var(--font-playfair)', 'Georgia', 'serif'],
    },
  },
}
```

Never write a raw hex value in a component. `#FFD700` appears exactly once in the repo,
in that config. Use `clsx` for conditional classes, and `tailwind-merge` when a component
accepts a `className` override.

Mobile-first: unprefixed classes are the phone layout, `sm:`/`md:`/`lg:` add to it.

---

## Errors

```ts
export class ApiError extends Error {
  constructor(
    readonly status: number,
    readonly detail: string,
    readonly requestId?: string,
  ) {
    super(`API ${status}: ${detail}`);
    this.name = 'ApiError';
  }
}
```

- Throw typed errors, never bare strings.
- User-facing copy is written for the user, not the developer. "We couldn't load seats
  for this showtime — try again in a moment." not "Request failed with status 502".
- Always surface the API's `request_id` in the error UI, quietly, so a bug report can be
  correlated with logs. See [`observability.md`](observability.md).

---

## Accessibility and performance

Both have their own docs and both are review blockers:

- [`accessibility.md`](accessibility.md) — the 2D seat map in particular must be fully
  keyboard-navigable and must never encode a seat's quality in colour alone.
- [`performance-budgets.md`](performance-budgets.md) — Lighthouse above 90 is a
  [M3](../product/milestones/M3.md) done-when criterion, and three.js must stay out of
  the initial bundle.

---

## Testing

See [`testing-strategy.md`](testing-strategy.md). In this repo:

```tsx
// src/components/seats/SeatScoreBadge.test.tsx
import { render, screen } from '@testing-library/react';
import { SeatScoreBadge } from './SeatScoreBadge';

it('exposes the score to assistive tech, not just colour', () => {
  render(<SeatScoreBadge score={98} isGolden />);
  expect(screen.getByLabelText('Seat score 98 out of 100')).toHaveTextContent('98');
});
```

Query by role and accessible name. `getByTestId` is a last resort — if you can't find an
element the way a user would, that's usually the finding.

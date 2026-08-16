---
name: server-first-rendering
description: Build or refactor Next.js App Router UI so it renders on the server by default, with the smallest possible client islands. Use when asked to make a page/feature server-side, reduce "use client", move filtering/sorting/pagination to the server, drive state from the URL, add Suspense streaming with skeletons, or split a client-heavy component into server + client parts.
---

# Server-First Rendering (RSC)

How to make a feature render on the server as much as possible, keeping `"use client"` confined to leaf interactivity. Default to a Server Component; reach for a client island only when something needs a hook, an event handler, or a browser API.

Every pattern below is self-contained — the code is the reference. Adapt names to the feature at hand (`<Feature>`, `<feature>`); the shapes are what matter.

## Core principles

1. **Server by default.** A file with no `"use client"` is a Server Component. Keep `page.tsx`, layout/orchestration, data shaping, lists, and list items on the server. Add `"use client"` only at the leaf that genuinely needs `useState`/`useEffect`/`useRef`, an `onClick`/`onChange`, `useRouter`/`useSearchParams`, or `window`/`localStorage`.
2. **Push the boundary down, not up.** When a component is "mostly static with one interactive bit," don't mark the whole thing client. Extract the interactive bit into its own client component and keep the parent on the server. A Server Component can freely render client components as children.
3. **URL is the source of truth for view state.** Filters, sort, and pagination live in `searchParams`, not React state. The server reads them; client islands write them. This makes state shareable, bookmarkable, and back-button-friendly, and removes a whole class of `useState`.
4. **Keep the shell static; stream the dynamic part.** The parts of the page that don't depend on the URL should not read it, so they prerender. Wrap only the URL-dependent regions in `<Suspense>` with a skeleton fallback.

## Patterns

### page.tsx awaits searchParams

`searchParams` is a Promise in current Next.js. Keep the page thin — await and hand off.

```tsx
// page.tsx (Server Component)
export default async function Page({
  searchParams,
}: {
  searchParams: Promise<SearchParams>;
}) {
  return <Browse searchParams={await searchParams} />;
}
```

`type SearchParams = Record<string, string | string[] | undefined>`.

### Static shell + URL-dependent islands (the important one)

The strongest version of this pattern: the orchestrator **never reads the URL itself**. It renders the whole static shell — heading, filter chrome, layout — and passes the *unawaited* `searchParams` Promise down to small async islands that each await it inside their own `<Suspense>`.

Because the shell doesn't depend on `searchParams`, it prerenders and is restored instantly on back-navigation; only the islands re-stream.

```tsx
// browse.tsx (Server Component — no "use client", no await of searchParams)
export async function Browse({
  searchParams,
}: {
  searchParams: Promise<SearchParams>;
}) {
  // Options for the filter UI come from the cached data source, not the URL,
  // so the shell stays static.
  const options = deriveOptions(await getItems());

  return (
    <div className="container mx-auto px-5 py-8">
      <h1 className="text-3xl font-semibold tracking-tight">Browse</h1>

      {/* Each island awaits searchParams itself, behind its own boundary. */}
      <Suspense fallback={<Skeleton className="mt-1 h-5 w-44" />}>
        <ResultsSummary searchParams={searchParams} />
      </Suspense>

      <FiltersPanel options={options} />   {/* client island, URL-writing */}

      <Suspense fallback={<SkeletonGrid count={6} />}>
        <Results searchParams={searchParams} />
      </Suspense>
    </div>
  );
}

/* One shared async helper so every island computes the same result set.
   The underlying fetch is cached, so the repeated calls collapse to one read. */
async function getResults(searchParams: Promise<SearchParams>) {
  const [items, sp] = await Promise.all([getItems(), searchParams]);
  return filterItems(items, parseFilters(sp), parseSort(sp));
}

async function ResultsSummary({ searchParams }: { searchParams: Promise<SearchParams> }) {
  const results = await getResults(searchParams);
  return <p className="mt-1 text-muted-foreground">{results.length} results</p>;
}

async function Results({ searchParams }: { searchParams: Promise<SearchParams> }) {
  return <ResultList results={await getResults(searchParams)} searchParams={await searchParams} />;
}
```

- Make each island an **`async` Server Component** so Suspense actually suspends and streams.
- The fallback should mirror the real layout's footprint — reuse the route's existing skeleton.
- **Cache the data read** (`getItems()`) so N islands calling `getResults` don't cause N fetches.

**On re-keying the boundary.** `<Suspense key={JSON.stringify(searchParams)}>` forces the skeleton to re-show on every filter change. Use it when results are slow and a stale grid would mislead. Skip it when the data is cached and fast — re-keying makes back-navigation flash a full skeleton for content that was already there. Prefer the static-shell split above; add the key only if a stale-content window is actually visible.

### Parsing + query logic in a plain server module

Put pure functions (`parseFilters`, `parseSort`, `parsePage`, `filterItems`, derived option lists, active-filter count) in `lib/<feature>.ts`. No `"use client"`, no React import. The orchestrator, the streamed islands, and the client filter panel all import them, so the URL contract is defined exactly once.

```ts
// lib/<feature>.ts
export type SearchParams = Record<string, string | string[] | undefined>;
export const PAGE_SIZE = 6;

const first = (v: string | string[] | undefined) => (Array.isArray(v) ? v[0] : v);

export function parseFilters(sp: SearchParams): Filters {
  const tags = first(sp.tags);
  return {
    q: first(sp.q) ?? DEFAULT_FILTERS.q,
    type: first(sp.type) ?? DEFAULT_FILTERS.type,
    minPrice: first(sp.minPrice) ?? "",
    tags: tags ? tags.split(",").filter(Boolean) : [],
  };
}

// Validate against a known set — never trust a raw param into a sort/filter.
const SORTS: SortKey[] = ["featured", "newest", "low", "high"];
export function parseSort(sp: SearchParams): SortKey {
  const s = first(sp.sort) as SortKey | undefined;
  return s && SORTS.includes(s) ? s : "featured";
}

export function parsePage(sp: SearchParams): number {
  const p = Number(first(sp.page));
  return Number.isFinite(p) && p >= 1 ? Math.floor(p) : 1;
}
```

Multi-value params ride as one comma-joined key (`tags=a,b,c`), not repeated keys — it keeps URLs short and the parse trivial.

### Client islands write to the URL

Centralize query-string writes in one client hook so islands stay tiny.

```tsx
// components/use-filter-nav.ts
"use client";

import { useCallback } from "react";
import { usePathname, useRouter, useSearchParams } from "next/navigation";

export function useFilterNav() {
  const router = useRouter();
  const pathname = usePathname();
  const searchParams = useSearchParams();

  const setParams = useCallback(
    (patch: Record<string, string | null>, opts?: { resetPage?: boolean }) => {
      const params = new URLSearchParams(searchParams.toString());
      for (const [key, value] of Object.entries(patch)) {
        if (value === null || value === "") params.delete(key);
        else params.set(key, value);
      }
      // Any filter/sort change invalidates the current page unless told otherwise.
      if (opts?.resetPage !== false) params.delete("page");
      const qs = params.toString();
      router.push(qs ? `${pathname}?${qs}` : pathname, { scroll: false });
    },
    [router, pathname, searchParams]
  );

  const reset = useCallback(() => router.push(pathname, { scroll: false }), [router, pathname]);

  return { searchParams, setParams, reset };
}
```

- `{ scroll: false }` so filter changes don't jump the viewport.
- **Drop a param when it equals its default** (`type=All`, `sort=featured`, `page=1`) — pass `null` — to keep URLs clean and canonical.
- The client panel derives its current values by reusing the same parser: `parseFilters(Object.fromEntries(searchParams.entries()))`, memoized on `searchParams`.

### Debounce text inputs before writing the URL

Writing on every keystroke pushes a history entry per character. Keep a local controlled value, push after ~350ms, and resync when the URL changes underneath (reset button, back/forward).

```tsx
"use client";

function useDebouncedTextParam(
  key: string,
  urlValue: string,
  setParams: (patch: Record<string, string | null>) => void
) {
  const [value, setValue] = React.useState(urlValue);

  React.useEffect(() => {
    // Deliberate resync: the URL changed underneath us (reset button,
    // back/forward). Repos with `react-hooks/set-state-in-effect` enabled
    // need the suppression — this is the exception the rule is about.
    // eslint-disable-next-line react-hooks/set-state-in-effect
    setValue(urlValue);
  }, [urlValue]);

  React.useEffect(() => {
    if (value === urlValue) return;
    const t = setTimeout(() => setParams({ [key]: value || null }), 350);
    return () => clearTimeout(t);
  }, [value, urlValue, key, setParams]);

  return [value, setValue] as const;
}
```

Use it for the search box and any numeric min/max fields. Selects, chips, and toggles write immediately — there's nothing to debounce.

### Pagination: real hrefs plus an intercepting onClick

Render genuine links so pages are crawlable and middle-clickable, then `preventDefault` for the in-app route.

```tsx
"use client";

const hrefFor = (p: number) => {
  const params = new URLSearchParams(searchParams.toString());
  if (p <= 1) params.delete("page");
  else params.set("page", String(p));
  const qs = params.toString();
  return qs ? `?${qs}` : "?";
};

const go = (p: number) => {
  const target = Math.min(Math.max(p, 1), totalPages);
  // resetPage: false — this *is* the page change; don't wipe it.
  setParams({ page: target <= 1 ? null : String(target) }, { resetPage: false });
  window.scrollTo({ top: 0, behavior: "smooth" });
};

<PaginationLink href={hrefFor(n)} isActive={n === page}
  onClick={(e) => { e.preventDefault(); go(n); }}>{n}</PaginationLink>
```

Clamp the page against `totalPages` on the **server** too, so a hand-typed `?page=999` renders the last page instead of an empty grid:

```tsx
const totalPages = Math.max(1, Math.ceil(results.length / PAGE_SIZE));
const page = Math.min(parsePage(searchParams), totalPages);
const pageResults = results.slice((page - 1) * PAGE_SIZE, (page - 1) * PAGE_SIZE + PAGE_SIZE);
```

### Keep clickable cards on the server (stretched link)

A whole-card link normally forces `useRouter` and `"use client"`. Instead, keep the card a Server Component and overlay a stretched `<Link>`; raise interactive islands above it.

```tsx
// item-card.tsx — Server Component
<Card className="group relative h-full overflow-hidden">
  {/* Only positioned z-10 element → sits above the static (z auto) media
      and text, so clicking anywhere on those navigates. */}
  <Link href={href} aria-label={item.title} className="absolute inset-0 z-10 focus-ring" />

  <div className="relative aspect-[16/9] overflow-hidden">
    <Image src={item.image} alt={item.title} fill sizes="(min-width:1024px) 33vw, 100vw" />
    <SaveButton id={item.id} />          {/* client island, absolute z-20 */}
  </div>

  <div className="p-4">{/* …static content… */}</div>
</Card>
```

- Interactive islands get `z-20` and `e.preventDefault(); e.stopPropagation()` so their clicks stay local.
- The anchor gives native keyboard focus and Enter for free — no `tabIndex`/`onKeyDown`.
- `h-full` on the card so it stretches inside a grid row or carousel slide.
- If the card needs a *second* mode that isn't navigation (a selection toggle), swap the stretched `<Link>` for a stretched `<button>` behind an optional function-valued prop. Server call sites simply omit the prop and keep the plain link card.

### Self-contained client islands (don't thread state props)

Shrink the client boundary by giving an island the minimal identifier and letting it own its own state/hook, instead of threading `value`/`onChange` from a client parent.

```tsx
// save-button.tsx
"use client";

export function SaveButton({ id }: { id: string }) {
  const { isSaved, toggleSave } = useSaved();   // owns its own state
  const saved = isSaved(id);
  return (
    <Button
      size="icon"
      onClick={(e) => {
        e.preventDefault();                      // above a stretched link —
        e.stopPropagation();                     // keep the click local
        toggleSave(id);
      }}
      className={cn("absolute top-3 right-3 z-20 w-9 h-9", saved && "bg-primary text-primary-foreground")}
      aria-label={saved ? "Remove from saved" : "Save"}
    >
      <Heart size={18} />
    </Button>
  );
}
```

Taking only `id` means the card — and every list around it — stays server-renderable and prop-free. It also removes the client parent that would otherwise exist purely to fan `saved`/`onToggleSave` down the tree.

## Refactor checklist (client-heavy → server-first)

1. **Find the real interactivity.** List every hook, handler, and browser API in the component. Everything else can be server.
2. **Lift view state into the URL.** Replace `useState` for filters/sort/page with `searchParams` reads (server) + a client nav hook (writes).
3. **Move pure logic to `lib/<feature>.ts`** (parse + filter + derive). Import it from Server Components.
4. **Split the leaves into client islands** — one per interactive concern (filter panel, sort menu, pagination, empty-state reset, save button). One component per file.
5. **Make the URL-dependent regions `async` Server Components** and wrap each in `<Suspense>` with a skeleton fallback; keep the shell free of `searchParams`.
6. **Convert the orchestrator + list item to Server Components** — stretched link for navigation, islands for the interactive bits.
7. **Keep `page.tsx` thin** — `await searchParams`, render the orchestrator.
8. **Verify** with the repo's build and lint commands (the package manager is whichever lockfile is present: `pnpm-lock.yaml` → `pnpm`, `package-lock.json` → `npm run`, `yarn.lock` → `yarn`, `bun.lockb` → `bun run`). The route should report Partial Prerender `◐` — static shell plus streamed dynamic content.

## Anti-patterns

- ❌ `"use client"` at the top of a page, orchestrator, or list to satisfy one button. Extract the button.
- ❌ Mirroring URL state into `useState` and syncing with effects. Read `searchParams` directly on the server.
- ❌ Whole-card `onClick={() => router.push()}`. Use a stretched `<Link>`.
- ❌ Threading `value`/`onToggle` callbacks down to a leaf that could own its own hook.
- ❌ `<Suspense>` around a synchronous component — it never suspends, so the fallback never shows.
- ❌ Awaiting `searchParams` in the orchestrator when only two islands need it — that makes the entire shell dynamic.
- ❌ Writing the URL on every keystroke. Debounce text inputs.
- ❌ Trusting a raw param (`sort`, `page`) without validating against a known set / clamping to range.

## Definition of done

- The route builds as Partial Prerender (`◐`) — static shell, dynamic regions streamed.
- `"use client"` appears only on true leaf islands; orchestrator, list, and list item are Server Components.
- Filter/sort/pagination state lives entirely in the URL; no `useState` mirrors it.
- Pure parse/filter logic sits in `lib/<feature>.ts` with no React import.
- Params are validated/clamped server-side; defaults are omitted from the URL.
- Build and lint pass.

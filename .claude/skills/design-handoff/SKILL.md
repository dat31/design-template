---
name: design-handoff
description: Implement a design (Claude Design output, Figma, or mockup) into pages in a Next.js App Router + Tailwind v4 + shadcn/ui repo. Use when the user asks to build/implement a screen, page, or UI from a design, or to translate a mockup into code. Enforces font/Tailwind/component-reuse constraints, zod + react-hook-form conventions, theme tokens, and the ordered scan→tokens→components→pages workflow.
---

# Design → Code Handoff

Rules for turning a design into implemented pages. Follow fully before writing UI code.

This skill is written to be repo-agnostic: it assumes the **Next.js App Router + Tailwind CSS v4 + shadcn/ui** stack but discovers the specifics (fonts, primitives, tokens, package manager) from the repo it is running in. Do step 0 first — never assume the values from another project.

## 0. Orient before you write anything

Run these and read the results. They replace every hardcoded assumption this skill could otherwise make.

```bash
# Which primitives already exist — this is the allowed vocabulary (step 3 below).
ls components/ui/

# The token system: @theme inline block, :root / .dark values, custom variants.
cat app/globals.css

# Which fonts are registered and under which CSS variable names.
cat app/layout.tsx

# Dependencies actually installed (forms? zod? animation? icons?).
cat package.json

# Package manager — pick the one whose lockfile is present.
ls pnpm-lock.yaml package-lock.json yarn.lock bun.lockb 2>/dev/null
```

Throughout this skill, `<pm>` means the package manager implied by the lockfile:
`pnpm-lock.yaml` → `pnpm` · `package-lock.json` → `npm run` · `yarn.lock` → `yarn` · `bun.lockb` → `bun run`.
Use `<pm> dlx` / `npx` / `bunx` correspondingly for one-off tools.

## Stack (do not change)

- **Next.js App Router** (`/app`), React 19, TypeScript.
- **Tailwind CSS v4** — theme is CSS-first in `app/globals.css` via `@theme inline`. There is no `tailwind.config.js`; do not add one.
- **shadcn/ui** primitives in `components/ui/` — built with `class-variance-authority` (CVA) + `radix-ui` + a `cn()` helper in `lib/utils.ts`.
- **Fonts via `next/font/google` only**, wired in `app/layout.tsx` as CSS variables. Whatever font the repo ships with is a *default*, not a requirement — swap it for whatever matches the design.
- **Forms**: `react-hook-form` + `@hookform/resolvers` + `zod`. Validation lives in zod schemas.

## Strict constraints

1. **Fonts** — only `next/font/google`. Pick the font(s) that match the design. Register each in `app/layout.tsx` as a `--font-*` variable, apply the variable class to `<html>`, and map it to a token inside `@theme inline` (e.g. `--font-sans`, `--font-heading`, `--font-mono`). Never hardcode a `font-family` string and never add a `<link>` tag or `@import` for a font.

2. **Tailwind first** — style with Tailwind utility classes driven by theme tokens; avoid inline `style={}` for anything a token can express. Tailwind does not have to be 100%: for CSS it can't express cleanly (complex `@keyframes`, `@supports`, intricate gradients), add the rule to a `.css` file (`globals.css` or a co-located module) and reference it by class. Reach for raw CSS only when Tailwind genuinely can't do it.

3. **Reuse existing components** — every UI element MUST map to an existing component in `components/ui/`. To match the design:
   - ✅ Update `className` on an instance.
   - ✅ Add or adjust a CVA **variant** / **size** inside the component file.
   - ✅ Update token values in `globals.css`.
   - ❌ Do NOT delete, rename, or invent sub-components.
   - ❌ Do NOT change a component's sub-component hierarchy or `data-slot` structure (e.g. `Card` → `CardHeader`/`CardContent`/`CardFooter` stays intact).
   - If the design needs an element with no matching primitive, **compose it from existing primitives** — do not fork shadcn structure.
   - If the design needs a primitive the repo genuinely doesn't have yet, add it with `<pm> dlx shadcn@latest add <name>` rather than hand-writing it, then style it through tokens and variants as above.

   The authoritative list of what's available is `ls components/ui/` from step 0 — not a list memorized from another project.

## Coding conventions

- **One component per file.** No co-locating multiple exported components in a single file.
- **Reusable types via zod** — define a `z.object(...)` schema and derive the type with `z.infer<typeof Schema>`. Don't write a standalone `interface`/`type` for data that has a schema.
- **Forms** — `useForm({ resolver: zodResolver(schema) })`, rendered with the `field` primitives from `components/ui/field.tsx` when present, otherwise `label` + `input` + the repo's error styling.
- **`cn()`** for all conditional/merged classNames. Keep the CVA pattern (`data-slot`, `data-variant`, `data-size`) when extending a component.
- **Imports** use the `@/` alias.
- **Navigation** — anything that navigates to a route renders a real `<Link>` (`next/link`), not an `onClick` with `router.push`. When a link should look like a button, use `<Button asChild><Link href="…">…</Link></Button>`. Reserve `router.push`/`useRouter` for imperative navigation that can't be expressed as a link (after a form submit or async action). Don't pass `onPreview`/`onEdit`-style navigation callbacks down to children — give the child the data it needs and let it build its own `<Link href>`.

## Server-first rendering (RSC)

Render on the server by default; keep `"use client"` confined to leaf interactivity. This is a hard requirement for new pages — **see the [server-first-rendering](../server-first-rendering/SKILL.md) skill for the full patterns** and apply it whenever a screen has filtering, sorting, pagination, or any "mostly static with one interactive bit" component.

What to enforce here:

- **Server by default.** `page.tsx`, layout/orchestration, lists, and list items are Server Components. Add `"use client"` only at the leaf that needs a hook, an event handler, or a browser API — then push that boundary *down* (extract the interactive bit) rather than marking the whole parent client.
- **URL is the source of truth** for filter/sort/pagination state — read `searchParams` on the server, write it from small client islands; don't mirror it in `useState`.
- **Stream the dynamic region** in `<Suspense>` with a skeleton fallback, and make the streamed part an `async` Server Component.
- **Keep clickable cards server-side** with a stretched `<Link className="absolute inset-0 z-10">`; raise interactive islands above it (`z-20` + `preventDefault`).
- Pure parse/filter logic goes in `lib/<feature>.ts` with no React import.

## Route / file structure

Co-locate everything a route owns under its folder:

```
app/
  <route>/                        # or (group)/<route>/ when the repo uses route groups
    page.tsx                      # page composition only
    loading.tsx                   # route skeleton (reuse as the Suspense fallback)
    components/<feature>.tsx      # one component per file
    constants/<feature>.ts        # static data, options, copy
    schemas/<feature>.ts          # zod schemas + inferred types
    lib/<feature>.ts              # pure server logic (parse/filter/derive), no React
    hooks/use-<feature>.ts        # route-local client hooks (optional)
```

- **Match the repo's existing conventions** over the sketch above: if routes already sit in groups like `(app)` / `(auth)`, put new pages in the group whose layout they need; if component files are already PascalCase, stay PascalCase (non-component modules stay lowercase either way).
- Shared, cross-route primitives stay in `components/ui/`; app-wide shared components in `components/` or `app/components/`, following whatever the repo already does. Shared helpers in `lib/`.

## Theme tokens

Tokens are CSS variables defined in `:root` and `.dark`, then exposed to Tailwind through `@theme inline` in `app/globals.css`. Colors are in **OKLCH**.

The shadcn token contract — expect these, and set them rather than inventing parallel names:

| Group | Tokens |
|---|---|
| Surface | `--background`, `--foreground`, `--card`, `--card-foreground`, `--popover`, `--popover-foreground` |
| Intent | `--primary`, `--secondary`, `--muted`, `--accent`, `--destructive` (each with a `-foreground` pair) |
| Chrome | `--border`, `--input`, `--ring` |
| Charts | `--chart-1` … `--chart-5` |
| Sidebar | `--sidebar`, `--sidebar-foreground`, `--sidebar-primary`, `--sidebar-accent`, `--sidebar-border`, `--sidebar-ring` |
| Shape | `--radius` (+ derived `--radius-sm` … `--radius-4xl` in `@theme inline`) |
| Type | `--font-sans`, `--font-mono`, `--font-heading` |

Rules:

- When the design introduces a new color, spacing, or radius: **add or update a token, then reference it.** Never hardcode a raw hex/oklch value in a `className`.
- **Keep `:root` and `.dark` in sync** — every token defined in one must exist in the other.
- Derived radii come from a single `--radius`; change that one value rather than each step, unless the design's scale isn't proportional.
- If the repo carries a **second, brand-level palette** on top of the shadcn layer (its own `--color-*` names in `@theme inline`), use the brand layer for page-level UI and set the shadcn layer *from* brand values — never introduce a third palette.
- If the repo makes any token **runtime-swappable** (e.g. redefined under `[data-theme="…"]` / `[data-accent="…"]` and driven by `next-themes`), anything colored by it must go through that variable so it swaps, and every new derived color must be added to **every** variant block so they stay in sync.

## Implementation steps (follow in order)

1. **Scan the design** — inventory screens, sections, repeated patterns, states (hover / focus / disabled / loading / empty / error), assets (images / icons / SVGs), and which existing primitive each element maps to. Name any gap now, while it's cheap.
2. **Update theme tokens** — colors, radius, spacing, typography in `globals.css` (light **and** dark, plus every theme-variant block). Wire any new font in `layout.tsx`. Put static assets in `public/` (or inline SVGs); use `next/image` for raster images.
3. **Update component styles** — adjust `className`s and add/extend CVA variants in `components/ui/` so primitives match the design. Preserve sub-component hierarchy and `data-slot`s.
4. **Define schemas + reusable components** — write the zod schemas under `schemas/` first (derive types with `z.infer`), then build feature components from primitives under the route's `components/`, one per file.
5. **Implement pages** — compose feature components in `page.tsx`. Keep page files thin. Cover every responsive breakpoint and every interaction state inventoried in step 1. Apply **server-first rendering**: Server Components by default, `"use client"` only on leaf islands, view state in the URL, dynamic regions streamed via `<Suspense>`.
6. **Verify** — run `<pm> build` and `<pm> lint`; fix until both pass.

## Definition of done

- `<pm> build` and `<pm> lint` pass.
- No new fonts, colors, or radii outside the token system; `:root` and `.dark` (and any theme-variant blocks) are in sync.
- No new or removed sub-components in `components/ui/`; hierarchy and `data-slot`s intact.
- All forms validate through zod schemas; all reusable data types derive from zod.
- One component per file; route-local files follow the structure above.
- Server-first: `"use client"` only on leaf islands; orchestrator, list, and list item are Server Components; filter/sort/pagination state lives in the URL. (See [server-first-rendering](../server-first-rendering/SKILL.md).)

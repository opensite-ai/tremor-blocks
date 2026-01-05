## AGENTS.md – Instructions for coding agents working on `tremor-blocks`

This file is for **AI coding agents and maintainers**. It encodes the non‑obvious
rules, constraints, and workflows of this repo so automated changes stay fast,
safe, and consistent.

When in doubt, favor: **(1) block & component API stability, (2) visual
correctness of data visualizations, (3) performance & bundle size**, in that
order.

---

## 1. Quick mental model of this repo

- This is a **Next.js 14 app** that packages a **curated catalog of Tremor‑style
  data‑viz components and higher‑level dashboard blocks**.
- It is consumed by the DashTrack ecosystem (e.g. `dt-cms/Source`) as the
  **source of truth for professionally designed analytics UI**.
- There are three important layers:
  - **Primitives** – Reusable chart & UI components under `src/components/*`
    and helpers in `src/lib/*` (e.g. `chartUtils`, `cx`, `useOnWindowResize`).
  - **Block content system** – Category + block metadata, React wrappers, and
    MDX documentation in `src/content/*`.
  - **Gallery app** – Next.js pages and UI shell in `src/app/*` that render the
    catalog for humans and agents to browse.
- Treat **IDs, categories, and block wiring as an external API** used by other
  codebases. Changes here can break dashboards that reference specific blocks.

Non‑goals:

- This is **not a generic product app**; avoid adding app‑specific behavior or
  business logic. Keep it focused on reusable visual building blocks.

---

## 2. Golden rules (must follow)

1. **Do not break block IDs or categories.**
   - `blocksMetadata` IDs, `categoryIds`, and keys in `blocksComponents` are
     treated as stable. Avoid renaming or deleting them; prefer adding new
     entries instead.
   - If a breaking change is truly necessary, coordinate it with all consuming
     apps and update MDX docs.

2. **Keep primitives stable and framework‑agnostic.**
   - Files in `src/components/*` and `src/lib/*` should stay **pure React +
     Tailwind** and avoid importing from `next/*` or app‑specific modules.
   - Public props on these components form a de‑facto API; change them
     conservatively.

3. **Preserve the blocks data model.**
   - Every block **ID** must be wired consistently across:
     - `src/content/blocks-metadata.ts` (`id`, `category`, `name`)
     - `src/content/blocks-components.ts` (ID → React component)
     - `src/content/markdown/<category>/<id>.mdx` (code & docs)
   - If you add or remove a block, update **all three places**.

4. **Use the shared utilities.**
   - Use `cx(...)` from `src/lib/utils.ts` for Tailwind class merging instead of
     manually combining `clsx` + `twMerge`.
   - For charts, reuse helpers from `src/lib/chartUtils.ts`
     (`AvailableChartColors`, `constructCategoryColors`, `getColorClassName`,
     `getYAxisDomain`, `hasOnlyOneValueForKey`) rather than re‑implementing
     color or domain logic.

5. **Respect client components and rendering boundaries.**
   - Most primitives and blocks are **client components** (`'use client'`).
   - Do not introduce Node‑only APIs or blocking I/O inside these components.
   - File‑system and MDX loading should stay in `src/app/db/blocks.ts` and
     other server‑side utilities.

6. **Keep dependencies lean.**
   - Preferred stack: **Next.js + React 18, TailwindCSS, Recharts, Framer
     Motion** and existing Radix + Headless UI integrations.
   - Avoid adding heavy new runtime dependencies without a strong reason and
     explicit approval.

7. **Prefer copying a proven pattern.**
   - When adding a component, block, or category, start from the **closest
     existing example** and adapt it instead of inventing a new structure.

8. **Always run minimal checks for code changes.**
   - At minimum, for non‑trivial changes:
     - `pnpm build` (or `npm run build`) – verify the app compiles.
     - Optionally `pnpm dev` – manual smoke test of affected blocks.
   - Do **not** claim checks passed unless you actually ran them.

---

## 3. Key directories (mental map)

- `src/components/*`
  - Tremor‑style **UI primitives and chart components** (e.g. `AreaChart`,
    `BarChart`, `Table`, `Tabs`, `Input`).
  - Used both directly by consuming apps and indirectly by higher‑level blocks
    in `src/content/components/*`.

- `src/lib/*`
  - Small, focused utilities that support primitives:
    - `chartUtils.ts` – color tokens and helpers shared by all charts.
    - `utils.ts` – `cx`, focus ring helpers.
    - `useOnWindowResize.ts`, `useToast.ts` – React hooks for common behavior.

- `src/content/*`
  - **Core blocks system**:
    - `declarations.ts` – shared types and `categoryIds` constants.
    - `blocks-metadata.ts` – flat list of all blocks (IDs, category, name).
    - `blocks-categories.ts` – category metadata, thumbnails, and block counts.
    - `blocks-components.ts` – ID → React component map; imports from
      `src/content/components/*`.
    - `components/*` – actual block implementations, grouped by category
      (area‑charts, tables, logins, billing‑usage, etc.).
    - `markdown/*` – MDX files with code & narrative for each block ID.

- `src/app/*`
  - Next.js **app‑router** implementation for the gallery.
  - Key pieces:
    - `app/(with-app-shell)/blocks/[blocksCategory]/page.tsx` – renders a block
      category page using `getBlocks`.
    - `app/db/blocks.ts` – reads MDX, parses frontmatter, joins with
      `blocksMetadata`/`blocksCategoriesMetadata`.
    - `app/ui/*` – gallery UI (layout, header, `Block`, `BlocksPreview`, tiles).

- `public/thumbnails/*`
  - Category thumbnails referenced from `blocks-categories.ts` and used in
    `BlocksGridTile` cards.

- `utils/generate-markdown.js`
  - Helper script used by `pnpm markdown` to generate markdown/MDX from
    `src/content/components`. Run it when you change block components heavily.

---

## 4. Blocks system: data model & invariants

The blocks system is **data‑driven**. Understanding how pieces fit together is
critical for safe edits.

1. **Categories**
   - Defined by `categoryIds` in `src/content/declarations.ts`.
   - Display metadata lives in `preblocksCategoriesMetadata` within
     `src/content/blocks-categories.ts` (name, thumbnail, `hasCharts`).
   - `blocks-categories.ts` derives `blocksCategoriesMetadata` by counting how
     many blocks in each category exist in `blocksMetadata`.

2. **Blocks metadata** (`blocks-metadata.ts`)
   - `blocksMetadata: BlocksMetadata[]` is the canonical list of all blocks.
   - Each entry has `{ id, category, name }` where:
     - `id` is a **stable unique string** (`filterbar-01`, `bar-chart-03`, …).
     - `category` is one of the `categoryIds` values.
     - `name` is human‑readable and shown in the gallery.

3. **React components** (`blocks-components.ts` + `content/components/*`)
   - `blocks-components.ts` maps each `id` to a React component imported from
     `src/content/components` (via `content/components/index.ts`).
   - Components are grouped by category; e.g. `content/components/area-charts`
     exports `AreaChart01..16`, which are then wired to IDs
     `area-chart-01..16`.

4. **MDX documentation** (`content/markdown/*`)
   - For each block ID there is an MDX file:
     - `src/content/markdown/<category>/<id>.mdx`
   - `app/db/blocks.ts` reads these files, parses optional frontmatter, and
     feeds their content into the gallery via `CustomMDX`.

5. **Runtime usage**
   - `getBlocks({ blocksCategory })` in `app/db/blocks.ts` joins:
     - `blocksCategoriesMetadata` (for the category name) and
     - `blocksMetadata` + MDX content (for each block).
   - `BlocksPreview` receives a `blocksId` and `blocksCategory` and looks up the
     React implementation via `blocksComponents[blocksId]`.

**Invariants to preserve:**

- Every `blocksMetadata.id` must:
  - exist exactly once in the `blocksMetadata` array,
  - appear as a key in `blocksComponents`, and
  - have a matching MDX file under `content/markdown/<category>/<id>.mdx`.
- `blocksCategoriesMetadata.count` must match the number of blocks per
  category (it is recomputed automatically; do not hard‑code counts).

---

## 5. Working with primitives (`src/components/*`)

See `src/components/AGENTS.md` for details. High‑level expectations:

- Components are **client‑side React components** that power many blocks.
- They rely heavily on:
  - `src/lib/utils.ts` (`cx`, focus helpers)
  - `src/lib/chartUtils.ts` for all chart color/domain logic
  - `src/lib/useOnWindowResize.ts` to adapt legends and layouts
- Many files start with a comment like `// Tremor AreaChart [v0.3.1]`. When you
  make significant changes, update the version token to keep provenance clear.
- Prefer adding new props or variants over removing or renaming existing ones.

---

## 6. Working with blocks & content (`src/content/*`)

See `src/content/AGENTS.md` for a deep dive. At a high level:

- Block implementations live in `src/content/components/<category>/*`.
- Each category has an `index.ts` that exports named components
  (`FeatureSection01`, `TableAction03`, etc.).
- `src/content/components/index.ts` re‑exports categories so
  `blocks-components.ts` can import them as `* as components`.
- MDX documentation in `src/content/markdown/*` is the **authoritative example
  code** for each block and should be kept in sync with the React
  implementation.

When adding or modifying blocks, always:

1. Update/extend the React component under `content/components/*`.
2. Wire it in `content/components/<category>/index.ts` (and
   `content/components/index.ts` if adding a new category).
3. Add/update the entry in `blocks-metadata.ts`.
4. Map the ID to the component in `blocks-components.ts`.
5. Add/update the corresponding MDX file under `content/markdown/*`.

---

## 7. Next.js gallery behavior (`src/app/*`)

- The gallery uses the **app router** (`src/app/(with-app-shell)`).
- `src/app/ui/*` contains layout components (header, container, footer) plus:
  - `block.tsx` – wraps a single block: heading, permalink anchor, preview.
  - `blocks-preview.tsx` – renders the block implementation and applies
    category‑specific padding.
  - `blocks-grid-tile.tsx` – one tile in the category grid, with thumbnail and
    block count.
- Category pages under `app/(with-app-shell)/blocks/[blocksCategory]/page.tsx`
  call `getBlocks` and map over `blocks.blocksData`.

**Important:** app‑level changes affect how all blocks are displayed but should
not alter block IDs or internals. Keep gallery behavior generic and styling
focused on presentation, not data logic.

---

## 8. Recommended workflows for agents

### 8.1 Add a new block variant in an existing category

1. Identify the closest existing block in that category under
   `src/content/components/<category>/*` and copy it.
2. Implement the new variant, keeping props and naming consistent.
3. Export it from the category `index.ts` and ensure
   `content/components/index.ts` already exports that category.
4. Add a new entry to `blocks-metadata.ts` with a unique `id`, the correct
   `category`, and a descriptive `name`.
5. Wire the ID to the component in `blocks-components.ts`.
6. Create a matching MDX file under `content/markdown/<category>/<id>.mdx`.
7. Run `pnpm build` and visually verify the new block via `pnpm dev`.

### 8.2 Add a brand‑new category of blocks

1. Add a new key/value to `categoryIds` in `content/declarations.ts`.
2. Add an entry to `preblocksCategoriesMetadata` in `blocks-categories.ts`
   (name, thumbnail path, `hasCharts`).
3. Create `content/components/<category>/` and its `index.ts` exporting
   components.
4. Create `content/markdown/<category>/` and add MDX files per block.
5. Add corresponding entries to `blocks-metadata.ts` and `blocks-components.ts`.
6. Run `pnpm build` and ensure the new category appears in the blocks index.

### 8.3 Adjust a core Tremor primitive

1. Locate the primitive in `src/components/*` and review its usage in
   `src/content/components/*`.
2. Make the smallest change that achieves the goal; avoid breaking props.
3. If you change chart colors or domains, do it through `chartUtils.ts` where
   possible.
4. Update any obviously affected blocks and MDX examples.
5. Run `pnpm build` and manually spot‑check representative blocks.

---

## 9. Verification checklist

Before considering a change “done”, an agent should:

- [ ] Run `pnpm build` (or `npm run build`) to ensure the project compiles.
- [ ] For content or visual changes, run `pnpm dev` and verify affected blocks
      in the browser.
- [ ] If you changed `src/content/components/*` heavily, consider running
      `pnpm markdown` to regenerate any derived markdown.
- [ ] Confirm new/changed blocks preserve the invariants in section 4.
- [ ] Clearly document any skipped verification steps (and why) in your PR or
      change description.

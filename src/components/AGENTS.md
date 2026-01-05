## AGENTS.md  Instructions for coding agents working in `src/components`

This file is for **AI coding agents and maintainers** working on the **Tremor
primitives** under `src/components/*`.

These components are reused across many blocks and dashboards. Treat their
**public props and behavior as an API**.

---

## 1. Mental model

- This directory contains **lowlevel UI and chart primitives**:
  - Charts: `AreaChart`, `BarChart`, `ComboChart`, `LineChart`, `SparkChart`,
    `DonutChart`, `CategoryBar`, `ProgressBar`, `ProgressCircle`, `Tracker`,
    etc.
  - Form/layout primitives: `Button`, `Input`, `Select`, `Checkbox`, `Tabs`,
    `Table`, `Dialog`, `Drawer`, `Toast`, etc.
- Charts are built on **Recharts** and share helpers from `src/lib/chartUtils`.
- Styling is **Tailwind CSS** with class merging via `cx` from `src/lib/utils`.
- Many files carry a provenance comment like `// Tremor AreaChart [v0.3.1]`.
  Keep or update this when you make significant, intentional changes.

---

## 2. Golden rules

1. **Keep components clientsafe.**
   - Most files start with `'use client'`. Do not add Nodeonly APIs or
     synchronous filesystem/network calls.

2. **Use shared utilities.**
   - Use `cx(...)` from `src/lib/utils.ts` for class merging instead of
     mixing `clsx` and `twMerge` manually.
   - For focus/validation styling on inputs, reuse `focusInput`, `focusRing`,
     and `hasErrorInput` from `src/lib/utils.ts`.

3. **Preserve public props.**
   - Assume exported component props are consumed by other repos.
   - Prefer **additive** changes (new optional props, new variants) over
     renaming/removing existing props.

4. **Keep behavior predictable & accessible.**
   - Respect existing keyboard interactions and ARIA attributes, especially for
     dialogs, menus, checkboxes, sliders, and tabs.
   - When using Radix primitives, follow their documented accessibility
     patterns.

5. **Avoid hardcoding theme values when tokens exist.**
   - Reuse Tailwind utility patterns already present in nearby components.
   - If you need new colors for charts, add them via `chartColors` in
     `src/lib/chartUtils.ts` instead of scattering raw classes.

---

## 3. Chart components: key patterns

- **Data model**
  - Chart components usually accept props like:
    - `data`: array of records.
    - `index`: xaxis key.
    - `categories`: series keys.
    - `colors?`: subset of `AvailableChartColorsKeys`.
    - Optional callbacks/value formatters.

- **Color & domain handling** (from `chartUtils.ts`):
  - Use `AvailableChartColors`, `constructCategoryColors`, and
    `getColorClassName` to determine series colors.
  - Use `getYAxisDomain(autoMinValue, minValue, maxValue)` instead of
    inventing your own domain logic.

- **Responsiveness & legends**
  - Components often use `useOnWindowResize` to adapt legends or layouts.
  - Preserve this behavior when refactoring; charts should remain usable on
    narrow viewports.

- **Recharts integration**
  - Keep the separation between data processing (in React) and drawing (in
    Recharts components like `Area`, `Line`, `Tooltip`, `Legend`).
  - Avoid passing unstable inline functions as props inside large lists when a
    memoized helper or small wrapper component is available.

---

## 4. Form & layout primitives

- **General rules**
  - Always forward `className` to the outermost relevant element and merge it
    with defaults using `cx`.
  - Prefer controlled, explicit props over hidden internal state, unless the
    component is clearly selfcontained.

- **Buttons / Inputs / Selects**
  - Keep focus states consistent using `focusInput` / `focusRing` helpers.
  - Ensure disabled and error states are visually distinct and accessible.

- **Dialog / Drawer / Toast / Tooltip**
  - Follow existing Radix usage patterns; these are sensitive to subtle prop
    changes.
  - Do not change trigger or content structure without checking how blocks in
    `src/content/components/*` consume them.

---

## 5. Making changes safely

### 5.1 Adjust an existing primitive

1. Search for usages in this repo (especially under `src/content/components`).
2. Make the **smallest change** that satisfies the requirement.
3. If behavior meaningfully changes, update the leading `// Tremor ... [vX.Y.Z]`
   comment.
4. Run `pnpm build` and visually inspect a few blocks that rely on the
   component.

### 5.2 Add a new primitive

1. Copy the closest existing component (e.g. from another chart or input type).
2. Keep props aligned with existing patterns so blocks can swap between
   primitives easily.
3. Export the new component from this directory and from any barrel file that
   the consuming app expects (outside this repo).
4. If relevant, add or reuse entries in `chartUtils.ts` for colors.
5. Run `pnpm build` and wire the new primitive into a block under
   `src/content/components/*` as a realworld usage test.

---

## 6. Quick verification checklist

- [ ] `pnpm build` succeeds with no TypeScript or Next.js errors.
- [ ] Representative blocks that use the modified primitive render correctly in
      the gallery (`pnpm dev`).
- [ ] Charts still render with sane colors and axes; interactive components
      retain keyboard/mouse accessibility.


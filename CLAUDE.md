# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

ChartDB is a 100% client-side, web-based database diagramming editor (React + TypeScript + Vite). There is no backend — all diagrams persist locally in IndexedDB via Dexie. Optional AI features hit OpenAI (or a compatible endpoint) directly from the browser.

Node version: **v24** (`.nvmrc`). License: **AGPL-3.0**.

## Commands

```bash
npm run dev            # Vite dev server (default: http://localhost:5173)
npm run build          # Runs lint, then `tsc -b`, then `vite build` — lint failures break the build
npm run lint           # ESLint with --max-warnings 0
npm run lint:fix       # ESLint --fix
npm test               # Vitest in watch mode
npm run test:ci        # Single run, verbose, --bail=1
npm run test:ui        # Vitest UI
```

Run a single test file or pattern:

```bash
npx vitest run src/lib/dbml/dbml-import/__tests__/dbml-import.test.ts
npx vitest run -t "name of the test"
```

A Husky `pre-commit` hook runs `npm run lint`.

## Environment variables

AI features are optional and configured at **build time** (Vite `VITE_*`) or, for the Docker image, **runtime** (the entrypoint script rewrites them into `window.env`):

- `VITE_OPENAI_API_KEY` — direct OpenAI usage.
- `VITE_OPENAI_API_ENDPOINT` + `VITE_LLM_MODEL_NAME` — custom OpenAI-compatible endpoint (e.g. vLLM). Use one option or the other, not both.
- `VITE_HIDE_CHARTDB_CLOUD`, `VITE_DISABLE_ANALYTICS` — privacy/branding flags. Runtime overrides via `window.env` are honored (see `src/lib/env.ts`).

## Architecture

### Entry and routing
`src/main.tsx` → `src/app.tsx` (wraps `HelmetProvider`, `TooltipProvider`, `RouterProvider`) → `src/router.tsx`. All page routes are **lazy-loaded** via React Router's `lazy()` so each page is its own chunk. The editor (`/` and `/diagrams/:diagramId`) is the primary route; templates and examples are supporting routes.

### Provider stack (editor page)
`src/pages/editor-page/editor-page.tsx` composes the runtime — many providers wrap the canvas, in this rough order: `StorageProvider` → `ConfigProvider` → `LocalConfigProvider` → `RedoUndoStackProvider` → `ChartDBProvider` → `HistoryProvider` → `ThemeProvider` → `ReactFlowProvider` → `ExportImageProvider` → `DialogProvider` → `KeyboardShortcutsProvider` → `AlertProvider` → `CanvasProvider` → `DiffProvider` → `DiagramFilterProvider`. New global features generally mean adding/extending a context under `src/context/`.

### Diagram state — `ChartDBProvider`
`src/context/chartdb-context/` holds the live diagram in memory (tables, relationships, dependencies, areas, custom types, notes, schemas). It exposes mutation methods and an `EventEmitter` (`events`) that emits typed `ChartDBEvent`s (`add_tables`, `update_table`, `remove_tables`, `add_field`, `remove_field`, `load_diagram`). Subscribers (canvas nodes, side panels) listen on this emitter rather than re-rendering off context — this is the cross-component update channel.

### Persistence — `StorageProvider`
`src/context/storage-context/storage-provider.tsx` defines a Dexie database called `ChartDB` with one `EntityTable` per domain object (`diagrams`, `db_tables`, `db_relationships`, `db_dependencies`, `areas`, `db_custom_types`, `notes`, `config`, plus diagram filters). All persistence flows through `StorageContext` — do not touch Dexie directly elsewhere. `listDiagrams`/`getDiagram` take `include*` flags to opt into joining child entities.

### Domain model
`src/lib/domain/` holds all entity types and **Zod schemas** (`diagram.ts`, `db-table.ts`, `db-field.ts`, `db-relationship.ts`, `db-index.ts`, `db-check-constraint.ts`, `db-dependency.ts`, `db-custom-type.ts`, `db-schema.ts`, `area.ts`, `note.ts`, `database-type.ts`, `database-edition.ts`). When adding domain fields, update both the TS interface and the Zod schema — schemas are used to validate imported diagram JSON.

### Canvas
`src/pages/editor-page/canvas/` is built on `@xyflow/react`. Custom node/edge types live in their own folders (`table-node`, `area-node`, `note-node`, `relationship-edge`, `dependency-edge`, `connection-line`, `temp-floating-edge`, `temp-cursor-node`, `create-relationship-node`). Toolbar and context menu sit alongside. Filtering (which schemas/tables to show) is driven by `DiagramFilter` from `src/lib/domain/diagram-filter/`.

### SQL & DBML import/export
- **SQL import** per dialect: `src/lib/data/sql-import/dialect-importers/{postgresql,mysql,sqlserver,sqlite,oracle}/`. Each dialect exposes a parser that walks the AST from `node-sql-parser` (or a dialect-specific parser) and produces domain objects. `src/lib/data/sql-import/sql-validator.ts` and `validators/` validate before importing.
- **SQL export**: `src/lib/data/sql-export/export-sql-script.ts` is the entry point; per-type generators in `export-per-type/`; cross-dialect normalization in `cross-dialect/`. Results are cached via `export-sql-cache.ts`.
- **Database metadata import** (the "Smart Query" JSON path): `src/lib/data/import-metadata/`. The magic queries themselves live under `scripts/` and produce a JSON blob the importer consumes.
- **DBML**: `src/lib/dbml/` has `dbml-import`, `dbml-export`, and `apply-dbml` (applies DBML edits onto an existing diagram). Backed by `@dbml/core` and `@dbml/parse`.

### i18n
`src/i18n/i18n.ts` initializes i18next with browser language detection. Translation bundles live in `src/i18n/locales/`. When adding UI strings, add a key to every locale — do not hardcode user-facing strings.

### Templates
`src/templates-data/` holds template metadata; `src/pages/templates-page/`, `template-page/`, and `clone-template-page/` render them. Routes preload data via React Router `loader`s.

## Conventions

### Code style — strictly enforced by ESLint and the build
- **`import type` is mandatory** for type-only imports (`@typescript-eslint/consistent-type-imports: error`). Mixing values and types in one import will fail lint and break the build.
- Prettier: **4-space indent**, single quotes, semicolons, `trailingComma: "es5"`.
- React + Hooks + JSX-a11y + Tailwind ESLint plugins are enabled; `eslint-plugin-tailwindcss` will flag unknown / out-of-order class names.
- Path alias `@/...` → `./src/...` (configured in both `vite.config.ts` and `vitest.config.ts`).

### Vite build quirk
`vite.config.ts` marks any module matching `/__test__/` as external — test files are colocated under `__tests__/` folders so they're stripped from the production bundle. Don't put non-test code under a `__tests__` path.

### Tests
Vitest with `happy-dom` and `@testing-library/react`. Setup file: `src/test/setup.ts` (extends `expect` with jest-dom matchers and auto-`cleanup`s after each test). Tests live in `__tests__/` folders next to the code they cover. Most existing tests target the import/export pipelines (SQL dialects, DBML, check-constraint parsing) — they're the high-value surface to add to when changing those areas.

### When adding a database dialect
Touch all of: `src/lib/domain/database-type.ts` (enum), `src/lib/domain/database-edition.ts` if applicable, `src/lib/data/sql-import/dialect-importers/<dialect>/`, `src/lib/data/sql-export/` (per-type generators), `src/lib/data/import-metadata/scripts/` (the magic query), and `src/lib/databases.ts` (display metadata). The `database-capabilities.ts` file gates feature availability per dialect — check it when a feature should be dialect-specific.

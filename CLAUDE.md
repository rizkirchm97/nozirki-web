# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
bun run dev      # Start dev server at http://localhost:3000
bun run build    # Production build
bun run start    # Start production server
bun run lint     # Run ESLint
```

Use `bun` as the package manager (bun.lock is present).

## Architecture

Next.js 16 app using the **App Router** with React 19 Server Components. All files under `app/` are Server Components by default — add `'use client'` only when browser APIs or interactivity are needed.

- `app/layout.tsx` — Root layout (Geist fonts, global metadata)
- `app/page.tsx` — Home page
- `app/globals.css` — Global styles with Tailwind v4 directives and CSS variable theme tokens

**Path alias:** `@/` maps to the project root (e.g., `import { Foo } from '@/components/Foo'`).

## Styling

Tailwind CSS v4 via PostCSS. Theme tokens are defined as CSS variables in `app/globals.css` under `@theme inline`. Dark mode is handled via `prefers-color-scheme` media query, not a class toggle.

## Skill Guides

The `.claude/` directory contains 12 topic-specific `SKILL.md` guides covering advanced patterns for this stack:

- `nextjs-core/` — App Router, RSC, rendering strategies (SSR/SSG/ISR/CSR)
- `nextjs-architecture/` — Feature-based structure, TypeScript patterns, Zod, BFF pattern
- `nextjs-api-integration/` — API routes, fetch patterns, error handling
- `nextjs-auth-security/` — Auth, session management, security
- `nextjs-performance/` — Image/font optimization, bundle splitting, caching
- `nextjs-state-caching/` — Zustand, Context, SWR, server-side caching
- `nextjs-testing-devops/` — Vitest, React Testing Library, Playwright, CI/CD
- `design-system-components/` — shadcn/ui, Radix UI, component patterns
- `tailwind-mastery/` — Design tokens, CVA, advanced variants (group, peer, has)
- `animation-motion/` — Framer Motion, CSS animations
- `css-layout/` — Flexbox, Grid, container queries
- `accessibility-a11y/` — WCAG 2.1, ARIA, keyboard navigation

Consult these guides when implementing features in their respective domains.

## TypeScript

Strict mode is enabled. `noEmit: true` means TypeScript only type-checks — Next.js handles compilation. Module resolution is set to `bundler`.

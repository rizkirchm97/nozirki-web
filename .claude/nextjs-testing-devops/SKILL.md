---
name: nextjs-testing-devops
description: >
  Panduan testing dan DevOps untuk Next.js: unit testing dengan Vitest/Jest, component testing
  dengan React Testing Library, E2E testing dengan Playwright, CI/CD dengan GitHub Actions,
  deployment ke Vercel atau self-hosted Docker, dan observability dengan Sentry. Gunakan skill
  ini ketika user bertanya tentang cara setup testing di Next.js App Router, mock untuk Server
  Actions atau Route Handlers, testing komponen yang menggunakan useSession/useRouter, pipeline
  CI/CD, Dockerfile untuk Next.js, atau setup monitoring dan error tracking production.
---

# Next.js Testing & DevOps

Skill ini mencakup strategi testing yang lengkap dan pipeline deployment untuk aplikasi Next.js production.

## Prinsip utama

Test yang baik memberi kepercayaan diri untuk refactor dan deploy. Fokus pada **integration tests** — test yang memverifikasi potongan fitur bekerja bersama — bukan sekedar unit test yang test fungsi satu per satu.

Piramida testing untuk Next.js:
- **Unit** (20%): fungsi utility, schema Zod, pure functions
- **Integration** (60%): komponen dengan data, form submission, API routes
- **E2E** (20%): alur kritis seperti login, checkout, onboarding

---

## Setup Vitest + React Testing Library

Vitest lebih cepat dari Jest untuk project Next.js modern.

```bash
npm install -D vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/user-event @testing-library/jest-dom
```

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./vitest.setup.ts'],
  },
  resolve: {
    alias: { '@': path.resolve(__dirname, '.') },
  },
})
```

```typescript
// vitest.setup.ts
import '@testing-library/jest-dom'
```

---

## Testing Komponen

### Komponen dengan mock router dan session

```typescript
// __tests__/components/UserMenu.test.tsx
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { UserMenu } from '@/components/UserMenu'

// Mock next/navigation
vi.mock('next/navigation', () => ({
  useRouter: () => ({ push: vi.fn(), refresh: vi.fn() }),
  usePathname: () => '/dashboard',
}))

// Mock next-auth
vi.mock('next-auth/react', () => ({
  useSession: () => ({
    data: { user: { name: 'Budi', email: 'budi@example.com', role: 'USER' } },
    status: 'authenticated',
  }),
}))

describe('UserMenu', () => {
  it('menampilkan nama user saat sudah login', () => {
    render(<UserMenu />)
    expect(screen.getByText('Budi')).toBeInTheDocument()
  })

  it('membuka dropdown saat diklik', async () => {
    const user = userEvent.setup()
    render(<UserMenu />)
    
    await user.click(screen.getByRole('button', { name: /budi/i }))
    expect(screen.getByRole('menuitem', { name: /logout/i })).toBeInTheDocument()
  })
})
```

### Testing form dengan Server Action mock

```typescript
// __tests__/components/CreatePostForm.test.tsx
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { CreatePostForm } from '@/components/CreatePostForm'
import * as actions from '@/app/actions/post'

vi.mock('@/app/actions/post', () => ({
  createPost: vi.fn(),
}))

describe('CreatePostForm', () => {
  it('mengirim data yang benar saat submit', async () => {
    const user = userEvent.setup()
    const mockCreatePost = vi.mocked(actions.createPost).mockResolvedValue({ id: '1' })
    
    render(<CreatePostForm />)
    
    await user.type(screen.getByLabelText(/judul/i), 'Post Baru')
    await user.type(screen.getByLabelText(/konten/i), 'Ini konten post yang panjang...')
    await user.click(screen.getByRole('button', { name: /simpan/i }))
    
    await waitFor(() => {
      expect(mockCreatePost).toHaveBeenCalledWith(
        expect.objectContaining({ title: 'Post Baru' })
      )
    })
  })

  it('menampilkan error validasi untuk judul kosong', async () => {
    const user = userEvent.setup()
    render(<CreatePostForm />)
    
    await user.click(screen.getByRole('button', { name: /simpan/i }))
    expect(await screen.findByText(/judul wajib diisi/i)).toBeInTheDocument()
  })
})
```

### Testing Route Handler

```typescript
// __tests__/api/posts.test.ts
import { POST } from '@/app/api/posts/route'
import { NextRequest } from 'next/server'

vi.mock('@/auth', () => ({
  auth: vi.fn().mockResolvedValue({ user: { id: 'user-1', role: 'USER' } }),
}))

vi.mock('@/lib/db', () => ({
  db: {
    posts: {
      create: vi.fn().mockResolvedValue({ id: 'post-1', title: 'Test Post' }),
    },
  },
}))

describe('POST /api/posts', () => {
  it('membuat post baru dengan data valid', async () => {
    const request = new NextRequest('http://localhost/api/posts', {
      method: 'POST',
      body: JSON.stringify({ title: 'Test Post', content: 'Konten yang cukup panjang...' }),
      headers: { 'Content-Type': 'application/json' },
    })
    
    const response = await POST(request)
    expect(response.status).toBe(201)
    
    const data = await response.json()
    expect(data).toMatchObject({ id: 'post-1' })
  })

  it('mengembalikan 422 untuk data tidak valid', async () => {
    const request = new NextRequest('http://localhost/api/posts', {
      method: 'POST',
      body: JSON.stringify({ title: '' }), // title kosong
      headers: { 'Content-Type': 'application/json' },
    })
    
    const response = await POST(request)
    expect(response.status).toBe(422)
  })
})
```

---

## Playwright — E2E Testing

Untuk alur kritis yang harus selalu bekerja di browser nyata.

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test'

test.describe('Alur Login', () => {
  test('user bisa login dengan credentials', async ({ page }) => {
    await page.goto('/login')
    
    await page.getByLabel('Email').fill('budi@example.com')
    await page.getByLabel('Password').fill('password123')
    await page.getByRole('button', { name: 'Masuk' }).click()
    
    // Verifikasi redirect ke dashboard
    await expect(page).toHaveURL('/dashboard')
    await expect(page.getByText('Selamat datang, Budi')).toBeVisible()
  })

  test('menampilkan error untuk credentials salah', async ({ page }) => {
    await page.goto('/login')
    
    await page.getByLabel('Email').fill('budi@example.com')
    await page.getByLabel('Password').fill('salah')
    await page.getByRole('button', { name: 'Masuk' }).click()
    
    await expect(page.getByText(/email atau password salah/i)).toBeVisible()
    await expect(page).toHaveURL('/login') // tidak redirect
  })
})
```

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  use: {
    baseURL: 'http://localhost:3000',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  webServer: {
    command: 'npm run dev',
    port: 3000,
    reuseExistingServer: !process.env.CI,
  },
})
```

---

## GitHub Actions — CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Lint
        run: npm run lint
      
      - name: Type check
        run: npm run type-check
      
      - name: Unit & Integration tests
        run: npm run test
        env:
          DATABASE_URL: ${{ secrets.TEST_DATABASE_URL }}
      
      - name: Build
        run: npm run build
        env:
          NEXT_PUBLIC_APP_URL: ${{ secrets.APP_URL }}

  e2e:
    runs-on: ubuntu-latest
    needs: test  # hanya jalan jika test sukses
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npm run test:e2e
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
```

---

## Dockerfile untuk Self-hosted

```dockerfile
FROM node:20-alpine AS base
RUN apk add --no-cache libc6-compat

FROM base AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED 1
RUN npm run build

FROM base AS runner
WORKDIR /app
ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT 3000

CMD ["node", "server.js"]
```

```typescript
// next.config.ts — wajib untuk standalone output
const nextConfig = {
  output: 'standalone', // menghasilkan folder minimal untuk production
}
```

---

## Monitoring dengan Sentry

```bash
npx @sentry/wizard@latest -i nextjs
```

```typescript
// sentry.client.config.ts
import * as Sentry from '@sentry/nextjs'

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  integrations: [Sentry.replayIntegration()],
})
```

```typescript
// Capture error manual dengan context
try {
  await riskyOperation()
} catch (error) {
  Sentry.captureException(error, {
    tags: { section: 'checkout' },
    extra: { orderId, userId },
  })
  throw error
}
```

---

## Scripts package.json

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "analyze": "ANALYZE=true next build"
  }
}
```

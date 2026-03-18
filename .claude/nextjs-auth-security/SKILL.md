---
name: nextjs-auth-security
description: >
  Panduan authentication dan security di Next.js: Auth.js (NextAuth v5), JWT, OAuth, session
  management, route protection via middleware, rate limiting, input validation, dan CORS.
  Gunakan skill ini ketika user bertanya tentang cara setup login dengan Google/GitHub/credentials,
  proteksi route berdasarkan role/session, implementasi JWT rotation, refresh token, cara kerja
  middleware Next.js untuk auth guard, validasi input API dengan Zod, rate limiting Route Handler,
  CORS configuration, atau keamanan environment variables. Aktif juga untuk Clerk, Lucia Auth,
  Better Auth, atau custom JWT implementation.
---

# Next.js Authentication & Security

Skill ini mencakup implementasi auth yang benar dan pola keamanan untuk aplikasi Next.js production.

## Prinsip utama

Auth yang lemah tidak terasa sampai terjadi breach. Dua aturan tidak bisa dikompromikan:
1. **Secrets tidak pernah menyentuh client** — tidak di bundle JS, tidak di response body
2. **Validasi di server, bukan hanya di client** — client-side validation adalah UX, bukan security

---

## Auth.js (NextAuth v5) — Setup Dasar

Auth.js adalah pilihan utama untuk Next.js karena integrasi native dengan App Router.

### Konfigurasi

```typescript
// auth.ts (di root project)
import NextAuth from 'next-auth'
import Google from 'next-auth/providers/google'
import Credentials from 'next-auth/providers/credentials'
import { DrizzleAdapter } from '@auth/drizzle-adapter'
import { db } from '@/lib/db'
import { z } from 'zod'
import bcrypt from 'bcryptjs'

export const { handlers, auth, signIn, signOut } = NextAuth({
  adapter: DrizzleAdapter(db), // simpan session ke DB
  
  providers: [
    Google({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
    
    Credentials({
      async authorize(credentials) {
        const parsed = z.object({
          email: z.string().email(),
          password: z.string().min(8),
        }).safeParse(credentials)
        
        if (!parsed.success) return null
        
        const user = await db.query.users.findFirst({
          where: eq(users.email, parsed.data.email)
        })
        
        if (!user?.password) return null
        
        const valid = await bcrypt.compare(parsed.data.password, user.password)
        if (!valid) return null
        
        return { id: user.id, email: user.email, role: user.role }
      }
    }),
  ],
  
  callbacks: {
    // Tambahkan custom fields ke JWT dan session
    jwt({ token, user }) {
      if (user) token.role = user.role
      return token
    },
    session({ session, token }) {
      session.user.role = token.role as string
      return session
    },
  },
  
  pages: {
    signIn: '/login',
    error: '/login',
  },
})
```

```typescript
// app/api/auth/[...nextauth]/route.ts
import { handlers } from '@/auth'
export const { GET, POST } = handlers
```

### Mengakses session

```typescript
// Di Server Component
import { auth } from '@/auth'

async function ProfilePage() {
  const session = await auth()
  if (!session) redirect('/login')
  
  return <div>Halo, {session.user.name}</div>
}
```

```typescript
// Di Client Component
'use client'
import { useSession } from 'next-auth/react'

function UserMenu() {
  const { data: session, status } = useSession()
  if (status === 'loading') return <Spinner />
  if (!session) return <LoginButton />
  return <span>{session.user.name}</span>
}
```

---

## Middleware untuk Route Protection

Middleware jalan di Edge Runtime — sangat cepat karena dieksekusi sebelum request mencapai handler.

```typescript
// middleware.ts (di root project)
import { auth } from '@/auth'
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export default auth((req) => {
  const { nextUrl, auth: session } = req
  const isLoggedIn = !!session
  
  // Route yang butuh auth
  const protectedRoutes = ['/dashboard', '/settings', '/profile']
  const isProtected = protectedRoutes.some(r => nextUrl.pathname.startsWith(r))
  
  // Route admin
  const adminRoutes = ['/admin']
  const isAdmin = adminRoutes.some(r => nextUrl.pathname.startsWith(r))
  
  if (isProtected && !isLoggedIn) {
    const loginUrl = new URL('/login', req.url)
    loginUrl.searchParams.set('callbackUrl', nextUrl.pathname)
    return NextResponse.redirect(loginUrl)
  }
  
  if (isAdmin && session?.user.role !== 'ADMIN') {
    return NextResponse.redirect(new URL('/403', req.url))
  }
  
  return NextResponse.next()
})

// Matcher: jalankan middleware hanya untuk route ini
export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
}
```

---

## Rate Limiting

Tanpa rate limiting, Route Handler bisa dieksploitasi untuk spam atau brute force.

```typescript
// lib/rate-limit.ts
import { LRUCache } from 'lru-cache'

type Options = { uniqueTokenPerInterval?: number; interval?: number }

export function rateLimit(options?: Options) {
  const tokenCache = new LRUCache<string, number[]>({
    max: options?.uniqueTokenPerInterval ?? 500,
    ttl: options?.interval ?? 60_000, // 1 menit
  })

  return {
    check: (limit: number, token: string) => {
      const tokenCount = tokenCache.get(token) ?? [0]
      
      if (tokenCount[0] === 0) {
        tokenCache.set(token, tokenCount)
      }
      
      tokenCount[0] += 1
      const currentUsage = tokenCount[0]
      const isRateLimited = currentUsage >= limit
      
      return { isRateLimited, currentUsage, limit }
    },
  }
}

const limiter = rateLimit({ interval: 60_000, uniqueTokenPerInterval: 500 })

// Penggunaan di Route Handler
export async function POST(request: NextRequest) {
  const ip = request.ip ?? request.headers.get('x-forwarded-for') ?? 'anonymous'
  const { isRateLimited } = limiter.check(10, ip) // max 10 req/menit per IP
  
  if (isRateLimited) {
    return NextResponse.json(
      { error: 'Terlalu banyak permintaan, coba lagi nanti' },
      { status: 429, headers: { 'Retry-After': '60' } }
    )
  }
  
  // lanjut proses request...
}
```

---

## Input Validation dengan Zod

Semua input dari user — body, params, searchParams — harus divalidasi di server.

```typescript
// app/api/posts/route.ts
import { z } from 'zod'

const CreatePostSchema = z.object({
  title: z.string().min(3).max(100).trim(),
  content: z.string().min(10).max(10000),
  tags: z.array(z.string()).max(5).optional(),
  published: z.boolean().default(false),
})

export async function POST(request: NextRequest) {
  const session = await auth()
  if (!session) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  
  const body = await request.json().catch(() => null)
  if (!body) return NextResponse.json({ error: 'Body tidak valid' }, { status: 400 })
  
  const result = CreatePostSchema.safeParse(body)
  if (!result.success) {
    return NextResponse.json(
      { error: 'Validasi gagal', details: result.error.flatten() },
      { status: 422 }
    )
  }
  
  // result.data sudah aman dan typed
  const post = await db.posts.create({ ...result.data, authorId: session.user.id })
  return NextResponse.json(post, { status: 201 })
}
```

---

## CORS Configuration

```typescript
// middleware.ts — tambahkan CORS headers
export function middleware(request: NextRequest) {
  const allowedOrigins = [
    'https://app.example.com',
    process.env.NODE_ENV === 'development' ? 'http://localhost:3000' : '',
  ].filter(Boolean)
  
  const origin = request.headers.get('origin') ?? ''
  const isAllowed = allowedOrigins.includes(origin)
  
  if (request.method === 'OPTIONS') {
    return new NextResponse(null, {
      status: 204,
      headers: {
        'Access-Control-Allow-Origin': isAllowed ? origin : '',
        'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type, Authorization',
        'Access-Control-Max-Age': '86400',
      },
    })
  }
  
  const response = NextResponse.next()
  if (isAllowed) {
    response.headers.set('Access-Control-Allow-Origin', origin)
  }
  return response
}
```

---

## Keamanan Environment Variables

```bash
# .env.local (TIDAK pernah di-commit ke git)
DATABASE_URL=postgresql://...
AUTH_SECRET=random-32-char-string  # generate: openssl rand -base64 32
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...

# Variabel yang AMAN dikirim ke client (prefix NEXT_PUBLIC_)
NEXT_PUBLIC_APP_URL=https://app.example.com
NEXT_PUBLIC_POSTHOG_KEY=phk_...
```

Aturan sederhana: jika nama variabel tidak dimulai `NEXT_PUBLIC_`, ia **tidak akan pernah** masuk ke bundle client — Next.js jamin ini.

---

## Server Actions — auth check wajib

Server Actions bisa dipanggil langsung dari client, jadi selalu cek session di awal.

```typescript
// app/actions/post.ts
'use server'
import { auth } from '@/auth'
import { revalidatePath } from 'next/cache'

export async function deletePost(postId: string) {
  // Selalu validasi session di Server Action
  const session = await auth()
  if (!session) throw new Error('Unauthorized')
  
  // Validasi ownership
  const post = await db.posts.findUnique({ where: { id: postId } })
  if (post?.authorId !== session.user.id && session.user.role !== 'ADMIN') {
    throw new Error('Forbidden')
  }
  
  await db.posts.delete({ where: { id: postId } })
  revalidatePath('/posts')
}
```

---

## Checklist security sebelum production

- [ ] `AUTH_SECRET` adalah string random 32+ karakter
- [ ] Semua Route Handler punya auth check
- [ ] Semua input divalidasi dengan Zod di server
- [ ] Rate limiting aktif di endpoint sensitif (login, register, reset password)
- [ ] `.env.local` ada di `.gitignore`
- [ ] Tidak ada secret di variabel `NEXT_PUBLIC_`
- [ ] CORS dibatasi hanya ke origin yang diizinkan
- [ ] Session expiry dikonfigurasi (default Auth.js: 30 hari)
- [ ] Middleware melindungi semua route yang butuh auth

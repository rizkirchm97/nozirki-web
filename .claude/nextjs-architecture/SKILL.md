---
name: nextjs-architecture
description: >
  Panduan arsitektur dan TypeScript untuk project Next.js skala besar: struktur folder
  feature-sliced, TypeScript strict mode, Zod untuk type safety end-to-end, pola BFF
  (Backend for Frontend), API aggregation layer, dan design patterns untuk maintainability.
  Gunakan skill ini ketika user bertanya tentang cara mengorganisir folder Next.js yang scalable,
  TypeScript generics dan utility types, pola untuk shared types antara server dan client,
  cara menstruktur monorepo Next.js, atau keputusan arsitektur seperti kapan pisahkan fitur
  menjadi modul tersendiri. Aktif juga untuk pertanyaan tentang tRPC, Prisma types, atau
  bagaimana mendesain layer abstraksi di atas database.
---

# Next.js Architecture & TypeScript Mastery

Skill ini mencakup keputusan arsitektur yang membuat project Next.js tetap maintainable saat berkembang.

## Prinsip utama

Struktur yang baik membuat lokasi kode mudah ditebak. Ketika tim baru bergabung, mereka bisa menemukan "di mana harus menambahkan fitur X?" dalam 30 detik. Jika tidak bisa, strukturnya perlu diperbaiki.

---

## Struktur Folder — Feature-based Organization

Pendekatan terbaik untuk project skala menengah ke besar adalah mengelompokkan kode berdasarkan **fitur**, bukan **jenis file**.

```
app/
├── (auth)/
│   ├── login/page.tsx
│   └── register/page.tsx
├── (dashboard)/
│   ├── layout.tsx
│   └── dashboard/page.tsx
├── api/
│   ├── auth/[...nextauth]/route.ts
│   ├── products/route.ts
│   └── orders/route.ts
└── layout.tsx

src/
├── features/                    ← Inti organisasi
│   ├── products/
│   │   ├── api/                 ← Fetching untuk fitur ini
│   │   │   └── get-products.ts
│   │   ├── components/          ← UI khusus products
│   │   │   ├── ProductCard.tsx
│   │   │   └── ProductList.tsx
│   │   ├── hooks/               ← Custom hooks untuk products
│   │   │   └── use-products.ts
│   │   ├── schemas/             ← Zod schemas & types
│   │   │   └── product.schema.ts
│   │   └── index.ts             ← Public API fitur ini
│   │
│   ├── orders/
│   │   └── ...
│   └── users/
│       └── ...
│
├── components/                  ← Komponen shared/reusable
│   ├── ui/                      ← Primitif UI (Button, Input, Modal)
│   └── layout/                  ← Shell, Sidebar, Navbar
│
├── lib/                         ← Utility & konfigurasi
│   ├── db.ts                    ← Database client
│   ├── api-client.ts            ← HTTP client terpusat
│   └── utils.ts
│
└── types/                       ← Global shared types
    └── index.ts
```

### Aturan import antar fitur

Fitur tidak boleh import langsung ke dalam fitur lain. Harus lewat `index.ts` (public API).

```typescript
// features/products/index.ts — yang boleh diakses dari luar
export { ProductCard } from './components/ProductCard'
export { ProductList } from './components/ProductList'
export { useProducts } from './hooks/use-products'
export type { Product, CreateProductInput } from './schemas/product.schema'
// Tidak export: detail implementasi internal

// Di fitur orders — import dari public API, bukan deep path
import { Product } from '@/features/products'  // ✓ benar
// import { Product } from '@/features/products/schemas/product.schema'  // ✗ hindari
```

---

## TypeScript — Pola untuk Next.js

### Strict mode — wajib di project baru

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,           // aktifkan semua strict checks
    "noUncheckedIndexedAccess": true,  // arr[0] bertipe T | undefined
    "exactOptionalPropertyTypes": true
  }
}
```

### Type-safe API response dengan generics

```typescript
// lib/api-client.ts
type ApiResponse<T> =
  | { success: true; data: T }
  | { success: false; error: string; status: number }

async function apiCall<T>(
  url: string,
  schema: z.ZodType<T>,
  options?: RequestInit
): Promise<ApiResponse<T>> {
  try {
    const res = await fetch(url, options)
    
    if (!res.ok) {
      return { success: false, error: res.statusText, status: res.status }
    }
    
    const json = await res.json()
    const parsed = schema.safeParse(json)
    
    if (!parsed.success) {
      return { success: false, error: 'Response shape tidak valid', status: 200 }
    }
    
    return { success: true, data: parsed.data }
  } catch {
    return { success: false, error: 'Network error', status: 0 }
  }
}

// Penggunaan
const result = await apiCall('/api/products', ProductListSchema)
if (result.success) {
  // TypeScript tahu result.data adalah Product[]
  console.log(result.data[0].name)
}
```

### Utility types yang sering dipakai

```typescript
// Buat type dari Prisma model tapi tanpa field sensitif
import { User } from '@prisma/client'

type PublicUser = Omit<User, 'password' | 'resetToken' | 'createdAt'>

// Partial untuk update (semua field opsional)
type UpdateProductInput = Partial<Pick<Product, 'name' | 'price' | 'stock'>>

// Discriminated union untuk handling berbagai state
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string }

// Type guard
function isSuccess<T>(state: FetchState<T>): state is Extract<FetchState<T>, { status: 'success' }> {
  return state.status === 'success'
}
```

### Type-safe environment variables

```typescript
// lib/env.ts — validasi di startup, bukan saat runtime
import { z } from 'zod'

const envSchema = z.object({
  DATABASE_URL: z.string().url(),
  AUTH_SECRET: z.string().min(32),
  NEXT_PUBLIC_APP_URL: z.string().url(),
  NODE_ENV: z.enum(['development', 'test', 'production']),
})

// Throws saat app start jika ada env yang kurang/salah
export const env = envSchema.parse(process.env)

// Penggunaan — autocomplete dan type-safe
import { env } from '@/lib/env'
const db = createClient(env.DATABASE_URL)
```

---

## Zod — Schema sebagai Single Source of Truth

Definisikan schema sekali, gunakan untuk validasi form (client) DAN API input (server).

```typescript
// features/products/schemas/product.schema.ts
import { z } from 'zod'

// Schema dasar
export const ProductSchema = z.object({
  id: z.string().cuid(),
  name: z.string().min(3, 'Nama minimal 3 karakter').max(100),
  price: z.number().positive('Harga harus lebih dari 0'),
  stock: z.number().int().min(0),
  categoryId: z.string().cuid(),
  createdAt: z.date(),
})

// Untuk form create — tanpa id dan createdAt
export const CreateProductSchema = ProductSchema.omit({ id: true, createdAt: true })

// Untuk form update — semua opsional kecuali id
export const UpdateProductSchema = ProductSchema.partial().required({ id: true })

// Infer types dari schema
export type Product = z.infer<typeof ProductSchema>
export type CreateProductInput = z.infer<typeof CreateProductSchema>
export type UpdateProductInput = z.infer<typeof UpdateProductSchema>
```

```typescript
// Pakai schema yang sama di form (client)
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { CreateProductSchema, CreateProductInput } from '@/features/products'

function CreateProductForm() {
  const form = useForm<CreateProductInput>({
    resolver: zodResolver(CreateProductSchema), // validasi otomatis
  })
  // ...
}
```

```typescript
// Dan di Route Handler (server)
import { CreateProductSchema } from '@/features/products'

export async function POST(req: NextRequest) {
  const body = await req.json()
  const result = CreateProductSchema.safeParse(body)
  if (!result.success) {
    return NextResponse.json({ errors: result.error.flatten() }, { status: 422 })
  }
  // result.data typed sebagai CreateProductInput
}
```

---

## BFF Pattern — Aggregation Layer

Daripada client memanggil 5 API berbeda, buat satu Route Handler yang menggabungkan semuanya.

```typescript
// app/api/dashboard/route.ts — BFF untuk halaman dashboard
import { auth } from '@/auth'
import { NextResponse } from 'next/server'

export async function GET() {
  const session = await auth()
  if (!session) return NextResponse.json({}, { status: 401 })

  // Fetch semua data yang dibutuhkan dashboard secara paralel
  const [user, recentOrders, stats, notifications] = await Promise.all([
    getUserById(session.user.id),
    getRecentOrders(session.user.id, 5),
    getDashboardStats(session.user.id),
    getUnreadNotifications(session.user.id),
  ])

  // Transform dan filter — kirim hanya yang dibutuhkan client
  return NextResponse.json({
    user: { name: user.name, avatar: user.avatar },
    recentOrders: recentOrders.map(o => ({
      id: o.id, total: o.total, status: o.status, createdAt: o.createdAt
    })),
    stats,
    notificationCount: notifications.length,
  })
}
```

---

## Barrel Exports yang Benar

Barrel exports (`index.ts`) mempercepat import, tapi jika sembarangan bisa merusak tree shaking.

```typescript
// features/products/index.ts
// Hanya export apa yang perlu diakses dari luar
export { ProductCard } from './components/ProductCard'
export { ProductList } from './components/ProductList'
export type { Product, CreateProductInput } from './schemas/product.schema'

// JANGAN re-export semua dengan wildcard
// export * from './components/ProductCard'  // ini rusak tree shaking
```

---

## Tanda arsitektur yang perlu diperbaiki

Ini sinyal bahwa struktur perlu di-refactor:

- **God component**: satu komponen > 200 baris — pecah menjadi komponen lebih kecil
- **Prop drilling lebih dari 2 level**: pakai Context atau pindahkan state ke atas
- **Import circular**: A import B, B import A — biasanya tanda fitur yang overlap, perlu dipisah
- **`any` type > 5 kali**: tiap `any` adalah technical debt, cari type yang tepat
- **Logika bisnis di komponen UI**: pindahkan ke custom hook atau service function
- **Duplikasi schema/type**: biasanya tanda fitur yang harusnya satu tapi dipecah

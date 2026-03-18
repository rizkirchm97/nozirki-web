---
name: nextjs-api-integration
description: >
  Panduan integrasi External API di Next.js: REST, GraphQL, WebSocket, dan real-time patterns.
  Gunakan skill ini ketika user bertanya tentang cara fetch data dari API eksternal, error handling
  yang robust, retry logic, type-safe API clients dengan Zod, pola BFF (Backend for Frontend),
  Route Handlers sebagai proxy, CORS handling, rate limiting, atau perbandingan REST vs GraphQL
  vs tRPC. Aktif juga untuk pertanyaan tentang Axios, ky, ofetch, urql, Apollo Client, atau
  implementasi WebSocket dan Server-Sent Events di Next.js.
---

# Next.js API Integration

Skill ini mencakup pola-pola terbaik untuk mengintegrasikan berbagai jenis API eksternal ke dalam aplikasi Next.js.

## Prinsip utama

API integration yang baik bukan hanya soal "bisa fetch data". Senior developer memikirkan: validasi tipe dari ujung ke ujung, graceful degradation saat API down, tidak expose secrets ke client, dan performa melalui caching yang tepat.

---

## REST API Integration

### Pola dasar dengan type safety (Zod)

Selalu validasi response dari API eksternal — jangan percaya bahwa shape-nya sesuai ekspektasi.

```typescript
// lib/api/products.ts
import { z } from 'zod'

const ProductSchema = z.object({
  id: z.string(),
  name: z.string(),
  price: z.number(),
  stock: z.number().optional(),
})

const ProductListSchema = z.array(ProductSchema)
export type Product = z.infer<typeof ProductSchema>

export async function getProducts(): Promise<Product[]> {
  const res = await fetch('https://api.store.com/products', {
    next: { revalidate: 300 },
  })
  
  if (!res.ok) {
    throw new Error(`API error: ${res.status} ${res.statusText}`)
  }
  
  const data = await res.json()
  return ProductListSchema.parse(data) // throws jika shape salah
}
```

### API client terpusat

Buat satu titik masuk untuk semua panggilan API. Ini memudahkan penambahan auth header, logging, dan retry logic di satu tempat.

```typescript
// lib/api/client.ts
interface FetchOptions extends RequestInit {
  params?: Record<string, string>
}

async function apiClient<T>(
  endpoint: string,
  options: FetchOptions = {}
): Promise<T> {
  const { params, ...fetchOptions } = options
  
  const url = new URL(`${process.env.API_BASE_URL}${endpoint}`)
  if (params) {
    Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v))
  }

  const res = await fetch(url.toString(), {
    ...fetchOptions,
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${process.env.API_SECRET_KEY}`,
      ...fetchOptions.headers,
    },
  })

  if (!res.ok) {
    const error = await res.json().catch(() => ({ message: res.statusText }))
    throw new ApiError(res.status, error.message)
  }

  return res.json() as Promise<T>
}

export class ApiError extends Error {
  constructor(public status: number, message: string) {
    super(message)
    this.name = 'ApiError'
  }
}
```

### Retry logic dengan exponential backoff

```typescript
// lib/api/retry.ts
async function fetchWithRetry<T>(
  fn: () => Promise<T>,
  maxRetries = 3,
  baseDelay = 500
): Promise<T> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn()
    } catch (error) {
      const isLastAttempt = attempt === maxRetries
      const isRetryable = error instanceof ApiError && error.status >= 500
      
      if (isLastAttempt || !isRetryable) throw error
      
      const delay = baseDelay * Math.pow(2, attempt) // 500, 1000, 2000ms
      await new Promise(resolve => setTimeout(resolve, delay))
    }
  }
  throw new Error('Unreachable')
}

// Penggunaan
const products = await fetchWithRetry(() => getProducts())
```

---

## Route Handler sebagai BFF (Backend for Frontend)

Jangan pernah panggil API eksternal langsung dari client component jika ada secrets yang terlibat. Gunakan Route Handler sebagai proxy.

```typescript
// app/api/products/route.ts
import { NextRequest, NextResponse } from 'next/server'

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url)
  const category = searchParams.get('category')

  try {
    // Secret key AMAN di sini — tidak dikirim ke browser
    const res = await fetch(`${process.env.THIRD_PARTY_API}/products?category=${category}`, {
      headers: { 'X-API-Key': process.env.SECRET_API_KEY! },
    })
    
    const data = await res.json()
    
    // Bisa transform/filter data sebelum dikirim ke client
    const sanitized = data.map(({ internalId, costPrice, ...safe }: any) => safe)
    
    return NextResponse.json(sanitized)
  } catch (error) {
    return NextResponse.json({ error: 'Gagal mengambil produk' }, { status: 500 })
  }
}
```

---

## GraphQL Integration

### Dengan urql (ringan, kompatibel RSC)

```typescript
// lib/graphql/client.ts
import { createClient, cacheExchange, fetchExchange } from '@urql/core'

export const graphqlClient = createClient({
  url: 'https://api.example.com/graphql',
  exchanges: [cacheExchange, fetchExchange],
  fetchOptions: {
    headers: { Authorization: `Bearer ${process.env.API_TOKEN}` },
  },
})

// Penggunaan di Server Component
const PRODUCTS_QUERY = `
  query GetProducts($limit: Int!) {
    products(limit: $limit) {
      id
      name
      price
    }
  }
`

async function ProductList() {
  const { data, error } = await graphqlClient.query(PRODUCTS_QUERY, { limit: 10 }).toPromise()
  
  if (error) return <ErrorMessage error={error} />
  return <ul>{data.products.map((p: any) => <li key={p.id}>{p.name}</li>)}</ul>
}
```

---

## WebSocket & Real-time

### Server-Sent Events (lebih sederhana untuk satu arah)

```typescript
// app/api/events/route.ts
export async function GET() {
  const stream = new ReadableStream({
    start(controller) {
      const encoder = new TextEncoder()
      
      const send = (data: object) => {
        controller.enqueue(encoder.encode(`data: ${JSON.stringify(data)}\n\n`))
      }

      // Kirim update setiap 5 detik
      const interval = setInterval(async () => {
        const stats = await getRealtimeStats()
        send(stats)
      }, 5000)

      // Cleanup
      return () => clearInterval(interval)
    },
  })

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
    },
  })
}
```

```typescript
// components/LiveStats.tsx
'use client'
import { useEffect, useState } from 'react'

function LiveStats() {
  const [stats, setStats] = useState(null)
  
  useEffect(() => {
    const source = new EventSource('/api/events')
    source.onmessage = (e) => setStats(JSON.parse(e.data))
    return () => source.close()
  }, [])
  
  return <div>{stats ? JSON.stringify(stats) : 'Menghubungkan...'}</div>
}
```

---

## Error Handling Patterns

### Error boundary untuk async Server Components

```typescript
// app/products/error.tsx
'use client'
export default function ProductsError({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div>
      <h2>Gagal memuat produk</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Coba lagi</button>
    </div>
  )
}
```

### Parallel fetching — hindari waterfall

```typescript
// BURUK: waterfall — tunggu satu selesai baru mulai yang lain
async function BadPage() {
  const user = await getUser()         // 200ms
  const orders = await getOrders()     // 300ms — mulai setelah user selesai
  const products = await getProducts() // 400ms — mulai setelah orders selesai
  // Total: 900ms
}

// BAGUS: paralel — semua mulai bersamaan
async function GoodPage() {
  const [user, orders, products] = await Promise.all([
    getUser(),      // \
    getOrders(),    //  > semua mulai bersamaan
    getProducts(),  // /
  ])
  // Total: max(200, 300, 400) = 400ms
}
```

---

## REST vs GraphQL vs tRPC — kapan pakai apa

| | REST | GraphQL | tRPC |
|---|---|---|---|
| API milik kita sendiri | ✓ | ✓ | ✓ (terbaik) |
| API pihak ketiga | ✓ (satu-satunya pilihan) | ✓ jika mereka support | ✗ |
| Over/under-fetching | Sering terjadi | Tidak ada | Tidak ada |
| Type safety e2e | Manual/codegen | codegen | Otomatis |
| Learning curve | Rendah | Sedang | Sedang |
| Cocok untuk | Public API, microservice | Data kompleks, banyak consumer | Full-stack monorepo |

---

## Contoh output yang diharapkan

**Input:** "Bagaimana cara memanggil API eksternal yang butuh API key dari komponen React?"

**Output yang benar:**
Jangan pernah taruh API key di client component. Alurnya:
1. Buat Route Handler di `app/api/proxy/route.ts` yang menyimpan secret
2. Client component fetch ke `/api/proxy` (bukan ke API eksternal langsung)
3. Route Handler yang forward request dengan secret key

Sertakan kode lengkap untuk keduanya dan jelaskan alasan keamanannya.

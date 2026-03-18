---
name: nextjs-state-caching
description: >
  Panduan state management dan caching di Next.js modern: TanStack Query untuk server state,
  Zustand/Jotai untuk UI state, dan 4 layer caching App Router. Gunakan skill ini ketika user
  bertanya tentang kapan pakai TanStack Query vs Zustand vs Redux, cara kerja caching di App
  Router (Request Memoization, Data Cache, Full Route Cache, Router Cache), pola optimistic
  updates, cache invalidation, revalidatePath, revalidateTag, atau setup TanStack Query dengan
  React Server Components. Aktif juga untuk pertanyaan tentang SWR, Jotai, Valtio, atau
  masalah stale data di Next.js.
---

# Next.js State Management & Caching

Skill ini membantu memilih state solution yang tepat dan memahami layer caching App Router secara mendalam.

## Prinsip utama

Ada dua jenis state yang sering dikacaukan:
- **Server state**: data yang hidup di server/database, hanya di-*copy* ke client — `TanStack Query` atau `SWR`
- **UI state**: state yang hanya relevan di browser (modal open, form draft, tema) — `Zustand` atau `Jotai`

Salah memilih tool menyebabkan kompleksitas berlebihan. Redux untuk loading spinner adalah over-engineering; `useState` untuk daftar produk dari API adalah under-engineering.

---

## TanStack Query — Server State

Gunakan untuk semua data yang berasal dari server/API.

### Setup dengan App Router

```typescript
// app/providers.tsx
'use client'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { useState } from 'react'

export function Providers({ children }: { children: React.ReactNode }) {
  // Buat per-instance agar tidak shared antar request di server
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // data dianggap segar selama 1 menit
        retry: 2,
      },
    },
  }))

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}
```

### Pola: prefetch di server, hydrate di client

Ini adalah pola terbaik untuk App Router — data sudah ada saat halaman dirender, tanpa loading spinner.

```typescript
// app/products/page.tsx (Server Component)
import { dehydrate, HydrationBoundary, QueryClient } from '@tanstack/react-query'
import { getProducts } from '@/lib/api/products'

export default async function ProductsPage() {
  const queryClient = new QueryClient()
  
  // Prefetch di server
  await queryClient.prefetchQuery({
    queryKey: ['products'],
    queryFn: getProducts,
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <ProductList /> {/* Client component sudah punya data */}
    </HydrationBoundary>
  )
}
```

```typescript
// components/ProductList.tsx
'use client'
import { useQuery } from '@tanstack/react-query'
import { getProducts } from '@/lib/api/products'

export function ProductList() {
  const { data: products, isLoading, error } = useQuery({
    queryKey: ['products'],
    queryFn: getProducts,
    // Data sudah ada dari prefetch — tidak akan loading spinner
  })

  if (isLoading) return <Skeleton />
  if (error) return <ErrorMessage />
  return <ul>{products?.map(p => <ProductCard key={p.id} product={p} />)}</ul>
}
```

### Optimistic Updates

```typescript
'use client'
import { useMutation, useQueryClient } from '@tanstack/react-query'

function LikeButton({ postId }: { postId: string }) {
  const queryClient = useQueryClient()

  const { mutate: toggleLike } = useMutation({
    mutationFn: (id: string) => fetch(`/api/posts/${id}/like`, { method: 'POST' }),
    
    // Update UI sebelum server konfirmasi
    onMutate: async (id) => {
      await queryClient.cancelQueries({ queryKey: ['posts', id] })
      const previous = queryClient.getQueryData(['posts', id])
      
      queryClient.setQueryData(['posts', id], (old: any) => ({
        ...old,
        likes: old.likes + 1,
        likedByMe: true,
      }))
      
      return { previous } // untuk rollback jika error
    },
    
    onError: (err, id, context) => {
      // Rollback jika server error
      queryClient.setQueryData(['posts', id], context?.previous)
    },
    
    onSettled: (data, error, id) => {
      // Refetch untuk sinkronisasi akhir
      queryClient.invalidateQueries({ queryKey: ['posts', id] })
    },
  })

  return <button onClick={() => toggleLike(postId)}>Like</button>
}
```

---

## Zustand — UI State

Gunakan untuk state yang murni di sisi client: sidebar open/closed, shopping cart sementara, wizard step, tema.

```typescript
// store/ui.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface CartItem {
  id: string
  name: string
  qty: number
  price: number
}

interface CartStore {
  items: CartItem[]
  addItem: (item: Omit<CartItem, 'qty'>) => void
  removeItem: (id: string) => void
  clearCart: () => void
  total: () => number
}

export const useCartStore = create<CartStore>()(
  persist( // persist ke localStorage otomatis
    (set, get) => ({
      items: [],
      
      addItem: (item) => set((state) => {
        const existing = state.items.find(i => i.id === item.id)
        if (existing) {
          return { items: state.items.map(i => 
            i.id === item.id ? { ...i, qty: i.qty + 1 } : i
          )}
        }
        return { items: [...state.items, { ...item, qty: 1 }] }
      }),
      
      removeItem: (id) => set(state => ({
        items: state.items.filter(i => i.id !== id)
      })),
      
      clearCart: () => set({ items: [] }),
      
      total: () => get().items.reduce((sum, i) => sum + i.price * i.qty, 0),
    }),
    { name: 'cart-storage' }
  )
)
```

---

## 4 Layer Caching App Router

Memahami ini adalah tanda senior developer. Banyak developer bingung kenapa data "tidak update" karena tidak tahu layer mana yang perlu di-invalidate.

```
Request masuk
     ↓
1. Request Memoization  ← Dalam satu render tree, fetch URL sama hanya jalan sekali
     ↓
2. Data Cache           ← Hasil fetch disimpan di filesystem Next.js
     ↓
3. Full Route Cache     ← HTML + RSC payload halaman statis disimpan di server
     ↓
4. Router Cache         ← Client-side cache di browser (bertahan selama navigasi)
```

### Layer 1: Request Memoization

Otomatis — `fetch` ke URL yang sama dalam satu request tree hanya dipanggil sekali. Tidak perlu action apa-apa.

```typescript
// Kedua komponen ini fetch ke URL yang sama
// Next.js hanya akan buat 1 HTTP request
async function UserAvatar() {
  const user = await fetch('/api/user') // request ke-1
}
async function UserName() {
  const user = await fetch('/api/user') // TIDAK buat request baru, pakai hasil yang sama
}
```

### Layer 2: Data Cache

Dikontrol via `fetch` options. Persist across requests dan deployments.

```typescript
fetch(url, { cache: 'force-cache' })          // simpan selamanya (default SSG)
fetch(url, { cache: 'no-store' })             // jangan cache (SSR)
fetch(url, { next: { revalidate: 3600 } })    // cache, regenerate tiap 1 jam
fetch(url, { next: { tags: ['products'] } })  // cache dengan tag untuk on-demand revalidation
```

### Layer 3: Full Route Cache

Halaman statis di-cache di server. Di-invalidate otomatis saat `next build` atau saat `revalidatePath` dipanggil.

```typescript
// app/actions.ts
'use server'
import { revalidatePath, revalidateTag } from 'next/cache'

export async function updateProduct(id: string, data: any) {
  await db.products.update(id, data)
  
  revalidatePath('/products')        // invalidate halaman /products
  revalidatePath(`/products/${id}`)  // invalidate halaman spesifik
  revalidateTag('products')          // invalidate semua fetch dengan tag 'products'
}
```

### Layer 4: Router Cache

Browser cache untuk RSC payload. Bertahan selama sesi navigasi.

```typescript
'use client'
import { useRouter } from 'next/navigation'

// Force refresh dari server
const router = useRouter()
router.refresh() // invalidate Router Cache untuk halaman saat ini
```

---

## Tabel keputusan state

| Jenis state | Tool | Alasan |
|---|---|---|
| Data dari API | TanStack Query / SWR | Caching, deduplication, background refetch |
| Form state sementara | `useState` | Lokal, tidak perlu global |
| Modal/drawer open | Zustand (jika diakses banyak komponen) atau `useState` | UI state, tidak perlu persist |
| Shopping cart | Zustand + `persist` | Global, perlu survive reload |
| Auth user | TanStack Query atau Context | Server state yang perlu di-share |
| URL filter/search | `useSearchParams` | Shareable, navigable |
| Server data global | Jangan disimpan di client state! | Fetch ulang atau gunakan cache |

---

## Anti-pattern yang sering terjadi

```typescript
// BURUK: menyimpan server data di Zustand
const useProductStore = create((set) => ({
  products: [], // ini harusnya di TanStack Query
  fetchProducts: async () => {
    const data = await fetch('/api/products')
    set({ products: await data.json() })
    // Tidak ada caching, tidak ada revalidation, tidak ada error handling otomatis
  }
}))

// BAGUS: pisahkan concerns
// Server state → TanStack Query
const { data: products } = useQuery({ queryKey: ['products'], queryFn: fetchProducts })

// UI state → Zustand
const selectedIds = useProductStore(state => state.selectedIds)
```

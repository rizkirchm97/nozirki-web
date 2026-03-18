---
name: nextjs-core
description: >
  Panduan mendalam tentang Next.js App Router, Pages Router, rendering strategies (SSR, SSG, ISR, CSR),
  React Server Components (RSC), dan arsitektur modern Next.js 13+. Gunakan skill ini ketika user
  bertanya tentang App Router vs Pages Router, cara kerja rendering di Next.js, kapan menggunakan
  server vs client components, setup layout bersarang, metadata API, route handlers, atau migrasi
  dari Pages Router ke App Router. Juga aktif untuk pertanyaan tentang next/navigation, next/headers,
  cookies(), dan pola async component.
---

# Next.js Core & Rendering Strategy

Skill ini mencakup arsitektur inti Next.js modern dengan fokus pada App Router dan keputusan rendering yang tepat.

## Prinsip utama

Setiap keputusan rendering harus dimulai dari satu pertanyaan: **seberapa sering data ini berubah, dan siapa yang butuh melihatnya?** Jawaban itu menentukan strategi — bukan preferensi atau kebiasaan.

---

## App Router vs Pages Router

### Perbedaan fundamental

| Aspek | Pages Router | App Router |
|---|---|---|
| Lokasi file | `pages/` | `app/` |
| Unit route | satu file = satu route | folder dengan `page.tsx` |
| Data fetching | `getServerSideProps`, `getStaticProps` | `async` component langsung |
| Layout | `_app.tsx` global | `layout.tsx` bersarang per-segment |
| Loading state | Manual | `loading.tsx` built-in |
| Error boundary | Manual | `error.tsx` built-in |
| Komponen default | Client component | Server component |

### Kapan tetap pakai Pages Router

- Project lama yang belum siap migrasi besar
- Tim yang belum familiar dengan RSC mental model
- Library pihak ketiga yang belum kompatibel dengan App Router

### Kapan pindah ke App Router

- Project baru — App Router adalah default resmi sejak Next.js 13.4
- Butuh nested layout yang efisien
- Ingin manfaat penuh RSC (zero JS bundle untuk server components)

---

## Rendering Strategies

### SSR — Server-Side Rendering

Data di-fetch setiap request. Gunakan ketika data harus selalu fresh dan personal (dashboard user, feed real-time).

```typescript
// App Router: cukup jadikan komponen async + no-store
async function UserDashboard() {
  const data = await fetch('https://api.example.com/user', {
    cache: 'no-store' // SSR behavior
  })
  const user = await data.json()
  return <div>{user.name}</div>
}
```

```typescript
// Pages Router equivalent
export async function getServerSideProps(context) {
  const res = await fetch(`https://api.example.com/user`)
  const user = await res.json()
  return { props: { user } }
}
```

### SSG — Static Site Generation

Di-generate saat build. Gunakan untuk konten yang jarang berubah: landing page, blog, dokumentasi.

```typescript
// App Router: default behavior (force-cache)
async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`, {
    cache: 'force-cache' // SSG behavior
  }).then(r => r.json())
  return <article>{post.content}</article>
}

// Generate semua halaman statis
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json())
  return posts.map((post: any) => ({ slug: post.slug }))
}
```

### ISR — Incremental Static Regeneration

Hybrid: statis tapi bisa di-revalidate. Gunakan untuk konten yang berubah tapi tidak perlu real-time.

```typescript
// App Router: next.revalidate
async function ProductPage({ params }: { params: { id: string } }) {
  const product = await fetch(`https://api.example.com/products/${params.id}`, {
    next: { revalidate: 3600 } // regenerate setiap 1 jam
  }).then(r => r.json())
  return <div>{product.name}</div>
}
```

### CSR — Client-Side Rendering

Di-render di browser. Gunakan hanya untuk UI yang sangat interaktif atau data yang tidak perlu SEO.

```typescript
'use client'
import { useState, useEffect } from 'react'

function LiveCounter() {
  const [count, setCount] = useState(0)
  // Ambil data di client, bukan server
  useEffect(() => {
    const interval = setInterval(() => setCount(c => c + 1), 1000)
    return () => clearInterval(interval)
  }, [])
  return <span>{count}</span>
}
```

---

## React Server Components (RSC)

### Mental model yang benar

Bayangkan dua dunia terpisah:
- **Server world**: akses DB langsung, secrets aman, nol JS dikirim ke client, tidak bisa useState/useEffect
- **Client world**: interaktif, bisa hooks, tapi JS-nya dikirim ke browser

### Aturan `'use client'`

`'use client'` bukan berarti "ini hanya jalan di client" — ini berarti "ini adalah batas antara server dan client tree". Semua komponen yang diimpor oleh client component otomatis jadi client component juga.

```typescript
// Server Component (default) — aman untuk secrets
async function ProductList() {
  // Ini TIDAK dikirim ke client
  const apiKey = process.env.STRIPE_SECRET_KEY
  const products = await db.query('SELECT * FROM products')
  
  return (
    <ul>
      {products.map(p => (
        // Bisa pass server data ke client component
        <AddToCartButton key={p.id} productId={p.id} name={p.name} />
      ))}
    </ul>
  )
}
```

```typescript
'use client'
// Client Component — interaktif
function AddToCartButton({ productId, name }: { productId: string; name: string }) {
  const [added, setAdded] = useState(false)
  return (
    <button onClick={() => setAdded(true)}>
      {added ? 'Ditambahkan!' : `Tambah ${name}`}
    </button>
  )
}
```

### Pola: server component sebagai shell

```typescript
// BENAR: server component bungkus client component
// app/dashboard/page.tsx (Server Component)
async function DashboardPage() {
  const user = await getUser() // langsung dari DB
  return (
    <DashboardShell user={user}>
      <InteractiveChart /> {/* Client Component */}
    </DashboardShell>
  )
}
```

---

## Layout System (App Router)

Layout di-render satu kali dan tidak di-remount saat navigasi antar halaman dalam segment yang sama. Ini membuat shared state di layout tetap hidup.

```
app/
├── layout.tsx          ← Root layout (wajib, wraps semua)
├── (marketing)/
│   ├── layout.tsx      ← Layout khusus halaman marketing
│   └── about/page.tsx
└── (dashboard)/
    ├── layout.tsx      ← Layout dengan sidebar
    └── settings/page.tsx
```

```typescript
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="id">
      <body>
        <Navbar />
        {children}
        <Footer />
      </body>
    </html>
  )
}
```

Route groups `(nama)` mengorganisir file tanpa mempengaruhi URL.

---

## Metadata API

```typescript
// Static metadata
export const metadata: Metadata = {
  title: 'Judul Halaman',
  description: 'Deskripsi untuk SEO',
}

// Dynamic metadata
export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const product = await fetchProduct(params.id)
  return {
    title: product.name,
    openGraph: { images: [product.image] },
  }
}
```

---

## Checklist keputusan rendering

Gunakan urutan ini untuk menentukan strategi:

1. Apakah data **personal** dan **sensitif**? → `cache: 'no-store'` (SSR)
2. Apakah data **berubah periodik** (menit/jam)? → `next: { revalidate: N }` (ISR)
3. Apakah data **statis saat build**? → `cache: 'force-cache'` atau default (SSG)
4. Apakah butuh **interaktivitas real-time** di client? → `'use client'` + hooks (CSR)

---

## Contoh output yang diharapkan

**Input:** "Bagaimana cara fetch data user yang harus fresh setiap request di App Router?"

**Output yang benar:**
```typescript
// app/profile/page.tsx
async function ProfilePage() {
  const user = await fetch('https://api.example.com/me', {
    cache: 'no-store',
    headers: { Authorization: `Bearer ${cookies().get('token')?.value}` }
  }).then(r => r.json())

  return <ProfileCard user={user} />
}
```
Jelaskan kenapa `cache: 'no-store'` dipilih dan implikasinya terhadap performa.

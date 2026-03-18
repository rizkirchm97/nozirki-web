---
name: nextjs-performance
description: >
  Panduan optimasi performa Next.js: Core Web Vitals (LCP, CLS, INP), image optimization,
  bundle analysis, code splitting, font strategy, dan lazy loading. Gunakan skill ini ketika
  user bertanya tentang cara memperbaiki skor Lighthouse, mengoptimasi LCP atau CLS, penggunaan
  next/image, next/font, dynamic import, bundle size yang besar, third-party scripts lambat,
  atau cara membaca webpack bundle analyzer. Aktif juga untuk pertanyaan tentang Partial
  Prerendering (PPR), Streaming dengan Suspense, atau optimasi Time to First Byte (TTFB).
---

# Next.js Performance Optimization

Skill ini membantu mengidentifikasi dan memperbaiki bottleneck performa di aplikasi Next.js.

## Prinsip utama

Optimasi tanpa pengukuran adalah spekulasi. Selalu ukur dulu dengan Lighthouse atau `@next/bundle-analyzer` sebelum mulai mengubah kode. Perbaiki masalah terbesar dulu — 20% perubahan biasanya menghasilkan 80% peningkatan.

---

## Core Web Vitals — Memahami Metrik

### LCP (Largest Contentful Paint) — target < 2.5 detik
Waktu sampai elemen terbesar (hero image, heading utama) muncul. Ini yang paling sering bermasalah.

**Penyebab umum:**
- Gambar hero tidak di-preload
- Font memblokir render
- Server response lambat

### CLS (Cumulative Layout Shift) — target < 0.1
Seberapa banyak konten bergeser saat halaman dimuat. Biasanya karena gambar tanpa dimensi atau ads yang muncul terlambat.

### INP (Interaction to Next Paint) — target < 200ms
Responsivitas saat user klik/ketik. Biasanya karena JavaScript yang terlalu berat di main thread.

---

## next/image — Image Optimization

`next/image` otomatis mengubah format ke WebP/AVIF, meresize sesuai viewport, dan lazy load. Jangan pernah pakai `<img>` HTML biasa untuk gambar konten.

### Penggunaan yang benar

```typescript
import Image from 'next/image'

// Gambar lokal — width/height otomatis dari file
import heroImage from '@/public/hero.jpg'

function HeroSection() {
  return (
    <Image
      src={heroImage}
      alt="Deskripsi gambar yang bermakna"
      priority          // ← penting untuk LCP! Preload gambar above-the-fold
      sizes="100vw"     // memberi tahu browser ukuran sebelum layout selesai
    />
  )
}

// Gambar remote — wajib cantumkan width & height
function ProductCard({ imageUrl }: { imageUrl: string }) {
  return (
    <Image
      src={imageUrl}
      alt="Produk"
      width={400}
      height={300}
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 400px"
      // sizes yang tepat = browser pilih ukuran file terkecil yang cukup
    />
  )
}
```

### Konfigurasi domain remote di next.config

```typescript
// next.config.ts
const nextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'cdn.example.com',
        pathname: '/images/**',
      },
    ],
    formats: ['image/avif', 'image/webp'], // prioritaskan AVIF (lebih kecil)
  },
}
```

### Mencegah CLS dengan gambar

```typescript
// BURUK: tidak ada dimensi = layout shift
<img src="/hero.jpg" alt="Hero" style={{ width: '100%' }} />

// BAGUS: fill + container dengan aspect ratio
<div style={{ position: 'relative', aspectRatio: '16/9' }}>
  <Image src="/hero.jpg" alt="Hero" fill sizes="100vw" />
</div>
```

---

## next/font — Zero Layout Shift untuk Fonts

Font dari Google Fonts atau font lokal dihosting sendiri oleh Next.js, menghilangkan request ke domain lain dan mencegah CLS dari font swap.

```typescript
// app/layout.tsx
import { Inter, Poppins } from 'next/font/google'
import localFont from 'next/font/local'

const inter = Inter({
  subsets: ['latin'],
  variable: '--font-inter',  // CSS variable untuk dipakai di Tailwind
  display: 'swap',
})

const poppins = Poppins({
  subsets: ['latin'],
  weight: ['400', '600', '700'],
  variable: '--font-poppins',
})

// Font lokal custom
const brandFont = localFont({
  src: [
    { path: '../public/fonts/Brand-Regular.woff2', weight: '400' },
    { path: '../public/fonts/Brand-Bold.woff2', weight: '700' },
  ],
  variable: '--font-brand',
})

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html className={`${inter.variable} ${poppins.variable} ${brandFont.variable}`}>
      <body>{children}</body>
    </html>
  )
}
```

---

## Code Splitting & Dynamic Import

JavaScript yang tidak perlu di halaman tertentu jangan dimuat. `dynamic()` membuat komponen berat hanya dimuat saat dibutuhkan.

```typescript
import dynamic from 'next/dynamic'

// Komponen berat yang tidak dibutuhkan saat SSR
const HeavyChart = dynamic(() => import('@/components/HeavyChart'), {
  loading: () => <ChartSkeleton />,
  ssr: false, // jika komponen tidak kompatibel dengan SSR (e.g., pakai window)
})

// Modal — tidak perlu dimuat sampai user klik
const VideoPlayer = dynamic(() => import('@/components/VideoPlayer'), {
  loading: () => <div>Memuat video...</div>,
})

// Penggunaan normal — Next.js otomatis split bundle
function ProductPage() {
  const [showChart, setShowChart] = useState(false)
  
  return (
    <div>
      <button onClick={() => setShowChart(true)}>Lihat Chart</button>
      {showChart && <HeavyChart />} {/* baru dimuat saat dibutuhkan */}
    </div>
  )
}
```

### Analisis bundle dengan @next/bundle-analyzer

```bash
npm install @next/bundle-analyzer
```

```typescript
// next.config.ts
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
})

module.exports = withBundleAnalyzer(nextConfig)
```

```bash
ANALYZE=true npm run build
# Buka browser — lihat apa yang paling besar dan apakah bisa di-lazy load
```

---

## Streaming dengan Suspense

Daripada menunggu semua data selesai di-fetch baru render halaman, streaming memungkinkan bagian halaman muncul secara bertahap.

```typescript
// app/dashboard/page.tsx
import { Suspense } from 'react'

export default function DashboardPage() {
  return (
    <div>
      {/* Ini muncul langsung */}
      <DashboardHeader />
      
      {/* Ini stream masuk saat data siap */}
      <Suspense fallback={<StatsSkeleton />}>
        <RevenueStats />  {/* async component yang fetch data */}
      </Suspense>
      
      <Suspense fallback={<TableSkeleton />}>
        <OrdersTable />   {/* fetch data sendiri secara paralel */}
      </Suspense>
    </div>
  )
}

// Komponen ini fetch datanya sendiri
async function RevenueStats() {
  const stats = await getRevenueStats() // bisa lambat
  return <StatsCard data={stats} />
}
```

Hasilnya: user sudah lihat header dan skeleton sebelum data selesai dimuat, bukan halaman kosong.

---

## Third-party Scripts

Script eksternal (analytics, chat widget, ads) sering menjadi penyebab lambatnya LCP/INP.

```typescript
import Script from 'next/script'

// afterInteractive: load setelah halaman interaktif (analytics)
<Script
  src="https://www.googletagmanager.com/gtag/js?id=GA_ID"
  strategy="afterInteractive"
/>

// lazyOnload: load saat browser idle (chat widget, low priority)
<Script
  src="https://cdn.chatsupport.com/widget.js"
  strategy="lazyOnload"
/>

// beforeInteractive: load sebelum hydration (rare, consent banner)
<Script
  src="/scripts/consent.js"
  strategy="beforeInteractive"
/>
```

---

## Checklist performa sebelum production

**LCP:**
- [ ] Gambar hero menggunakan `<Image priority />`
- [ ] `sizes` prop diisi dengan nilai yang tepat di semua `<Image>`
- [ ] Font menggunakan `next/font` bukan CDN eksternal

**CLS:**
- [ ] Semua gambar punya `width` dan `height` atau container dengan `aspect-ratio`
- [ ] Font menggunakan `display: 'swap'` atau `display: 'optional'`
- [ ] Tidak ada konten yang muncul terlambat tanpa reserved space

**INP:**
- [ ] Komponen berat (chart, editor, map) menggunakan `dynamic()`
- [ ] Event handler yang berat dipindah ke Web Worker atau `requestIdleCallback`

**Bundle:**
- [ ] `@next/bundle-analyzer` dijalankan dan tidak ada dependensi raksasa yang tidak perlu
- [ ] Lodash diimpor per-fungsi: `import debounce from 'lodash/debounce'` bukan `import _ from 'lodash'`

**Umum:**
- [ ] Streaming dengan `<Suspense>` untuk halaman dengan banyak data fetch independen
- [ ] `script strategy="afterInteractive"` untuk semua analytics/tracking

---
name: animation-motion
description: >
  Panduan animasi dan motion di Next.js: Framer Motion untuk React, CSS animations dan transitions,
  View Transitions API, dan prinsip motion design yang tidak mengganggu performance. Gunakan skill
  ini ketika user bertanya tentang cara animasi komponen masuk/keluar, page transitions di Next.js,
  gesture animations (drag, swipe), scroll-triggered animations, layout animations dengan Framer
  Motion, atau kapan pakai CSS animation vs JavaScript animation. Aktif juga untuk pertanyaan
  tentang prefers-reduced-motion, animasi yang menyebabkan CLS/INP buruk, AnimatePresence,
  useMotionValue, atau GSAP di Next.js.
---

# Animation & Motion

Skill ini mencakup implementasi animasi yang terasa natural, performant, dan tidak mengganggu user.

## Prinsip utama

Animasi yang baik **tidak terasa seperti animasi** — ia terasa seperti physics. Benda yang bergerak mengikuti hukum alam: ada momentum, ada easing, tidak ada yang muncul atau menghilang secara instan. Dan yang terpenting: selalu sediakan opsi untuk user yang sensitif terhadap gerakan dengan `prefers-reduced-motion`.

---

## `prefers-reduced-motion` — Wajib di Semua Animasi

Ini bukan opsional. User dengan epilepsi atau vestibular disorder bisa terdampak animasi yang berlebihan.

```css
/* CSS — matikan semua animasi jika user request */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

```tsx
// React hook untuk cek preference
function usePrefersReducedMotion(): boolean {
  const [prefersReduced, setPrefersReduced] = useState(false)
  
  useEffect(() => {
    const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)')
    setPrefersReduced(mediaQuery.matches)
    
    const handler = (e: MediaQueryListEvent) => setPrefersReduced(e.matches)
    mediaQuery.addEventListener('change', handler)
    return () => mediaQuery.removeEventListener('change', handler)
  }, [])
  
  return prefersReduced
}
```

---

## CSS Transitions — Untuk Hover & State Changes

Gunakan CSS transitions untuk perubahan state sederhana. Ini paling performant karena di-handle browser natively.

```css
/* Selalu animasikan hanya: opacity, transform, filter */
/* Jangan animasikan: width, height, left, top — trigger layout reflow */

.button {
  background-color: hsl(var(--primary));
  transition: background-color 150ms ease, transform 100ms ease;
}

.button:hover {
  background-color: hsl(var(--primary) / 0.9);
  transform: translateY(-1px);
}

.button:active {
  transform: translateY(0);
  transition-duration: 50ms;
}
```

### CSS Keyframes untuk animasi berulang

```css
@keyframes spin {
  to { transform: rotate(360deg); }
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

@keyframes bounce-in {
  0% { transform: scale(0.8); opacity: 0; }
  60% { transform: scale(1.05); opacity: 1; }
  100% { transform: scale(1); }
}

/* Hanya aktif jika user tidak request reduced motion */
@media (prefers-reduced-motion: no-preference) {
  .spinner { animation: spin 1s linear infinite; }
  .skeleton { animation: pulse 2s ease-in-out infinite; }
  .card-enter { animation: bounce-in 0.3s ease-out; }
}
```

---

## Framer Motion — Animasi React yang Powerful

### Setup

```bash
npm install framer-motion
```

### Animasi masuk sederhana

```tsx
import { motion } from 'framer-motion'

// Komponen motion langsung pakai
function FadeInCard({ children }: { children: React.ReactNode }) {
  return (
    <motion.div
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
      transition={{ duration: 0.3, ease: 'easeOut' }}
    >
      {children}
    </motion.div>
  )
}
```

### `AnimatePresence` — animasi keluar (unmount)

React tidak bisa menganimasikan komponen yang di-unmount. `AnimatePresence` menangani ini.

```tsx
import { AnimatePresence, motion } from 'framer-motion'

function NotificationList({ notifications }: { notifications: Notification[] }) {
  return (
    <AnimatePresence initial={false}>
      {notifications.map((n) => (
        <motion.div
          key={n.id}
          initial={{ opacity: 0, height: 0, y: -10 }}
          animate={{ opacity: 1, height: 'auto', y: 0 }}
          exit={{ opacity: 0, height: 0 }}           // animasi saat di-unmount
          transition={{ duration: 0.2 }}
          className="notification-item"
        >
          {n.message}
        </motion.div>
      ))}
    </AnimatePresence>
  )
}
```

### Layout animations — animasi perubahan posisi otomatis

```tsx
// Framer Motion otomatis animasikan perubahan posisi/ukuran
function SortableList({ items }: { items: Item[] }) {
  const [sorted, setSorted] = useState(false)
  
  return (
    <>
      <button onClick={() => setSorted(!sorted)}>Sort</button>
      {(sorted ? [...items].sort() : items).map((item) => (
        <motion.div
          key={item.id}
          layout                           // aktifkan layout animation
          transition={{ type: 'spring', stiffness: 300, damping: 30 }}
        >
          {item.name}
        </motion.div>
      ))}
    </>
  )
}
```

### Stagger children — animasi berurutan

```tsx
const containerVariants = {
  hidden: { opacity: 0 },
  visible: {
    opacity: 1,
    transition: {
      staggerChildren: 0.08,     // setiap child delay 80ms dari sebelumnya
      delayChildren: 0.1,
    },
  },
}

const itemVariants = {
  hidden: { opacity: 0, x: -20 },
  visible: { opacity: 1, x: 0 },
}

function AnimatedList({ items }: { items: string[] }) {
  return (
    <motion.ul variants={containerVariants} initial="hidden" animate="visible">
      {items.map((item, i) => (
        <motion.li key={i} variants={itemVariants}>
          {item}
        </motion.li>
      ))}
    </motion.ul>
  )
}
```

### Scroll-triggered animations

```tsx
import { motion, useInView } from 'framer-motion'
import { useRef } from 'react'

function RevealOnScroll({ children }: { children: React.ReactNode }) {
  const ref = useRef(null)
  const isInView = useInView(ref, {
    once: true,       // hanya animasi sekali
    margin: '-10%',   // trigger 10% sebelum masuk viewport
  })

  return (
    <motion.div
      ref={ref}
      initial={{ opacity: 0, y: 40 }}
      animate={isInView ? { opacity: 1, y: 0 } : {}}
      transition={{ duration: 0.5, ease: 'easeOut' }}
    >
      {children}
    </motion.div>
  )
}
```

### Gesture animations

```tsx
function DraggableCard() {
  return (
    <motion.div
      drag                                    // aktifkan drag semua arah
      dragConstraints={{ left: -100, right: 100, top: -50, bottom: 50 }}
      dragElastic={0.1}                       // resistance di luar constraints
      whileHover={{ scale: 1.02 }}
      whileTap={{ scale: 0.98 }}
      whileDrag={{ cursor: 'grabbing', scale: 1.05 }}
      className="cursor-grab"
    >
      Drag me
    </motion.div>
  )
}
```

---

## View Transitions API — Page Transitions Native

```tsx
// next.config.ts — aktifkan experimental
const nextConfig = {
  experimental: {
    viewTransition: true,
  },
}
```

```tsx
// Gunakan unstable_ViewTransition dari React
import { unstable_ViewTransition as ViewTransition } from 'react'

// Wrap elemen yang ingin di-animasikan antar navigasi
function ProductCard({ product }: { product: Product }) {
  return (
    <ViewTransition name={`product-${product.id}`}>
      <img src={product.image} alt={product.name} />
    </ViewTransition>
  )
}

// Di halaman detail, pakai name yang sama
function ProductDetail({ product }: { product: Product }) {
  return (
    <ViewTransition name={`product-${product.id}`}>
      <img src={product.image} alt={product.name} className="w-full" />
    </ViewTransition>
  )
}
// Browser otomatis animasikan transisi antara dua elemen dengan name yang sama
```

---

## Easing Reference — Pilih yang Tepat

```
ease-in     → lambat di awal, cepat di akhir → untuk elemen yang keluar/hilang
ease-out    → cepat di awal, lambat di akhir → untuk elemen yang masuk/muncul  
ease-in-out → lambat di kedua ujung → untuk perubahan state in-place
linear      → kecepatan konstan → untuk spin, progress bar
spring      → overshoot natural → untuk feedback interaksi (hover, klik)
```

```tsx
// Spring di Framer Motion — paling natural untuk interaksi
<motion.div
  transition={{
    type: 'spring',
    stiffness: 400,   // seberapa kaku per (lebih kaku = lebih cepat balik)
    damping: 25,      // resistensi (lebih tinggi = kurang bounce)
  }}
/>
```

---

## Animasi yang Merusak Performance — Hindari

```css
/* BURUK — trigger layout recalculation */
.bad {
  transition: width 0.3s, height 0.3s, top 0.3s, left 0.3s;
}

/* BAGUS — hanya trigger composite layer */
.good {
  transition: transform 0.3s, opacity 0.3s;
}

/* Hint browser untuk composite layer */
.will-animate {
  will-change: transform;  /* gunakan hemat, jangan semua elemen */
}
```

---

## Checklist animasi sebelum production

- [ ] Semua animasi punya fallback `prefers-reduced-motion: reduce`
- [ ] Tidak ada animasi yang animate `width`, `height`, `top`, `left` — pakai `transform` + `opacity`
- [ ] `will-change` hanya dipakai pada elemen yang benar-benar akan dianimasikan
- [ ] `AnimatePresence` dipakai untuk komponen yang bisa unmount
- [ ] Durasi animasi masuk ≤ 300ms, keluar ≤ 200ms (user tidak mau menunggu)
- [ ] INP tidak turun setelah animasi ditambahkan (ukur di Chrome DevTools)

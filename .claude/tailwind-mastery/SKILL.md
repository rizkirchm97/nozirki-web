---
name: tailwind-mastery
description: >
  Panduan mendalam Tailwind CSS untuk Next.js: konfigurasi custom design tokens, arbitrary values,
  advanced variants (group, peer, has, data-*), plugin ekosistem, dan pola komponen yang scalable
  dengan clsx/tailwind-merge. Gunakan skill ini ketika user bertanya tentang cara kustomisasi
  Tailwind config, penggunaan CSS variables dengan Tailwind, conditional class dengan clsx,
  tailwind-merge untuk class deduplication, plugin @tailwindcss/typography atau forms,
  pola folder untuk Tailwind di project besar, atau migrasi ke Tailwind v4. Aktif juga untuk
  pertanyaan tentang mengapa utility class overflow atau tidak ter-purge saat production build.
---

# Tailwind CSS Mastery

Skill ini mencakup Tailwind CSS dari konfigurasi dasar hingga pola advanced yang dipakai di project production skala besar.

## Prinsip utama

Tailwind bukan hanya mengganti CSS dengan class pendek. Senior developer menggunakan Tailwind sebagai **design token system** — semua nilai warna, spacing, font didefinsikan satu kali di `tailwind.config.ts`, dan seluruh codebase mengacu ke token tersebut, bukan nilai hardcoded.

---

## Konfigurasi — Design Tokens

```typescript
// tailwind.config.ts
import type { Config } from 'tailwindcss'

const config: Config = {
  content: ['./app/**/*.{ts,tsx}', './components/**/*.{ts,tsx}'],
  
  theme: {
    extend: {
      // Warna dari CSS variables — support dark mode otomatis
      colors: {
        background: 'hsl(var(--background) / <alpha-value>)',
        foreground: 'hsl(var(--foreground) / <alpha-value>)',
        primary: {
          DEFAULT: 'hsl(var(--primary) / <alpha-value>)',
          foreground: 'hsl(var(--primary-foreground) / <alpha-value>)',
        },
        muted: {
          DEFAULT: 'hsl(var(--muted) / <alpha-value>)',
          foreground: 'hsl(var(--muted-foreground) / <alpha-value>)',
        },
        border: 'hsl(var(--border) / <alpha-value>)',
        destructive: 'hsl(var(--destructive) / <alpha-value>)',
      },
      
      // Radius dari CSS variable untuk konsistensi
      borderRadius: {
        lg: 'var(--radius)',
        md: 'calc(var(--radius) - 2px)',
        sm: 'calc(var(--radius) - 4px)',
      },
      
      // Custom font families
      fontFamily: {
        sans: ['var(--font-sans)', 'system-ui', 'sans-serif'],
        mono: ['var(--font-mono)', 'monospace'],
      },
      
      // Spacing custom jika dibutuhkan
      spacing: {
        '18': '4.5rem',
        '112': '28rem',
        '128': '32rem',
      },
      
      // Animation custom
      keyframes: {
        'fade-in': {
          from: { opacity: '0', transform: 'translateY(4px)' },
          to: { opacity: '1', transform: 'translateY(0)' },
        },
        'slide-in': {
          from: { transform: 'translateX(-100%)' },
          to: { transform: 'translateX(0)' },
        },
      },
      animation: {
        'fade-in': 'fade-in 0.2s ease-out',
        'slide-in': 'slide-in 0.3s ease-out',
      },
    },
  },
  
  plugins: [
    require('@tailwindcss/typography'),
    require('@tailwindcss/forms'),
  ],
}

export default config
```

```css
/* app/globals.css — definisikan CSS variables */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --primary: 221.2 83.2% 53.3%;
    --primary-foreground: 210 40% 98%;
    --muted: 210 40% 96.1%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --border: 214.3 31.8% 91.4%;
    --destructive: 0 84.2% 60.2%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --primary: 217.2 91.2% 59.8%;
    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;
    --border: 217.2 32.6% 17.5%;
  }
}
```

---

## Variants Advanced

### `group` — styling child berdasarkan parent state

```tsx
<div className="group cursor-pointer rounded-lg border p-4 hover:border-primary">
  <h3 className="font-medium group-hover:text-primary transition-colors">
    Judul Card
  </h3>
  <p className="text-muted-foreground group-hover:text-foreground transition-colors">
    Deskripsi yang berubah warna saat parent di-hover
  </p>
  {/* Icon hanya muncul saat hover */}
  <ArrowRight className="h-4 w-4 opacity-0 group-hover:opacity-100 transition-opacity" />
</div>
```

### `peer` — styling sibling berdasarkan state element lain

```tsx
{/* Input error state yang mengubah label warnanya */}
<div>
  <input
    type="email"
    className="peer border rounded px-3 py-2 invalid:border-destructive focus:outline-none"
    placeholder="email@example.com"
  />
  <p className="text-sm text-muted-foreground peer-invalid:text-destructive">
    Masukkan email yang valid
  </p>
</div>
```

### `has` — parent bereaksi terhadap child state (CSS `:has()`)

```tsx
{/* Card yang highlight jika ada checkbox yang di-check di dalamnya */}
<label className="flex cursor-pointer items-center gap-3 rounded-lg border p-4 has-[:checked]:border-primary has-[:checked]:bg-primary/5">
  <input type="checkbox" className="h-4 w-4 accent-primary" />
  <span>Pilihan ini</span>
</label>
```

### `data-*` — styling berdasarkan data attributes

```tsx
{/* Komponen dengan state dari data attribute */}
<div
  data-state={isOpen ? 'open' : 'closed'}
  className="overflow-hidden transition-all data-[state=open]:animate-fade-in data-[state=closed]:hidden"
>
  Konten accordion
</div>
```

---

## Conditional Classes dengan `clsx` + `tailwind-merge`

`clsx` untuk logika kondisional, `tailwind-merge` (`cn`) untuk resolve konflik class Tailwind.

```typescript
// lib/utils.ts
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

```tsx
// Penggunaan
import { cn } from '@/lib/utils'

interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'default' | 'outline' | 'ghost' | 'destructive'
  size?: 'sm' | 'md' | 'lg'
}

function Button({ variant = 'default', size = 'md', className, ...props }: ButtonProps) {
  return (
    <button
      className={cn(
        // Base styles
        'inline-flex items-center justify-center rounded-md font-medium transition-colors',
        'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring',
        'disabled:pointer-events-none disabled:opacity-50',
        // Variant styles
        {
          'bg-primary text-primary-foreground hover:bg-primary/90': variant === 'default',
          'border border-input bg-background hover:bg-accent': variant === 'outline',
          'hover:bg-accent hover:text-accent-foreground': variant === 'ghost',
          'bg-destructive text-destructive-foreground hover:bg-destructive/90': variant === 'destructive',
        },
        // Size styles
        {
          'h-8 px-3 text-xs': size === 'sm',
          'h-10 px-4 py-2': size === 'md',
          'h-11 px-8 text-base': size === 'lg',
        },
        // Override dari luar — twMerge resolve konflik
        className
      )}
      {...props}
    />
  )
}
```

Kenapa butuh `tailwind-merge`: tanpa itu, `className="p-4"` yang dipass dari luar tidak akan override `p-2` dari base style. `twMerge` cerdas menghapus class yang tertimpa.

---

## CVA — Class Variance Authority (lebih clean dari object literal)

Untuk komponen dengan banyak variant, `cva` lebih readable dari `if/else` atau object literal.

```typescript
import { cva, type VariantProps } from 'class-variance-authority'
import { cn } from '@/lib/utils'

const badgeVariants = cva(
  // Base styles
  'inline-flex items-center rounded-full border px-2.5 py-0.5 text-xs font-semibold transition-colors',
  {
    variants: {
      variant: {
        default: 'border-transparent bg-primary text-primary-foreground',
        secondary: 'border-transparent bg-secondary text-secondary-foreground',
        destructive: 'border-transparent bg-destructive text-destructive-foreground',
        outline: 'text-foreground',
        success: 'border-transparent bg-green-100 text-green-800 dark:bg-green-900 dark:text-green-100',
      },
    },
    defaultVariants: {
      variant: 'default',
    },
  }
)

interface BadgeProps
  extends React.HTMLAttributes<HTMLDivElement>,
    VariantProps<typeof badgeVariants> {}

function Badge({ className, variant, ...props }: BadgeProps) {
  return <div className={cn(badgeVariants({ variant }), className)} {...props} />
}
```

---

## Arbitrary Values — Keluar dari grid sistem

Untuk nilai yang tidak ada di preset Tailwind.

```tsx
// Ukuran pixel spesifik
<div className="w-[327px] h-[156px]" />

// CSS variables arbitary
<div className="bg-[--brand-color] text-[--brand-text]" />

// Calc
<div className="w-[calc(100%-2rem)]" />

// Gradient custom
<div className="bg-[linear-gradient(135deg,#667eea_0%,#764ba2_100%)]" />

// CSS property yang tidak ada di Tailwind
<div className="[scroll-snap-type:x_mandatory] [scroll-behavior:smooth]" />

// Pseudo-element
<div className="before:content-[''] before:absolute before:inset-0 before:bg-black/50" />
```

---

## @tailwindcss/typography — Styling konten markdown

Untuk halaman blog atau dokumentasi yang render HTML dari markdown.

```tsx
// Gunakan kelas 'prose' pada wrapper konten
<article className="prose prose-lg dark:prose-invert max-w-none
  prose-headings:font-bold prose-headings:tracking-tight
  prose-a:text-primary prose-a:no-underline hover:prose-a:underline
  prose-code:rounded prose-code:bg-muted prose-code:px-1">
  <div dangerouslySetInnerHTML={{ __html: post.content }} />
</article>
```

---

## Pola: `@layer components` untuk utility class yang dipakai berulang

Jangan duplikasi string panjang class Tailwind di banyak tempat. Buat abstraksi di CSS layer.

```css
/* app/globals.css */
@layer components {
  .card {
    @apply rounded-lg border bg-card text-card-foreground shadow-sm;
  }
  
  .input-base {
    @apply flex h-10 w-full rounded-md border border-input bg-background px-3 py-2
           text-sm ring-offset-background placeholder:text-muted-foreground
           focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring
           disabled:cursor-not-allowed disabled:opacity-50;
  }
  
  .section-container {
    @apply mx-auto max-w-7xl px-4 sm:px-6 lg:px-8;
  }
}
```

---

## Checklist Tailwind production

- [ ] `content` di config mencakup semua file yang pakai class Tailwind (cek purge benar)
- [ ] Warna dan spacing pakai CSS variables, bukan hardcoded hex
- [ ] `cn()` helper tersedia dan dipakai di semua komponen
- [ ] Tidak ada `!important` atau style inline kecuali untuk nilai dynamic
- [ ] Dark mode variant dicek di semua komponen
- [ ] `prefers-reduced-motion` dicek untuk semua animasi Tailwind

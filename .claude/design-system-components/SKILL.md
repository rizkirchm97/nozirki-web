---
name: design-system-components
description: >
  Panduan membangun design system dan komponen library di Next.js: shadcn/ui sebagai fondasi,
  Radix UI primitives, design tokens, theming multi-brand, dan dokumentasi dengan Storybook.
  Gunakan skill ini ketika user bertanya tentang cara setup shadcn/ui, membuat komponen yang
  reusable dan accessible, pola compound components, render props, controlled vs uncontrolled
  components, cara kerja Radix UI primitives, membangun design token system, multi-theme
  support, atau dokumentasi komponen dengan Storybook. Aktif juga untuk pertanyaan tentang
  React Aria, HeadlessUI, atau kapan build custom vs pakai library.
---

# Design System & Component Library

Skill ini mencakup cara membangun sistem komponen yang konsisten, accessible, dan mudah di-maintain.

## Prinsip utama

Design system yang baik adalah **satu sumber kebenaran** untuk visual dan behavior. Tim developer tidak perlu berdiskusi tentang ukuran border radius atau warna button — itu sudah diputuskan dan diencapsulate di sistem. Komponen tidak hanya cantik, tapi juga accessible dan bisa di-compose dengan bebas.

---

## shadcn/ui — Fondasi yang Tepat

shadcn/ui bukan library yang di-install sebagai npm package — komponennya di-*copy* ke dalam project. Artinya kamu punya full ownership dan bisa modifikasi sebebasnya.

```bash
npx shadcn@latest init

# Tambah komponen satu per satu
npx shadcn@latest add button
npx shadcn@latest add input dialog card
```

Ini menghasilkan file di `components/ui/` yang bisa langsung dimodifikasi sesuai design system project.

### Kustomisasi tema shadcn

```css
/* app/globals.css — override token shadcn */
@layer base {
  :root {
    /* Brand color utama */
    --primary: 231 48% 48%;       /* indigo-600 */
    --primary-foreground: 0 0% 100%;
    
    /* Radius lebih rounded untuk feel modern */
    --radius: 0.75rem;
  }
}
```

---

## Pola Komponen yang Reusable

### Compound Component Pattern

Untuk komponen kompleks yang butuh bagian terpisah tapi masih terhubung (Card, Dialog, Accordion).

```tsx
// components/ui/card.tsx
interface CardContextValue {
  variant: 'default' | 'outlined' | 'elevated'
}

const CardContext = createContext<CardContextValue>({ variant: 'default' })

function Card({ variant = 'default', className, ...props }: CardProps) {
  return (
    <CardContext.Provider value={{ variant }}>
      <div
        className={cn(
          'rounded-lg bg-card text-card-foreground',
          {
            'border shadow-sm': variant === 'default',
            'border-2': variant === 'outlined',
            'shadow-lg': variant === 'elevated',
          },
          className
        )}
        {...props}
      />
    </CardContext.Provider>
  )
}

function CardHeader({ className, ...props }: React.HTMLAttributes<HTMLDivElement>) {
  return <div className={cn('flex flex-col space-y-1.5 p-6', className)} {...props} />
}

function CardTitle({ className, ...props }: React.HTMLAttributes<HTMLHeadingElement>) {
  const { variant } = useContext(CardContext)
  return (
    <h3
      className={cn(
        'text-2xl font-semibold leading-none tracking-tight',
        variant === 'elevated' && 'text-primary',
        className
      )}
      {...props}
    />
  )
}

function CardContent({ className, ...props }: React.HTMLAttributes<HTMLDivElement>) {
  return <div className={cn('p-6 pt-0', className)} {...props} />
}

// Export sebagai namespace
Card.Header = CardHeader
Card.Title = CardTitle
Card.Content = CardContent

// Penggunaan
<Card variant="elevated">
  <Card.Header>
    <Card.Title>Judul</Card.Title>
  </Card.Header>
  <Card.Content>Konten</Card.Content>
</Card>
```

### Polymorphic Component (as prop)

Komponen yang bisa render sebagai tag HTML berbeda atau komponen lain.

```tsx
// components/ui/text.tsx
type TextElement = 'p' | 'span' | 'div' | 'h1' | 'h2' | 'h3' | 'h4' | 'label'

interface TextProps<T extends TextElement = 'p'> {
  as?: T
  variant?: 'body' | 'caption' | 'label' | 'heading'
  className?: string
  children: React.ReactNode
}

function Text<T extends TextElement = 'p'>({
  as,
  variant = 'body',
  className,
  children,
  ...props
}: TextProps<T> & Omit<React.ComponentPropsWithoutRef<T>, keyof TextProps<T>>) {
  const Component = (as ?? 'p') as TextElement

  return (
    <Component
      className={cn(
        {
          'text-base leading-7': variant === 'body',
          'text-sm text-muted-foreground': variant === 'caption',
          'text-sm font-medium leading-none': variant === 'label',
          'text-2xl font-bold tracking-tight': variant === 'heading',
        },
        className
      )}
      {...props}
    >
      {children}
    </Component>
  )
}

// Penggunaan
<Text>Paragraf biasa</Text>
<Text as="h2" variant="heading">Judul Section</Text>
<Text as="span" variant="caption">Keterangan kecil</Text>
<Text as="label" variant="label" htmlFor="email">Email</Text>
```

---

## Radix UI Primitives — Accessible Headless Components

Radix menyediakan behavior dan accessibility, kamu menyediakan styling.

```tsx
// Contoh: Dialog accessible dengan Radix
import * as DialogPrimitive from '@radix-ui/react-dialog'
import { X } from 'lucide-react'

const Dialog = DialogPrimitive.Root
const DialogTrigger = DialogPrimitive.Trigger
const DialogPortal = DialogPrimitive.Portal

const DialogOverlay = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Overlay>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Overlay>
>(({ className, ...props }, ref) => (
  <DialogPrimitive.Overlay
    ref={ref}
    className={cn(
      'fixed inset-0 z-50 bg-black/80',
      // Animasi masuk/keluar
      'data-[state=open]:animate-in data-[state=closed]:animate-out',
      'data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0',
      className
    )}
    {...props}
  />
))

const DialogContent = React.forwardRef<
  React.ElementRef<typeof DialogPrimitive.Content>,
  React.ComponentPropsWithoutRef<typeof DialogPrimitive.Content>
>(({ className, children, ...props }, ref) => (
  <DialogPortal>
    <DialogOverlay />
    <DialogPrimitive.Content
      ref={ref}
      className={cn(
        'fixed left-[50%] top-[50%] z-50 translate-x-[-50%] translate-y-[-50%]',
        'w-full max-w-lg rounded-lg border bg-background p-6 shadow-lg',
        'data-[state=open]:animate-in data-[state=closed]:animate-out',
        'data-[state=closed]:fade-out-0 data-[state=open]:fade-in-0',
        'data-[state=closed]:zoom-out-95 data-[state=open]:zoom-in-95',
        className
      )}
      {...props}
    >
      {children}
      {/* Close button dengan aria-label untuk screen reader */}
      <DialogPrimitive.Close className="absolute right-4 top-4 rounded-sm opacity-70 hover:opacity-100">
        <X className="h-4 w-4" />
        <span className="sr-only">Tutup</span>
      </DialogPrimitive.Close>
    </DialogPrimitive.Content>
  </DialogPortal>
))
```

---

## Design Tokens — Sumber Kebenaran Visual

Design token adalah abstraksi di atas nilai primitif. `color.primary` lebih bermakna dari `#4F46E5`.

```typescript
// tokens/index.ts
export const tokens = {
  color: {
    // Primitif (jangan dipakai langsung di komponen)
    blue: {
      50: '#eff6ff',
      500: '#3b82f6',
      900: '#1e3a8a',
    },
    // Semantic (pakai ini di komponen)
    brand: {
      primary: 'hsl(var(--primary))',
      primaryForeground: 'hsl(var(--primary-foreground))',
    },
    feedback: {
      error: 'hsl(var(--destructive))',
      success: 'hsl(142 76% 36%)',
      warning: 'hsl(38 92% 50%)',
    },
  },
  
  spacing: {
    px: '1px',
    xs: '0.5rem',
    sm: '0.75rem',
    md: '1rem',
    lg: '1.5rem',
    xl: '2rem',
    '2xl': '3rem',
  },
  
  typography: {
    size: {
      xs: '0.75rem',
      sm: '0.875rem',
      base: '1rem',
      lg: '1.125rem',
      xl: '1.25rem',
    },
    weight: {
      normal: '400',
      medium: '500',
      semibold: '600',
      bold: '700',
    },
  },
} as const
```

---

## Multi-theme Support

```tsx
// providers/theme-provider.tsx
'use client'
import { createContext, useContext, useEffect, useState } from 'react'

type Theme = 'light' | 'dark' | 'system'

interface ThemeContextValue {
  theme: Theme
  setTheme: (theme: Theme) => void
}

const ThemeContext = createContext<ThemeContextValue>({
  theme: 'system',
  setTheme: () => null,
})

export function ThemeProvider({ children }: { children: React.ReactNode }) {
  const [theme, setTheme] = useState<Theme>('system')

  useEffect(() => {
    const root = document.documentElement
    root.classList.remove('light', 'dark')
    
    if (theme === 'system') {
      const systemTheme = window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light'
      root.classList.add(systemTheme)
    } else {
      root.classList.add(theme)
    }
    
    localStorage.setItem('theme', theme)
  }, [theme])

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  )
}

export const useTheme = () => useContext(ThemeContext)
```

---

## Storybook — Dokumentasi Komponen

```typescript
// stories/Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react'
import { Button } from '@/components/ui/button'

const meta: Meta<typeof Button> = {
  title: 'UI/Button',
  component: Button,
  parameters: {
    layout: 'centered',
  },
  tags: ['autodocs'],
  argTypes: {
    variant: {
      control: 'select',
      options: ['default', 'outline', 'ghost', 'destructive'],
    },
    size: {
      control: 'select',
      options: ['sm', 'md', 'lg'],
    },
  },
}

export default meta
type Story = StoryObj<typeof Button>

export const Default: Story = {
  args: { children: 'Button' }
}

export const AllVariants: Story = {
  render: () => (
    <div className="flex gap-2">
      <Button variant="default">Default</Button>
      <Button variant="outline">Outline</Button>
      <Button variant="ghost">Ghost</Button>
      <Button variant="destructive">Destructive</Button>
    </div>
  )
}
```

---

## Checklist Design System

- [ ] Semua nilai warna mengacu ke CSS variables (bukan hardcoded hex)
- [ ] `cn()` dipakai konsisten di semua komponen
- [ ] Setiap komponen interaktif punya variant `disabled` yang jelas
- [ ] Komponen form menggunakan `forwardRef` agar bisa dipakai dengan `react-hook-form`
- [ ] Dark mode dicek untuk semua komponen di Storybook
- [ ] Komponen yang punya behavior complex (dialog, dropdown, tooltip) pakai Radix primitives
- [ ] `displayName` diset untuk komponen yang pakai `forwardRef` (membantu debugging)

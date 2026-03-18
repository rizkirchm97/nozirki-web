---
name: accessibility-a11y
description: >
  Panduan accessibility (a11y) untuk komponen React dan Next.js: ARIA roles dan attributes,
  keyboard navigation, focus management, color contrast WCAG, dan testing dengan axe-core.
  Gunakan skill ini ketika user bertanya tentang cara membuat komponen modal yang accessible,
  keyboard navigation di dropdown atau menu, focus trap, skip navigation link, ARIA live regions
  untuk update dinamis, semantic HTML yang benar, color contrast ratio, atau cara testing
  accessibility dengan screen reader. Aktif juga untuk pertanyaan tentang WCAG 2.1/2.2,
  React Aria, @radix-ui accessibility, atau cara fix accessibility violation dari axe.
---

# Accessibility (a11y)

Skill ini mencakup implementasi accessibility yang benar di komponen React dan aplikasi Next.js.

## Prinsip utama

Accessibility bukan fitur tambahan — ia adalah **kualitas dasar** seperti tidak ada bug. Komponen yang tidak accessible adalah komponen yang broken bagi sebagian user. Senior developer menulis komponen accessible dari awal, bukan mem-patch belakangan.

Tiga area utama:
1. **Semantic HTML** — pakai elemen yang tepat untuk maknanya
2. **Keyboard navigation** — semua fungsi bisa diakses tanpa mouse
3. **Screen reader support** — informasi yang disampaikan secara visual harus tersedia secara programatik

---

## Semantic HTML — Fondasi Accessibility

Pilih elemen HTML yang mewakili makna konten, bukan hanya tampilannya.

```tsx
// BURUK — div tidak punya semantic meaning
<div onClick={handleSubmit} className="btn">Submit</div>

// BAGUS — button punya role, keyboard support, dan accessibility otomatis
<button type="submit" onClick={handleSubmit}>Submit</button>
```

```tsx
// Struktur halaman yang benar
<body>
  <a href="#main-content" className="sr-only focus:not-sr-only">
    Skip to main content
  </a>
  
  <header role="banner">
    <nav aria-label="Navigasi utama">
      <ul>
        <li><a href="/">Beranda</a></li>
        <li><a href="/about">Tentang</a></li>
      </ul>
    </nav>
  </header>
  
  <main id="main-content">
    <h1>Judul Halaman</h1>  {/* Satu h1 per halaman */}
    
    <article>
      <h2>Sub-topik</h2>
    </article>
  </main>
  
  <aside aria-label="Konten tambahan">
    {/* Sidebar */}
  </aside>
  
  <footer role="contentinfo">
    {/* Footer */}
  </footer>
</body>
```

---

## ARIA — Roles, Properties, States

ARIA dipakai **hanya ketika HTML semantic tidak cukup**. Aturan pertama ARIA: jangan pakai ARIA jika ada HTML element yang tepat.

### Role yang sering dipakai

```tsx
// Alert — konten penting yang perlu diumumkan ke screen reader
<div role="alert" aria-live="assertive">
  {errorMessage}
</div>

// Status — update non-kritis (loading, saving...)
<div role="status" aria-live="polite">
  {statusMessage}
</div>

// Dialog / modal
<div
  role="dialog"
  aria-modal="true"
  aria-labelledby="dialog-title"
  aria-describedby="dialog-desc"
>
  <h2 id="dialog-title">Konfirmasi Hapus</h2>
  <p id="dialog-desc">Apakah Anda yakin ingin menghapus item ini?</p>
</div>
```

### Label dan description

```tsx
// Label untuk input
<div>
  <label htmlFor="email">Email</label>
  <input
    id="email"
    type="email"
    aria-required="true"
    aria-invalid={hasError}
    aria-describedby={hasError ? 'email-error' : undefined}
  />
  {hasError && (
    <p id="email-error" role="alert">
      Format email tidak valid
    </p>
  )}
</div>

// Icon button — selalu butuh label
<button aria-label="Hapus item: Produk A">
  <TrashIcon aria-hidden="true" />  {/* sembunyikan icon dari screen reader */}
</button>

// Informasi tambahan yang tidak terlihat
<button>
  Tambah ke keranjang
  <span className="sr-only">: Sepatu Nike Air Max</span>
</button>
```

### Expanded/collapsed state

```tsx
function Accordion({ title, children }: AccordionProps) {
  const [isOpen, setIsOpen] = useState(false)
  const contentId = useId()

  return (
    <div>
      <button
        onClick={() => setIsOpen(!isOpen)}
        aria-expanded={isOpen}
        aria-controls={contentId}
      >
        {title}
        <ChevronDown
          aria-hidden="true"
          className={cn('transition-transform', isOpen && 'rotate-180')}
        />
      </button>
      
      <div
        id={contentId}
        hidden={!isOpen}          // lebih accessible dari display:none via CSS
        role="region"
        aria-label={title}
      >
        {children}
      </div>
    </div>
  )
}
```

---

## Keyboard Navigation

Semua fitur harus bisa diakses hanya dengan keyboard.

### Focus management di Modal

```tsx
'use client'
import { useEffect, useRef } from 'react'

function Modal({ isOpen, onClose, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null)
  const previousFocusRef = useRef<HTMLElement | null>(null)

  useEffect(() => {
    if (isOpen) {
      // Simpan focus sebelum modal buka
      previousFocusRef.current = document.activeElement as HTMLElement
      
      // Pindahkan focus ke modal
      modalRef.current?.focus()
    } else {
      // Kembalikan focus ke element sebelumnya saat modal tutup
      previousFocusRef.current?.focus()
    }
  }, [isOpen])

  // Tutup dengan Escape
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Escape' && isOpen) onClose()
    }
    document.addEventListener('keydown', handleKeyDown)
    return () => document.removeEventListener('keydown', handleKeyDown)
  }, [isOpen, onClose])

  if (!isOpen) return null

  return (
    <div
      ref={modalRef}
      role="dialog"
      aria-modal="true"
      tabIndex={-1}              // bisa di-focus tapi tidak ada di tab order
      className="focus:outline-none"
    >
      {children}
    </div>
  )
}
```

### Focus trap di dalam Modal

```tsx
// Gunakan library untuk focus trap — jangan reinvent wheel
import { FocusTrap } from 'focus-trap-react'

function Modal({ isOpen, onClose, children }: ModalProps) {
  if (!isOpen) return null
  
  return (
    <FocusTrap
      focusTrapOptions={{
        initialFocus: '#first-focusable',
        escapeDeactivates: true,
        onDeactivate: onClose,
      }}
    >
      <div role="dialog" aria-modal="true">
        {children}
        <button id="first-focusable">Tombol pertama</button>
        <button onClick={onClose}>Tutup</button>
      </div>
    </FocusTrap>
  )
}
```

### Roving tabindex — keyboard navigation di dalam grup

```tsx
// Pattern untuk navigasi arrow key di toolbar/menubar
function Toolbar({ items }: { items: ToolbarItem[] }) {
  const [activeIndex, setActiveIndex] = useState(0)

  const handleKeyDown = (e: React.KeyboardEvent, index: number) => {
    if (e.key === 'ArrowRight') {
      const next = (index + 1) % items.length
      setActiveIndex(next)
    }
    if (e.key === 'ArrowLeft') {
      const prev = (index - 1 + items.length) % items.length
      setActiveIndex(prev)
    }
  }

  return (
    <div role="toolbar" aria-label="Format teks">
      {items.map((item, i) => (
        <button
          key={item.id}
          tabIndex={i === activeIndex ? 0 : -1}  // roving tabindex
          onClick={item.action}
          onKeyDown={(e) => handleKeyDown(e, i)}
          aria-pressed={item.isActive}
          ref={i === activeIndex ? (el) => el?.focus() : undefined}
        >
          {item.icon}
          <span className="sr-only">{item.label}</span>
        </button>
      ))}
    </div>
  )
}
```

---

## Color Contrast — WCAG Standards

```
WCAG AA (minimum):
  Text normal (< 18pt): rasio 4.5:1
  Text besar (≥ 18pt atau ≥ 14pt bold): rasio 3:1
  UI components dan graphics: rasio 3:1

WCAG AAA (enhanced):
  Text normal: rasio 7:1
  Text besar: rasio 4.5:1
```

```css
/* Pastikan warna yang dipilih memenuhi contrast ratio */
/* Tool: https://webaim.org/resources/contrastchecker/ */

/* Contoh — jangan hanya pakai warna untuk menyampaikan info */
.error {
  color: hsl(0, 84%, 60%);       /* red — TIDAK CUKUP sendiri */
  border-left: 3px solid currentColor;
  padding-left: 0.5rem;
}

.error::before {
  content: "⚠ ";                  /* tambahkan icon text */
}

/* Atau pakai tekst eksplisit */
<p className="error">
  <span aria-hidden="true">⚠</span>
  <span className="sr-only">Error: </span>
  {errorMessage}
</p>
```

---

## Skip Navigation

Wajib ada untuk halaman dengan banyak navigasi di atas konten.

```tsx
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="id">
      <body>
        {/* Skip link — tersembunyi kecuali saat di-focus */}
        <a
          href="#main-content"
          className="sr-only focus:not-sr-only focus:fixed focus:top-4 focus:left-4 
                     focus:z-50 focus:rounded focus:bg-primary focus:px-4 focus:py-2 
                     focus:text-primary-foreground"
        >
          Langsung ke konten utama
        </a>
        
        <Navbar />
        <main id="main-content" tabIndex={-1}>
          {children}
        </main>
      </body>
    </html>
  )
}
```

---

## Testing Accessibility

### Setup axe-core di Vitest/Jest

```bash
npm install -D @axe-core/react jest-axe
```

```tsx
// __tests__/a11y/Button.test.tsx
import { render } from '@testing-library/react'
import { axe, toHaveNoViolations } from 'jest-axe'
import { Button } from '@/components/ui/button'

expect.extend(toHaveNoViolations)

describe('Button accessibility', () => {
  it('tidak ada violation WCAG', async () => {
    const { container } = render(<Button>Submit</Button>)
    const results = await axe(container)
    expect(results).toHaveNoViolations()
  })

  it('icon button punya accessible label', async () => {
    const { container } = render(
      <button aria-label="Hapus item">
        <TrashIcon aria-hidden="true" />
      </button>
    )
    const results = await axe(container)
    expect(results).toHaveNoViolations()
  })
})
```

---

## CSS Utility Classes Accessibility

```css
/* Visually hidden tapi tetap ada untuk screen reader */
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border-width: 0;
}

/* Tampilkan kembali saat focus (untuk skip links) */
.focus\:not-sr-only:focus {
  position: static;
  width: auto;
  height: auto;
  padding: inherit;
  margin: inherit;
  overflow: visible;
  clip: auto;
  white-space: normal;
}

/* Focus ring yang jelas dan tidak mengganggu desain */
:focus-visible {
  outline: 2px solid hsl(var(--ring));
  outline-offset: 2px;
}
```

---

## Checklist Accessibility

- [ ] Semua halaman punya skip navigation link
- [ ] Satu `<h1>` per halaman, heading hierarchy berurutan (h1 → h2 → h3)
- [ ] Semua form input punya `<label>` yang terhubung via `htmlFor`/`id`
- [ ] Error message form terhubung ke input via `aria-describedby`
- [ ] Semua icon button punya `aria-label`
- [ ] Icon dekoratif punya `aria-hidden="true"`
- [ ] Modal punya focus trap dan kembalikan focus saat tutup
- [ ] Bisa navigasi semua fitur hanya dengan keyboard
- [ ] Color contrast minimal 4.5:1 untuk text normal
- [ ] Informasi tidak disampaikan hanya melalui warna
- [ ] `lang` attribute ada di tag `<html>`
- [ ] `axe-core` dijalankan dan tidak ada critical violation

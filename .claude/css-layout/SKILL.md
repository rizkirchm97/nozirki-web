---
name: css-layout-mastery
description: >
  Panduan CSS layout modern untuk Next.js: Flexbox, CSS Grid, Container Queries, Subgrid,
  dan pola layout yang dipakai di aplikasi production. Gunakan skill ini ketika user bertanya
  tentang kapan pakai Flexbox vs Grid, cara membuat layout responsive tanpa media query
  menggunakan Container Queries, pola Holy Grail layout, Masonry layout di CSS, sticky
  sidebar, overlap layout, atau bagaimana Subgrid membantu alignment antar komponen.
  Aktif juga untuk pertanyaan tentang aspect-ratio, logical properties (inline/block),
  scroll-driven animations, atau masalah layout yang tidak expected di browser tertentu.
---

# CSS Layout Mastery

Skill ini mencakup teknik layout modern yang membuat senior developer mampu membangun layout kompleks tanpa hack atau workaround.

## Prinsip utama

Pilih tool yang sesuai tugasnya: **Flexbox untuk satu dimensi** (baris atau kolom), **Grid untuk dua dimensi** (baris DAN kolom sekaligus). Kesalahan paling umum adalah memakai Flexbox untuk layout yang seharusnya pakai Grid.

---

## Flexbox — Pola yang Paling Sering Dipakai

### Centering universal

```css
/* Centering apapun, cara paling clean */
.center {
  display: flex;
  align-items: center;
  justify-content: center;
}
```

### Navigation bar

```css
.navbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 1.5rem;
}

.navbar-links {
  display: flex;
  align-items: center;
  gap: 0.5rem;           /* lebih clean dari margin */
  list-style: none;
}

/* Logo push ke kiri, nav ke kanan */
.navbar-brand { margin-right: auto; }
```

### Card grid yang auto-wrap

```css
/* Otomatis wrapping tanpa media query */
.card-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 1.5rem;
}

.card-grid > * {
  flex: 1 1 280px;   /* grow, shrink, min-width 280px */
  /* Tidak perlu media query — auto wrap saat tidak muat */
}
```

### Sidebar layout

```css
.layout {
  display: flex;
  min-height: 100vh;
}

.sidebar {
  width: 260px;
  flex-shrink: 0;    /* sidebar tidak mengecil */
}

.main {
  flex: 1;           /* isi sisa ruang */
  min-width: 0;      /* cegah overflow pada children flex */
}
```

---

## CSS Grid — Dua Dimensi

### Responsive grid tanpa media query

```css
/* Auto-fit: kolom sebanyak mungkin sesuai lebar minimum */
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1.5rem;
}

/* Auto-fill: pertahankan slot kosong (ukuran fixed) */
.grid-fill {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 1rem;
}
```

### Named areas — layout yang sangat readable

```css
.page {
  display: grid;
  grid-template-areas:
    "header  header"
    "sidebar main  "
    "footer  footer";
  grid-template-columns: 260px 1fr;
  grid-template-rows: auto 1fr auto;
  min-height: 100vh;
}

.page-header  { grid-area: header; }
.page-sidebar { grid-area: sidebar; }
.page-main    { grid-area: main; }
.page-footer  { grid-area: footer; }
```

### Subgrid — alignment antar komponen yang berbeda

Masalah klasik: card list dengan tinggi judul yang berbeda-beda. Subgrid membuatnya bisa aligned.

```css
/* Parent grid */
.card-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 1.5rem;
}

/* Card dengan subgrid — ikut row dari parent */
.card {
  display: grid;
  grid-row: span 4;                   /* card ambil 4 row dari parent */
  grid-template-rows: subgrid;        /* row mengikuti ukuran parent */
  gap: 0;
  border: 1px solid;
  border-radius: 0.5rem;
  overflow: hidden;
}

/* Sekarang image, title, desc, button semua aligned antar card */
.card-image   { /* row 1 */ }
.card-title   { /* row 2 — semua card sejajar meski panjang berbeda */ }
.card-desc    { /* row 3 */ }
.card-actions { /* row 4 — semua di bawah */ }
```

### Overlap layout

```css
/* Overlay text di atas image */
.hero {
  display: grid;
  grid-template-areas: "hero";  /* satu area, semua isi di sana */
}

.hero > * {
  grid-area: hero;    /* semua child di posisi yang sama = overlap */
}

.hero-image {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.hero-content {
  z-index: 1;
  align-self: end;   /* konten di bawah */
  background: linear-gradient(transparent, rgba(0,0,0,0.7));
  padding: 2rem;
}
```

---

## Container Queries — Responsif Berdasarkan Container

Media query merespons ukuran viewport. Container query merespons ukuran **parent container** — jauh lebih powerful untuk komponen yang reusable.

```css
/* Tandai container */
.card-container {
  container-type: inline-size;
  container-name: card;    /* opsional, untuk targeting spesifik */
}

/* Styling berdasarkan ukuran container, bukan viewport */
.card {
  padding: 1rem;
}

@container card (min-width: 400px) {
  .card {
    display: grid;
    grid-template-columns: 150px 1fr;
    gap: 1rem;
  }
  
  .card-image {
    aspect-ratio: 1;
  }
}
```

```tsx
// Di React/Next.js dengan Tailwind (via @tailwindcss/container-queries plugin)
<div className="@container">
  <div className="flex flex-col @md:flex-row @md:items-center gap-4">
    <img className="w-full @md:w-40 aspect-video @md:aspect-square object-cover" />
    <div className="flex-1">
      <h3 className="text-base @md:text-lg font-semibold">{title}</h3>
    </div>
  </div>
</div>
```

---

## Pola Layout Spesifik

### Sticky sidebar dengan scrollable main

```css
.layout {
  display: grid;
  grid-template-columns: 260px 1fr;
  align-items: start;     /* penting! tanpa ini sidebar stretch ke tinggi main */
  gap: 2rem;
}

.sidebar {
  position: sticky;
  top: 1rem;              /* jarak dari atas viewport */
  max-height: calc(100vh - 2rem);
  overflow-y: auto;
}
```

### Full-bleed section di dalam container

```css
.content {
  max-width: 65ch;         /* lebar optimal untuk bacaan */
  margin-inline: auto;
  padding-inline: 1rem;
}

/* Elemen tertentu melebar ke full viewport */
.full-bleed {
  width: 100vw;
  margin-inline: calc(-50vw + 50%);
}
```

### Masonry layout (CSS-only, experimental)

```css
/* Masih experimental, cek browser support sebelum pakai */
.masonry {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  grid-template-rows: masonry;  /* CSS Masonry */
  gap: 1rem;
}

/* Fallback dengan columns */
.masonry-fallback {
  columns: 3 200px;
  gap: 1rem;
}

.masonry-fallback > * {
  break-inside: avoid;
  margin-bottom: 1rem;
}
```

---

## Logical Properties — i18n-ready

Gunakan logical properties agar layout otomatis flip untuk RTL languages (Arab, Ibrani).

```css
/* Physical (lama) */
.old {
  margin-left: 1rem;
  padding-right: 0.5rem;
  border-left: 2px solid;
}

/* Logical (baru — benar untuk RTL) */
.new {
  margin-inline-start: 1rem;   /* kiri di LTR, kanan di RTL */
  padding-inline-end: 0.5rem;
  border-inline-start: 2px solid;
}

/* Tailwind equivalent */
.tw { @apply ms-4 pe-2 border-s-2; }
```

---

## Aspect Ratio

```css
/* Modern — pakai ini */
.video { aspect-ratio: 16 / 9; }
.square { aspect-ratio: 1; }
.portrait { aspect-ratio: 3 / 4; }

/* Responsive image container */
.image-container {
  aspect-ratio: 16 / 9;
  overflow: hidden;
  border-radius: 0.5rem;
}

.image-container img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}
```

---

## Scroll Behavior

```css
/* Smooth scroll ke anchor */
html { scroll-behavior: smooth; }

/* Scroll snap — carousel atau section-by-section */
.carousel {
  display: flex;
  overflow-x: auto;
  scroll-snap-type: x mandatory;
  scroll-padding-inline-start: 1rem;
  -webkit-overflow-scrolling: touch;
}

.carousel-item {
  flex-shrink: 0;
  width: 80%;
  scroll-snap-align: start;
}

/* Cegah overscroll yang pindah ke parent */
.modal-content {
  overscroll-behavior: contain;
}
```

---

## Tabel keputusan: Flexbox vs Grid

| Situasi | Pilihan |
|---|---|
| Row atau kolom saja (nav, toolbar, chip list) | Flexbox |
| Grid dua dimensi (card list, dashboard, form) | Grid |
| Ukuran item ditentukan kontennya | Flexbox |
| Ukuran item ditentukan container/grid | Grid |
| Item perlu overlap | Grid |
| Layout yang perlu named areas | Grid |
| Alignment satu aksis saja | Flexbox |
| Komponen reusable yang responsif | Container Queries |

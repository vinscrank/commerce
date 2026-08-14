# Commerce — Specifiche di progetto

## Obiettivo

Base riutilizzabile per ecommerce **headless Shopify**, costruita su Next.js Commerce (integrazione Shopify Storefront API già presente). Ogni progetto futuro parte da questa base e monta sopra **design e contenuti editoriali personalizzati**, senza riscrivere la logica commerce.

**Stack target:**

| Layer | Tecnologia | Ruolo |
|-------|-----------|-------|
| Frontend | Next.js 15 (App Router) | UI, SSR, Server Actions, caching |
| Commerce | Shopify Storefront API (GraphQL) | Prodotti, varianti, carrello, checkout |
| CMS | Sanity | Contenuti editoriali, layout pagine, SEO, blocchi marketing |
| Styling | Tailwind CSS 4 | Design system per-progetto |

---

## Principi architetturali

1. **Shopify resta source of truth per il commerce** — catalogo, prezzi, inventario, carrello, checkout.
2. **Sanity gestisce i contenuti non-commerce** — homepage editoriale, landing, blog, banner, testi marketing, metadata custom, blocchi modulari.
3. **`lib/shopify/` non va modificato** salvo bugfix o upgrade API. Tutta la logica Shopify vive qui.
4. **I componenti UI sono sostituibili** — ogni progetto può avere design diverso mantenendo gli stessi data fetcher.
5. **Separazione dati / presentazione** — le page fetchano dati (Shopify + Sanity), i componenti ricevono props tipizzate.

---

## Cosa esiste gia (base Shopify)

### Layer dati — `lib/shopify/`

Integrazione completa con Shopify Storefront API via GraphQL.

**Funzioni esportate da `lib/shopify/index.ts`:**

| Funzione | Descrizione |
|----------|-------------|
| `shopifyFetch` | Client GraphQL generico verso Storefront API |
| `createCart` | Crea un nuovo carrello |
| `addToCart` | Aggiunge varianti al carrello |
| `removeFromCart` | Rimuove linee dal carrello |
| `updateCart` | Aggiorna quantita |
| `getCart` | Recupera carrello (cookie `cartId`) |
| `getCollection` | Singola collection per handle |
| `getCollectionProducts` | Prodotti di una collection con sort/filter |
| `getCollections` | Lista collections |
| `getMenu` | Menu di navigazione Shopify |
| `getPage` | Pagina Shopify per handle |
| `getPages` | Lista pagine Shopify |
| `getProduct` | Prodotto per handle |
| `getProductRecommendations` | Prodotti correlati |
| `getProducts` | Lista prodotti con sort/filter |
| `revalidate` | Webhook handler per invalidazione cache |

**Struttura interna:**

```
lib/shopify/
  index.ts          # Client + funzioni pubbliche
  types.ts          # Tipi TypeScript (Product, Cart, Collection, ...)
  fragments/        # GraphQL fragments riutilizzabili
  queries/          # Query GraphQL (product, cart, collection, menu, page)
  mutations/        # Mutations GraphQL (cart)
```

### Funzionalita Shopify da preservare

- Catalogo prodotti con varianti e opzioni
- Collections con filtri e ordinamento
- Carrello persistente (cookie `cartId`)
- Server Actions per add/remove/update cart
- Redirect a Shopify Checkout (`checkoutUrl`)
- Menu di navigazione da Shopify Admin
- Pagine statiche Shopify (`/[page]`)
- SEO metadata da Shopify (title, description, OG images)
- Tag `nextjs-frontend-hidden` per escludere prodotti dall'indice
- Webhook revalidation (`POST /api/revalidate`) per prodotti e collections
- Cache Next.js con tag (`products`, `collections`, `cart`)
- Immagini da `cdn.shopify.com` (configurate in `next.config.ts`)

### Route App Router esistenti

```
app/
  page.tsx                          # Homepage (ThreeItemGrid + Carousel)
  product/[handle]/page.tsx         # Scheda prodotto
  search/page.tsx                   # Ricerca prodotti
  search/[collection]/page.tsx      # Prodotti per collection
  [page]/page.tsx                   # Pagine Shopify
  api/revalidate/route.ts           # Webhook Shopify
  sitemap.ts                        # Sitemap dinamica
  robots.ts                         # Robots.txt
```

### Componenti UI (sostituibili per-progetto)

```
components/
  cart/           # Carrello (modal, context, actions)
  grid/           # Griglie prodotti
  layout/         # Navbar, footer, search, filtri
  product/        # Gallery, variant selector, description
```

### Server Actions carrello — `components/cart/actions.ts`

- `addItem` — aggiunge variante al carrello
- `removeItem` — rimuove dal carrello
- `updateItemQuantity` — aggiorna quantita
- `redirectToCheckout` — redirect a Shopify Checkout
- `createCartAndSetCookie` — inizializza carrello

---

## Integrazione Sanity (da implementare)

### Ruolo di Sanity

Sanity **non sostituisce** Shopify per prodotti/prezzi/carrello. Gestisce:

- Layout homepage editoriale (hero, banner, sezioni promozionali)
- Landing page marketing
- Blog / articoli
- Testi globali (footer, newsletter, trust badges)
- Blocchi modulari (Portable Text + custom blocks)
- SEO override per pagine editoriali
- Configurazione theme per-progetto (colori, font, logo)

### Struttura prevista

```
lib/sanity/
  client.ts           # Sanity client (GROQ fetch)
  queries/            # Query GROQ per document type
  types.ts            # Tipi TypeScript per documenti Sanity
  image.ts            # Helper per Sanity Image URL builder

sanity/
  schemas/            # Schema Sanity (document types, blocks)
  sanity.config.ts    # Config studio
  sanity.cli.ts       # CLI config
```

### Pattern di composizione pagina

Le page fetchano **entrambe le sorgenti** e passano dati ai componenti:

```tsx
// Esempio: homepage futura
export default async function HomePage() {
  const [sanityPage, featuredProducts] = await Promise.all([
    getSanityPage("home"),
    getCollectionProducts({ collection: "frontpage" }),
  ]);

  return (
    <>
      <SanityPageBuilder blocks={sanityPage.blocks} />
      <ProductGrid products={featuredProducts} />
    </>
  );
}
```

### Document types Sanity previsti

| Schema | Descrizione |
|--------|-------------|
| `page` | Pagina editoriale con blocchi modulari |
| `homePage` | Homepage con sezioni configurabili |
| `globalSettings` | Logo, social, footer text, SEO default |
| `navigation` | Link navigazione custom (integrati con menu Shopify) |
| `blogPost` | Articoli blog |
| `blockHero` | Hero banner |
| `blockProductCarousel` | Carousel prodotti (handle collection Shopify) |
| `blockRichText` | Contenuto Portable Text |
| `blockImageBanner` | Banner immagine con CTA |

### Regola chiave Sanity + Shopify

Quando un blocco Sanity referenzia prodotti o collections, usa **handle Shopify** (stringa), non ID. Il fetch dei dati prodotto avviene sempre tramite `lib/shopify/`.

---

## Strategia design per-progetto

Questa repo e la **base tecnica condivisa**. Per ogni nuovo ecommerce:

1. Clona/fork questa repo
2. Configura `.env` con credenziali Shopify + Sanity del cliente
3. Sostituisci/override i componenti in `components/` con il design del cliente
4. Configura schema Sanity e contenuti editoriali
5. Mantieni intatto `lib/shopify/` e `components/cart/actions.ts`

**Cosa si personalizza:**

- Componenti UI (`components/`)
- Stili globali (`app/globals.css`, Tailwind theme)
- Layout pagine (`app/`)
- Schema e contenuti Sanity
- Font, colori, animazioni

**Cosa NON si tocca:**

- `lib/shopify/` (layer dati commerce)
- `components/cart/actions.ts` (Server Actions carrello)
- `app/api/revalidate/route.ts` (webhook)
- Tipi in `lib/shopify/types.ts`

---

## Variabili d'ambiente

### Shopify (obbligatorie)

```env
SHOPIFY_STORE_DOMAIN="[store].myshopify.com"
SHOPIFY_STOREFRONT_ACCESS_TOKEN=""
SHOPIFY_REVALIDATION_SECRET=""
COMPANY_NAME=""
SITE_NAME=""
```

### Sanity (da aggiungere)

```env
NEXT_PUBLIC_SANITY_PROJECT_ID=""
NEXT_PUBLIC_SANITY_DATASET="production"
SANITY_API_TOKEN=""
```

---

## Convenzioni di sviluppo

### Fetch dati

- **Shopify**: sempre tramite funzioni in `lib/shopify/index.ts`, mai query GraphQL dirette nei componenti
- **Sanity**: tramite funzioni in `lib/sanity/`, con GROQ queries tipizzate
- Usare `Promise.all` per fetch paralleli quando una page usa entrambe le sorgenti

### Caching

- Shopify usa `unstable_cacheTag` / `unstable_cacheLife` (Next.js cache)
- Tag cache: `products`, `collections`, `cart` (definiti in `lib/constants.ts`)
- Sanity: configurare revalidation via webhook Sanity o ISR con tag dedicati

### Componenti

- Server Components di default
- `"use client"` solo per interattivita (carrello, filtri, menu mobile)
- Server Actions in file dedicati (`actions.ts`)

### TypeScript

- Tipi Shopify in `lib/shopify/types.ts` — non duplicare
- Tipi Sanity in `lib/sanity/types.ts`
- Props componenti sempre tipizzate

### Styling

- Tailwind CSS 4 con `@tailwindcss/postcss`
- Plugin: container-queries, typography
- Font: Geist Sans (sostituibile per-progetto)
- Dark mode supportato via classi Tailwind

---

## Feature Next.js in uso

- **App Router** con layout annidati
- **React Server Components** per data fetching
- **Server Actions** per mutazioni carrello
- **Suspense** per loading states
- **PPR** (Partial Prerendering) — abilitato in `next.config.ts`
- **useCache** — caching sperimentale Next.js
- **generateMetadata** per SEO dinamico
- **Dynamic OG images** (`opengraph-image.tsx`)

---

## Roadmap

### Fase 1 — Base (stato attuale)

- [x] Integrazione Shopify Storefront API completa
- [x] Carrello con Server Actions
- [x] Catalogo, collections, search, filtri
- [x] Checkout redirect Shopify
- [x] Webhook revalidation
- [x] SEO e sitemap

### Fase 2 — Sanity CMS

- [ ] Setup Sanity studio e schema
- [ ] Client GROQ in `lib/sanity/`
- [ ] Page builder con blocchi modulari
- [ ] Homepage editoriale via Sanity
- [ ] Global settings (logo, footer, SEO)
- [ ] Webhook revalidation Sanity

### Fase 3 — Primo progetto cliente

- [ ] Design system custom
- [ ] Override componenti UI
- [ ] Contenuti editoriali in Sanity
- [ ] Deploy Vercel con env per-progetto

---

## Comandi

```bash
pnpm install    # Installa dipendenze
pnpm dev        # Dev server (Turbopack)
pnpm build      # Build produzione
pnpm start      # Avvia build produzione
```

## Repository

- GitHub: https://github.com/vinscrank/commerce
- Base originale: https://github.com/vercel/commerce (Shopify provider)

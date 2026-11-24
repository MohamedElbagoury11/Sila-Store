# SILA frontend redesign plan

Planning document only. No application code, style, package, or backend change accompanies it.

Audit date: 24 November 2026. Audited commit: `0603a73` plus the uncommitted working copy.

## Contents

1. Executive summary
2. Current architecture overview
3. Existing UI/UX audit
4. SILA brand analysis
5. Proposed design system
6. Page-by-page redesign specifications
7. Component architecture proposal
8. File-level modification plan
9. Responsive strategy
10. Implementation phases
11. Testing and regression strategy
12. Risks and dependencies
13. Acceptance criteria for each phase
14. Prioritized task checklist
15. Missing information, assets, and open decisions

---

## 1. Executive summary

The storefront is a Create React App project with React 19, TypeScript, Tailwind CSS 3.4, React Router 7, TanStack Query, and i18next with English and Arabic. It works end to end for browsing, cart, cash-on-delivery checkout, wishlist, reviews, orders, and notifications. The architecture is sound for a store this size and the plan keeps it.

The visual layer has no design system. `tailwind.config.js` has an empty theme, no fonts are loaded, and 23 files hardcode Tailwind's default blue, pink, green, yellow, red, and purple. The brand name "All She Needs" appears in 15 files. Most components use large radii, heavy shadows, and glass blur, which is the opposite of the SILA card.

The plan has four parts.

1. Add design tokens to Tailwind and load four fonts. Every later change reads from these tokens.
2. Replace the brand identity in one pass: logo, name, tagline, social links, favicon, manifest, SEO.
3. Restyle the shared components, then the pages, in order of revenue impact: product card, listing, product page, cart, checkout, then account and content pages.
4. Finish Arabic and RTL, accessibility, and performance as a gated pass.

No API call, request payload, context, or route changes. The plan adds no npm dependency.

The audit found problems that a restyle alone would hide. The owner must decide on these before implementation starts.

- A product on sale shows its discounted price on the product card and its full price on the product page, in the cart, and in the order total sent to the backend.
- Products have no size, colour, or variant data in the storefront, the type definitions, or the admin product form. The brief asks for a variant and size design. That needs backend and admin work first.
- The working copy points the API at `http://localhost:3000`. The committed value is the Railway URL.
- The SILA logo exists only inside a JPEG photo of the card. Implementation needs a vector or transparent file.
- Several visible controls do nothing: pagination, "Buy again", the contact form, three profile menu items, "Save changes", and links to `/privacy` and `/forgot-password`.

Section 15 lists every missing asset and open decision.

---

## 2. Current architecture overview

### 2.1 Stack

| Area | What the code uses | Version |
|---|---|---|
| Build | Create React App, `react-scripts` | 5.0.1 |
| UI library | React, React DOM | 19.2 |
| Language | TypeScript, `strict` on | 4.9 |
| Styling | Tailwind CSS through PostCSS and Autoprefixer | 3.4 |
| Routing | `react-router-dom`, `BrowserRouter` with declarative routes | 7.13 |
| Server data | `@tanstack/react-query` | 5.64 |
| HTTP | `axios`, one shared instance | 1.14 |
| Forms | `react-hook-form` in Login, Register, Checkout | 7.54 |
| Animation | `framer-motion` in Home, ProductCard, ProductDetails | 12 |
| Icons | `lucide-react` | 0.470 |
| i18n | `i18next`, `react-i18next`, browser language detector | 23 and 14 |
| SEO | `react-helmet-async` | 3 |
| Class helpers | `clsx`, `tailwind-merge` through `src/utils/cn.ts` | |
| Hosting | Netlify, Node 20, `CI=false`, SPA redirect in `public/_redirects` | |

### 2.2 Project structure

```
src/
  api/axios.ts              shared axios instance, token interceptor
  assets/                   OnlyLogo.png in use, home.webp unused
  components/
    cart/CartRecommendations.tsx
    common/                 InstallBanner, LanguageSwitcher, Logo, PageLoader,
                            ProtectedRoute, SEO, ScrollToTop
    layout/                 Navbar, Footer
    product/                ProductCard, ReviewForm
    ui/                     Badge, Button, Card, Input
  context/                  Auth, Cart, Wishlist, Notification
  hooks/useInstallPrompt.ts
  i18n/                     config.ts, locales/en.json, locales/ar.json
  layouts/MainLayout.tsx
  pages/                    17 pages
  routes/AppRoutes.tsx
  types/index.ts
  utils/cn.ts
```

`src/App.css`, `src/logo.svg`, and `src/assets/home.webp` are not imported anywhere.

### 2.3 Routing

All routes render inside `MainLayout`, which stacks `InstallBanner`, `Navbar`, a centred `main` with `max-w-7xl` and padding, and `Footer`.

| Path | Page | Access |
|---|---|---|
| `/` | Home | public |
| `/products` | Products, reads `category` and `search` query params | public |
| `/products/:id` | ProductDetails | public |
| `/cart` | Cart | public |
| `/login`, `/register` | Login, Register | public |
| `/wishlist` | Wishlist | public |
| `/search` | SearchResults, reads `q` | public, no link points to it |
| `/notifications` | Notifications | public route, API needs a token |
| `/about`, `/contact`, `/faq`, `/terms` | content pages | public |
| `/checkout`, `/orders`, `/profile` | Checkout, Orders, Profile | `ProtectedRoute` |
| `/admin` | 8-line stub that prints "Admin Dashboard" | `ProtectedRoute`, no role check |

No route exists for `/privacy` or `/forgot-password`, although the footer and the login page link to them. There is no catch-all 404 route.

### 2.4 State management

| State | Where it lives | Persistence |
|---|---|---|
| User and token | `AuthContext` | `localStorage` keys `token` and `user` |
| Cart | `CartContext` | `localStorage` key `cart_<userId>` or `cart_guest` |
| Wishlist | `WishlistContext` | backend only, loaded when a token exists |
| Toasts | `NotificationContext`, which also renders the toast UI | none |
| Products, categories, orders, notifications | TanStack Query | in-memory cache |

### 2.5 API integration

`src/api/axios.ts` creates one axios instance. A request interceptor adds `Authorization: Bearer <token>` from `localStorage`. A response interceptor removes `token` and `user` from `localStorage` on a 401 but does not update `AuthContext`.

Most callers unwrap the response with `data.data ? data.data : data`, so the code tolerates two response shapes. Responses are mostly typed as `any`.

The storefront makes these calls. The redesign must leave every one unchanged.

| Method and path | Caller | Params or body |
|---|---|---|
| `POST /auth/login` | Login | `{ email, password }`, expects `{ token, user }` |
| `POST /auth/register` | Register | `{ name, email, password }`, expects `{ token, user }` |
| `GET /products` | Home | `limit=8` |
| `GET /products` | Products | `category`, `sort`, `search` |
| `GET /products` | CartRecommendations | `category` and `limit=10`, then `limit=10` |
| `GET /products/:id` | ProductDetails | |
| `GET /categories` | Home, Products | |
| `POST /reviews` | ReviewForm | `{ product: { id }, rating, comment }` |
| `GET /wishlists` | WishlistContext | |
| `POST /wishlists` | WishlistContext | `{ productId }` |
| `DELETE /wishlists/:productId` | WishlistContext | |
| `POST /orders` | Checkout | `{ shipping: { fullName, email, address, city, phone }, items: [{ productId, quantity }], total }` |
| `GET /orders` | Orders | |
| `GET /notifications` | Navbar every 2 minutes, Notifications | |
| `PATCH /notifications/mark-all-read` | Notifications | |
| `PATCH /notifications/:id/read` | Notifications | |

### 2.6 Authentication flow

Login and Register post credentials, store the token and user through `AuthContext.login`, show a toast, and navigate to `/`. `ProtectedRoute` redirects to `/login` when there is no token. Logout exists only on the Profile page.

Reading the code suggests one defect that affects regression testing. `AuthProvider` loads the token inside `useEffect` at lines 19 to 27, so the first render has no token. A hard refresh on `/checkout`, `/orders`, or `/profile` should therefore redirect a logged-in user to `/login`. Confirm this in the browser before the redesign so nobody blames the redesign for it.

### 2.7 Internationalization and direction

`src/i18n/config.ts` loads `en.json` and `ar.json`, detects the language from `localStorage` then the browser, and sets `dir` and `lang` on `<html>` when the language changes. Product and category names come from `nameAr` or `nameEn` with `name` as fallback. The same ternary expression is copied into at least eight places.

### 2.8 Admin dashboard boundary

The storefront holds only a stub at `/admin`. The real admin is a separate Vite and Tailwind 4 project in the sibling folder `admin`, with its own login, its own `admin_token` storage key, and its own routes for dashboard, products, categories, orders, reviews, and notifications.

The admin project is out of scope. This plan changes nothing in it. The plan reads it for one purpose, which is to learn what product data exists. The admin product form in `admin/src/components/products/ProductModal.tsx` edits name, description, price, stock, category, discount price, an on-sale switch, and images. It has no field for size, colour, variant, featured, trending, or specifications.

### 2.9 Backend

The backend is not in this workspace. The committed API base URL is `https://allsheneedsback-production-a650.up.railway.app`. This plan treats the backend as fixed.

---

## 3. Existing UI/UX audit

### 3.1 Functionality that exists today

- Browse products with category filter, sort by newest, price, or rating, and text search from the navbar.
- Product page with image gallery, quantity, add to cart, wishlist toggle, description, specifications, and approved reviews.
- Review submission for logged-in users, pending admin approval.
- Cart stored per user in the browser, with quantity change and removal.
- Cart recommendations from the same categories, plus on-sale products.
- Checkout for logged-in users, cash on delivery only, Egyptian phone validation.
- Order history with expandable item lists and a status label.
- Wishlist synced to the backend for logged-in users.
- Notifications with unread count, mark one read, mark all read.
- Register, login, logout.
- English and Arabic with RTL.
- Install prompt banner for browsers that fire `beforeinstallprompt`.
- Static pages: About, Contact, FAQ, Terms.

### 3.2 Items in the brief that the code does not have

- Product variants and sizes. Absent from `src/types/index.ts`, every page, and the admin form.
- Order tracking beyond a status word. Orders have a `status` field and no tracking number, courier, or timeline.
- Filtering beyond category. There is no price, size, or colour filter.
- Working pagination. The buttons at `src/pages/Products.tsx` lines 229 to 235 are static.
- A dedicated search page in use. Navbar search goes to `/products?search=`. The `/search` route reads a different param and never fetches.
- Profile editing, saved addresses, password reset, and a privacy page.

### 3.3 Problems shared by every page

Styling system

- `tailwind.config.js` has `extend: {}`. There are no brand colours, fonts, radii, or shadows defined.
- `src/index.css` contains only the three Tailwind directives. No font is loaded, so Latin and Arabic both use the system font.
- Blue 600 acts as the primary colour in 23 files. Pink, green, yellow, red, and purple appear beside it.
- Radii of `rounded-2xl`, `rounded-3xl`, and `rounded-full`, plus `shadow-xl`, `shadow-2xl`, and `backdrop-blur`, appear on cards, summaries, buttons, and menus.
- Headings use `font-extrabold` and `font-black` in the system sans font.
- Four Tailwind class families in the code produce no CSS because the config lacks the plugin or screen: `xs:` in 3 places, `animate-in` with `slide-in-from-*` and `fade-in`, `prose`, and `hide-scrollbar`.

Layout

- `MainLayout` already applies `max-w-7xl` and horizontal padding. Products, ProductDetails, Cart, Checkout, and Orders apply the same container again, which doubles the side padding on phones.
- About, Contact, FAQ, and Terms contain full-width sections that sit inside the layout container, so they render as boxed panels on a grey page.
- The page background is `bg-gray-50`. Some pages paint `bg-white` blocks on top of it and others do not.

Brand

- `Logo.tsx` renders the old ASN monogram and the text "ALL SHE NEEDS".
- `public/index.html`, `public/manifest.json`, `SEO.tsx`, both locale files, the footer, and five page titles carry the old name.
- Footer and Contact link to the old Facebook group, TikTok, Instagram, WhatsApp number, and `support@allsheneeds.com`.
- Leftovers from a general store template remain: stock Unsplash photos for the hero, category fallback, and product fallback, fallback categories named Electronics and Home & Living, and an unused `getCategoryIcon` function in `Home.tsx`.

Arabic and RTL

- 12 translation keys used in code are missing from `en.json` and 15 from `ar.json`. Examples: `common.success`, `cart.added`, `wishlist.added`, `common.required` in Arabic, `review.verified_purchase` in Arabic.
- `en.json` defines the `review` object twice. The second replaces the first, which drops three keys.
- `ProductCard.tsx` hardcodes English for "Featured", "Trending", "Sale", "Add", and three toast messages. Cart and Checkout print the literal text "+ shipping".
- Physical direction classes are common: `left-3`, `right-0`, `ml-2`, `pl-10`, `text-left`, `text-right`, `border-l`. The mobile search field, the sort dropdown, the FAQ rows, the cart table, and the toast position stay left-to-right in Arabic.
- `Navbar.tsx` reads `document.documentElement.dir` during render seven times instead of using direction-aware classes.
- `uppercase tracking-wider` labels appear about 15 times. Letter spacing breaks the joins between Arabic letters.
- Arabic copy switches between feminine and masculine address.
- Order status and product category print the raw backend string in both languages.

Accessibility

- Icon-only links and buttons in the navbar, footer, and mobile menu have no accessible name.
- Form labels in Login, Register, Checkout, and Profile are not linked to their inputs.
- Quantity buttons in the cart measure 24 px, below the 44 px touch target.
- The FAQ toggle lacks `aria-expanded`. Toasts have no live region.
- Image alt text is generic or empty: "Thumbnail", "item", "Hero Background".
- `PageLoader` and the Sale badge animate forever with no reduced-motion handling.

Performance

- `public/logo192.png` and `public/logo512.png` are the same 1440 by 994 file at 1.7 MB each, and `SEO.tsx` uses one as the default share image.
- The manifest declares `ONLYLOGO.png`, a 574 by 435 image, as both a 192 and a 512 square icon.
- No image has `loading="lazy"`, `width`, or `height`. No route is code-split.
- Every product card mounts two `framer-motion` components for a hover effect that CSS can do.

### 3.4 Page-by-page findings

Header, `src/components/layout/Navbar.tsx`

- No link to Home, Shop, About, Contact, or Orders. The logo is the only route to the home page and nothing in the interface links to `/orders`.
- On phones the header shows the logo with text, the cart, and a menu toggle. Search, wishlist, notifications, language, and account are inside the menu.
- Between 640 and 768 px the icon row and the menu toggle both show, so the same links appear twice.
- The mobile menu is a dropdown panel with coloured tiles, a blue shadow button, and a red heart.
- Search markup is written twice with different styles.

Footer, `src/components/layout/Footer.tsx`

- Old social links and old copyright name.
- Links to `/privacy`, `/products?category=trending`, and `/products?category=new`. The first has no route. The other two filter by a category literally named "trending" or "new", which returns nothing unless such categories exist.
- TikTok uses a music note icon and WhatsApp uses a generic chat icon.

Home, `src/pages/Home.tsx`

- The hero is a boxed, rounded, dark panel with a stock photo, a left-to-right dark gradient, a blue pill, and a 10 second zoom. The gradient direction does not flip in Arabic.
- Hero copy says "Elevate Your Everyday Essentials" and "New Collection 2026".
- Category tiles put white text on a dark scrim over the photo and scale the photo 110% on hover.
- "Featured products" is the first four results of `GET /products?limit=8` and "Trending now" is the next four. Neither uses a featured or trending flag.
- Both grids show the bouncing page loader inside the grid while loading, which shifts the layout.

Product listing, `src/pages/Products.tsx`

- The grid class is `grid-cols-1 xs:grid-cols-2 lg:grid-cols-3`. With no `xs` screen defined, the grid shows one column up to 1024 px. Tablets get a 256 px sidebar beside a single column of cards.
- The sidebar holds one filter group, the category list.
- The result line prints the count twice, for example "Showing results for 12 products in all categories (12)".
- The page number buttons do nothing.
- The active category name in the result line and in the SEO title is the raw backend name.
- Sort resets on every visit because it lives in component state, not in the URL.

Product card, `src/components/product/ProductCard.tsx`

- Glass card with `rounded-2xl`, `shadow-xl`, `backdrop-blur-md`, and a 5 px lift on hover.
- Add to cart, wishlist, and quick view appear only on hover at line 80, so touch users cannot rely on them.
- Yellow "Featured", blue "Trending", and pulsing red "Sale" pills.
- Five yellow stars show even when a product has zero reviews.
- The card lets a shopper add an out-of-stock product. Checkout rejects it later.
- Image ratio is 4:5. The product page uses 1:1.

Product page, `src/pages/ProductDetails.tsx`

- The main image is square, which crops a full-length garment. It has a zoom cursor and a 110% hover zoom but no zoom or lightbox.
- Line 131 prints `product.price` and ignores `onSale` and `discountPrice`.
- The description appears twice, above the buttons and again in the Description tab.
- The Specifications tab always shows, and the admin has no way to enter specifications.
- Every approved review carries a "Verified Purchase" label at line 257. The frontend does not check for a purchase.
- The trust box says "Secure Payment, 100% secure payment" in a store that takes cash on delivery only.
- Quantity has no upper limit tied to stock.
- Tab labels are large and scroll sideways on phones.

Cart, `src/pages/Cart.tsx`

- Product images are hidden below 640 px.
- Unit price, line total, and subtotal use `product.price`. `CartContext` lines 88 to 91 compute the subtotal the same way.
- The shipping row prints "+ shipping" in English in both languages.
- The summary is a glass panel with blur and `shadow-xl`.
- Pressing minus at quantity 1 removes the item with no confirmation.
- Recommendation grids show one column up to 1024 px for the same `xs:` reason.

Checkout, `src/pages/Checkout.tsx`

- Numbered blue circles, a green payment panel, a blue total, and a green submit button. The page uses three accent colours.
- The submit button shows the "Signing in..." string while it places the order.
- Inputs lack `autocomplete`, `inputmode`, and linked labels. Phone and email fields follow page direction in Arabic.
- The order summary sits below the whole form on phones.
- A guest who presses checkout lands on `/login`, and login then sends them to `/`.

Login and Register

- A shadowed white card on a grey page, with placeholders "John Doe" and "m@example.com".
- The "Forgot password?" link leads to a route that does not exist.

Profile, `src/pages/Profile.tsx`

- A solid blue header block.
- "Saved addresses", "Payment methods", and "Account settings" have no handler. "Save changes" has no handler. The phone field is not stored.
- No link to orders, wishlist, or notifications.

Orders, `src/pages/Orders.tsx`

- Status is a pill that is green for delivered and blue for everything else, with the raw status word.
- "Buy again" at line 84 has no handler.
- The details toggle shows "View" when the section is open.
- Item thumbnails are overlapping circles, which hides the garments.

Wishlist, Notifications, Search

- Wishlist shows one column on phones.
- A guest can press the heart, the item shows as saved, and it disappears on reload because the wishlist loads only with a token.
- Notifications use green, purple, and blue icon circles and a blue unread tint.
- `/search` shows a static "no results" screen and has no inbound link.

About, Contact, FAQ, Terms

- All copy describes All She Needs.
- The contact form has no submit handler, so the browser reloads the page.
- The FAQ has two questions. One answer promises tracking details that the interface never shows.
- Terms relies on the `prose` class, which does nothing here.

Shared pieces

- `PageLoader` bounces the old logo over blue and pink pulsing rings with a blue glow.
- `InstallBanner` uses an inline pink gradient and inline styles.
- Toasts are solid green, red, or yellow blocks fixed to the bottom right in both directions.

---

## 4. SILA brand analysis

### 4.1 What the card shows

Only one reference arrived, the photo of the packaging card. The standalone logo file did not. The observations below come from that photo.

- The wordmark reads SILA in a high-contrast display serif with thin hairlines and heavy stems. The S has a long swash that loops below the baseline toward the left. The crossbar of the A is a curved stroke that continues upward as a stem with one open flower and two leaves.
- The flower is line art with no fill. The leaves are solid. The drawing is small relative to the letters and attached to them, not placed beside them.
- The tagline reads "Modest Wear | Designed for you" in a lighter old-style serif with generous letter spacing and a thin vertical bar as separator.
- Everything is black on white. The card has no second colour, no gradient, no shadow, and no frame.
- The composition is centred with wide margins. Roughly two thirds of the card is empty.
- Social icons are solid black glyphs with handles in the same serif: Instagram `@silabrand.eg`, TikTok `@silabrand.eg`, Facebook "SILA سيلا", WhatsApp `01115609753`.
- The Arabic name is سيلا.

### 4.2 What this means for the website

- The page should be quiet enough that the wordmark and the product photos are the darkest, most detailed things on it.
- Contrast comes from black type on a light ground. Colour stays neutral.
- Serif type belongs to headings and the tagline. The card uses no sans serif, but small interface text, prices, and form fields need one for legibility, so the system adds a restrained sans for those.
- The vertical bar separator and thin rules are the brand's only ornaments besides the flower. Hairline dividers can replace boxes and shadows across the site.
- The flower appears once on the card. On the site it should appear rarely.

### 4.3 Where the brief extends the card

The card is strictly black and white. The warm off-white, beige, and nude tones come from the brief, not from the reference. The palette in section 5 follows the brief and keeps those tones pale so the black wordmark still reads as the brand colour. If SILA has packaging, tissue, or label colours, send them and the palette will match them.

### 4.4 Logo rules

- Use the logo file exactly as the brand supplies it. Do not retype it in a font, redraw the flower, recolour it, add effects, or animate it. The current `Logo.tsx` scales the logo on hover. Remove that.
- Keep clear space around it equal to the height of the letter I.
- Minimum height is 24 px on screen. Below that the hairlines and the flower break up.
- The logo needs a transparent background. A white JPEG on the off-white page shows as a rectangle.
- A reversed version, paper colour on ink, is needed only if the logo ever sits on a dark area. This plan has no dark areas, so it is optional.

### 4.5 Botanical detail rules

- Use only artwork that the logo's designer supplies as a separate file. Do not trace the flower from the logo.
- Allowed places: empty states, the footer above the tagline, the order confirmation toast area, and the About page. One per screen at most.
- Never use it as a repeating pattern, a background, a bullet, or a loader.
- If no artwork arrives, use a 1 px rule or the vertical bar from the tagline instead.

---

## 5. Proposed design system

All values become Tailwind theme tokens in `tailwind.config.js`. Components then use token names and never raw HEX values or default Tailwind colours.

### 5.1 Colour

| Token | HEX | Use |
|---|---|---|
| `paper` | `#FAF7F2` | Page background |
| `surface` | `#FFFFFF` | Inputs, drawers, menus, image placeholders on sand areas |
| `sand-100` | `#F3EDE4` | Section bands, summary panels, image placeholder |
| `sand-200` | `#E7DED1` | Hairline dividers, card borders |
| `sand-300` | `#D6C9B8` | Disabled fills, skeleton shimmer |
| `nude-400` | `#C2A78A` | Decorative accents only, never text |
| `nude-600` | `#84664A` | Accent text, links on hover, rating stars |
| `ink-900` | `#1F1D1B` | Headings, body text, primary buttons |
| `ink-700` | `#4A4541` | Secondary text |
| `ink-500` | `#6F6860` | Muted text, placeholders, captions |
| `line` | `#8F877D` | Input borders and other control outlines |
| `success` | `#4A6650` | Success text and icons |
| `error` | `#9F3A32` | Error text, field errors, destructive actions |
| `warning` | `#7D5F28` | Warning text and icons |

Primary colours are `ink-900` and `paper`. Secondary colours are the sand and nude scales. The three status colours are muted on purpose and appear only in feedback messages.

Measured contrast ratios, computed with the WCAG formula:

| Foreground | On `paper` | On `surface` | On `sand-100` |
|---|---|---|---|
| `ink-900` | 15.7 | 16.8 | 14.4 |
| `ink-700` | 8.9 | | 8.1 |
| `ink-500` | 5.1 | 5.5 | 4.7 |
| `nude-600` | 4.9 | 5.3 | 4.5 |
| `line`, needs 3.0 as a control border | 3.3 | 3.5 | |
| `success` | 5.9 | | 5.5 |
| `error` | 6.3 | | 5.8 |
| `warning` | 5.6 | 5.9 | 5.1 |

`nude-400`, `sand-200`, and `sand-300` fail text contrast and must never carry text.

Rules

- No blue, pink, purple, green, or yellow Tailwind defaults anywhere in the storefront.
- No gradients.
- Sale prices use `ink-900` beside a struck-through original in `ink-500`. Red is reserved for errors.
- Selected and active states use an ink underline or an ink border, not a coloured fill.

### 5.2 Typography

Four font families, all under the SIL Open Font License, self-hosted as WOFF2 with `font-display: swap` and `unicode-range` so each visitor downloads only the scripts on screen. No npm package is needed.

| Role | English | Arabic |
|---|---|---|
| Display, for headings and the tagline | Cormorant Garamond 500 and 600 | Noto Naskh Arabic 500 and 600 |
| Text, for body, interface, and prices | Jost 400 and 500 | IBM Plex Sans Arabic 400, 500, and 600 |

Reasons

- Cormorant Garamond is close in character to the tagline on the card. It is a heading face only. Its small x-height makes it hard to read below 20 px.
- Jost is a quiet geometric sans that stays out of the way of the serif and has clear numerals for prices.
- IBM Plex Sans Arabic stays legible at 14 px and has true medium and semibold weights.
- Noto Naskh Arabic gives Arabic headings the same thick and thin contrast as the Latin serif.

The Arabic heading face is a recommendation made without rendering it beside the logo. Phase 1 must render Noto Naskh Arabic and one alternative, El Messiri, with real SILA headings and let the brand owner choose.

Mechanism: `src/index.css` defines `--font-display` and `--font-text` on `:root` and overrides both under `html[lang="ar"]`. Tailwind's `font-display` and `font-sans` point at the variables. Components never choose a font by language.

Type scale

| Token | Font | Desktop size and line height | Mobile size and line height | Use |
|---|---|---|---|---|
| `display` | display 500 | 56 / 60 | 38 / 44 | Home hero |
| `h1` | display 500 | 40 / 48 | 30 / 38 | Page titles, product name |
| `h2` | display 500 | 32 / 40 | 26 / 34 | Section headings |
| `h3` | display 600 | 24 / 32 | 22 / 30 | Sub-sections, summary titles |
| `body-lg` | text 400 | 18 / 30 | 17 / 28 | Lead paragraphs |
| `body` | text 400 | 16 / 26 | 16 / 26 | Default text, inputs |
| `body-sm` | text 400 | 14 / 22 | 14 / 22 | Card names, meta |
| `caption` | text 400 | 12 / 18 | 12 / 18 | Helper text, counts |
| `label` | text 500 | 12 / 16 | 12 / 16 | Small caps labels |
| `button` | text 500 | 14 / 20 | 14 / 20 | Buttons |
| `price` | text 500 | 16 / 24 | 16 / 24 | Prices, with tabular numerals |

Rules

- Headings use weight 500 or 600. Remove `font-extrabold` and `font-black`.
- In English, `label` and `button` use uppercase with 0.12em and 0.06em letter spacing. In Arabic they use neither. One CSS class, `label-caps`, applies both only under `[dir="ltr"]`.
- Arabic text uses 1.75 line height for body and 1.4 for headings, because Arabic letterforms need more vertical room.
- Inputs use 16 px text so iOS does not zoom the page on focus.
- Prices show Western digits in both languages. English shows "EGP 1,250". Arabic shows "1,250 ج.م". Confirm this format with the owner.
- One address form in Arabic copy. The audience is women, so the feminine form is the natural choice.

### 5.3 Spacing

Keep Tailwind's 4 px scale. Use these steps and avoid others.

| Step | Value | Use |
|---|---|---|
| 1 | 4 px | Icon to text gap |
| 2 | 8 px | Tight groups |
| 3 | 12 px | Mobile grid gap, field label to input |
| 4 | 16 px | Mobile page margin, card text block |
| 6 | 24 px | Desktop grid gap, form field spacing |
| 8 | 32 px | Between blocks inside a section |
| 12 | 48 px | Mobile section spacing |
| 16 | 64 px | Tablet section spacing |
| 24 | 96 px | Desktop section spacing |

The content container is 1280 px wide at most, with 16 px side margins on phones, 24 px from 640 px, and 40 px from 1024 px. Text-only pages use a 720 px measure.

### 5.4 Radius and shadow

- Images, cards, panels, and sections have radius 0.
- Buttons, inputs, and small tags have radius 2 px.
- Only count badges and avatar initials are circles.
- Cards and panels have no shadow. A 1 px `sand-200` border or a `sand-100` fill separates them.
- One shadow exists, `0 8px 24px rgba(31, 29, 27, 0.08)`, for elements that float above the page: the mobile drawer, the search panel, and toasts.
- No `backdrop-blur`.

### 5.5 Buttons

| Variant | Rest | Hover and focus | Use |
|---|---|---|---|
| Primary | `ink-900` fill, `paper` text | fill changes to `ink-700` | One per view: add to bag, checkout, place order, sign in |
| Secondary | transparent, 1 px `ink-900` border, `ink-900` text | fills `ink-900`, text turns `paper` | Continue shopping, view details |
| Ghost | `ink-900` text, no border | `sand-100` fill | Icon buttons, menu rows |
| Link | `ink-900` text with a 1 px underline offset 4 px | underline and text turn `nude-600` | Inline actions, remove, view all |
| Destructive | `error` text, link style | underline | Remove, log out |

Sizes are 40 px, 48 px, and 52 px tall. The smallest touch target is 44 by 44 px, so the 40 px size is for desktop toolbars only. Disabled buttons use `sand-300` fill with `ink-500` text and no pointer events. Focus shows a 2 px `ink-900` outline with a 2 px offset on every interactive element. Loading keeps the button width and swaps the label for a small spinner plus the label in progress form.

The existing `Button` API of `variant` and `size` props stays. `default` maps to Primary, `outline` to Secondary, and `secondary` to a `sand-100` filled button, so current call sites keep compiling.

### 5.6 Inputs and forms

- Height 48 px, `surface` fill, 1 px `line` border, 2 px radius, 16 px text, 16 px horizontal padding.
- Focus: border turns `ink-900` and gains a 1 px ring of the same colour. No blue ring.
- Error: border and helper text use `error`. The helper text sits below the field and is linked with `aria-describedby`.
- Labels sit above the field in `body-sm` weight 500. Every label is linked to its input by `id`.
- Placeholders use `ink-500` and never replace a label.
- Textareas and the sort select share the same border, radius, and focus style.
- Phone and email inputs set `dir="ltr"` so digits and addresses read correctly in Arabic, with text aligned to the start edge.
- Checkout and auth inputs get `autocomplete` and `inputmode` attributes.

### 5.7 Product card

- No border, no shadow, no background, no radius, no lift on hover.
- Image ratio 3:4 on a `sand-100` placeholder, `object-cover`, anchored to the top so heads are not cropped.
- On devices with hover, the second image fades in over 300 ms when `images[1]` exists.
- Wishlist heart at the top end corner of the image, always visible, 44 px target, outline when off and filled `ink-900` when on.
- Add to bag: on hover devices a full-width bar slides up from the bottom edge of the image on hover or keyboard focus. On touch devices a 44 px bag button sits at the bottom end corner of the image at all times.
- A sold-out product shows a "Sold out" label over the image and has no add control.
- Below the image: product name in `body-sm`, two lines at most, then the price. A sale shows the new price followed by the old price struck through.
- A rating shows only when `reviewsCount` is above zero, as one small star, the average, and the count in `caption`.
- Tags are small `label` text on a `paper` chip at the top start corner. Only "Sale" and "New" exist. Nothing pulses.
- The whole card is one link to the product page. The quick view eye icon goes away because it duplicates that link.
- All text comes from translation keys.

### 5.8 Header

- Sticky, `paper` background, 1 px `sand-200` bottom border, no shadow, no blur.
- Height 72 px on desktop and 56 px on phones.
- Desktop: text links at the start for Shop, About, and Contact. The SILA logo in the centre at 36 px tall. Icons at the end for search, language, wishlist, notifications, account, and bag.
- Mobile: menu button at the start, logo in the centre at 28 px tall, search and bag at the end.
- The logo has no text beside it.
- Icons are 20 px with a 1.5 stroke in `ink-900`. Count badges are 16 px `ink-900` circles with `paper` digits.
- Search opens as a panel under the header with one field. It submits to the same `/products?search=` route as today.
- The mobile menu is a full-height drawer from the start edge with a scrim. It lists Shop, the categories from the cached `/categories` query, About, Contact, then Wishlist, Notifications, Orders, and Account or Sign in, then the language switch.

### 5.9 Footer

- `sand-100` background with a 1 px `sand-200` top border.
- First row, centred: the logo at 48 px tall and the tagline "Modest Wear | Designed for you" in the display font.
- Second row: three link columns for Shop, Support, and Legal, which collapse to stacked lists on phones.
- Third row: social icons for Instagram, TikTok, Facebook, and WhatsApp, 20 px, `ink-900`, each with an accessible name.
- Last row: copyright line in `caption`, with the cash on delivery note.
- Links that lead nowhere today are removed or pointed at real routes. See section 15.

### 5.10 Motion

- Durations are 150 ms for colour and border changes, 300 ms for image crossfades, drawers, and accordions.
- Easing is `cubic-bezier(0.2, 0, 0, 1)`.
- Allowed effects: colour change, underline, opacity, and an 8 px slide for drawers and the add bar.
- Removed effects: scale on hover, lift on hover, bounce, ping, pulse, and the 10 second hero zoom.
- `prefers-reduced-motion: reduce` turns every transition into an instant change.
- Hover styles sit behind `@media (hover: hover)` so they do not stick on touch screens.

### 5.11 Product image presentation

- One ratio across the site, 3:4 portrait, for cards, gallery, cart rows, order rows, and category tiles. Full-length garments need the height.
- Recommended upload size is 1500 by 2000 px, JPEG or WebP, under 400 KB.
- Backgrounds should be plain and light, close to `paper` or `sand-100`, with the model centred and 5% headroom.
- The first image is a full-length front view. The second is a detail or back view, because cards show it on hover.
- The site never overlays text or a dark scrim on a product photo.
- Every `img` gets `width`, `height`, `loading="lazy"` except above the fold, and alt text from the product name.
- The admin preview is square today. Tell whoever uploads photos that the storefront crops to 3:4 from the top.

These numbers assume real SILA photos, which have not been supplied. Check the ratio against the first photo set and adjust once, in the token, if the garments need 2:3.

### 5.12 Feedback components

- Toasts use an `ink-900` fill with `paper` text for every type, with a small leading icon for success, error, or warning. They sit at the bottom centre on phones and the bottom end corner on desktop, inside an `aria-live="polite"` region.
- The loader is a thin 2 px `ink-900` bar at the top of the content area for page loads.
- Product grids show skeleton cards, a `sand-100` block at 3:4 with two short bars, so the layout does not jump.
- Empty states are a centred column with an optional botanical mark, a display heading, one line of text, and one secondary button.

---

## 6. Page-by-page redesign specifications

Each entry lists the layout, the hierarchy, the components, the interactions, the mobile behaviour, and what improves. Entries describe presentation only unless they say otherwise.

### 6.1 Header and navigation

- Layout: three zones as in 5.8.
- Hierarchy: the logo first, then the bag, then the text links.
- Components: `Navbar`, new `MobileMenu`, new `SearchPanel`, `Logo`, `LanguageSwitcher`.
- Interactions: the search icon toggles the panel and focuses the field. Escape closes the panel and the drawer. The drawer traps focus and returns it to the menu button on close. Route change closes both.
- Mobile: as in 5.8. Only one set of controls shows at each width.
- Improvements: adds links to Shop and Orders that do not exist today, removes the duplicate controls between 640 and 768 px, flips correctly in Arabic, and gives every icon a name for screen readers.
- Unchanged: `useCart`, `useAuth`, the notifications query and its 2 minute refetch, and `handleSearch`.

### 6.2 Footer

- Layout: as in 5.9.
- Improvements: real SILA contact points, correct brand icons, no dead links, tagline from the card.

### 6.3 Home

- Layout, top to bottom:
  1. Hero that spans the full viewport width. On desktop the photo takes seven of twelve columns and the text sits on `paper` in the other five. In Arabic the two sides swap.
  2. Category tiles.
  3. One product grid of eight.
  4. Brand statement band on `sand-100`.
- Hierarchy: hero heading in `display`, one line of supporting text, one Primary button to `/products`.
- Components: new `Container`, `SectionHeading`, `ProductGrid`, `ProductCard`, `ProductCardSkeleton`.
- Category tiles: 3:4 photo with the name in `h3` and the item count in `caption` below the photo. No scrim, no zoom. Two columns on phones and four on desktop. With three or fewer categories the tiles share the row equally.
- Product grid: the same eight products from the same query. Section 15 asks whether to keep two headings or merge them into one, because the split into "Featured" and "Trending" is arbitrary today.
- Brand band: the tagline, two sentences of About copy, and a Link button to `/about`.
- Interactions: none beyond links and card actions. No carousel.
- Mobile: the hero stacks with the photo on top at 4:5 and the text below. Grids are two columns.
- Improvements: removes the stock photo, the gradient, the blue pill, and the zoom. The hero no longer sits in a rounded box.
- Until a SILA campaign photo exists, the hero is typographic: the tagline in `display` on `sand-100`. Do not ship a stock photo.
- The second hero button to `?category=trending` goes away unless a category with that name exists.

### 6.4 Categories and product listing

- Layout: page title, then a toolbar, then the grid. The sidebar goes away because it holds a single filter.
- Title: the localized category name or "Shop" in `h1`, with the product count in `caption`.
- Toolbar: category tabs at the start, "All" first. The sort select at the end.
- Grid: two columns from the smallest phone, three from 768 px, four from 1024 px.
- Components: `ProductGrid`, `ProductCard`, `ProductCardSkeleton`, `EmptyState`.
- Interactions: a tab calls the existing `handleCategoryChange`. The active tab has an ink underline. Sort calls the existing `setSortParam`.
- Search state: when `search` is present the title reads "Results for" with the term, and a Link button clears the search.
- Mobile: tabs scroll sideways inside the container with the page margin preserved. The sort select takes the full width under the tabs.
- Empty state: heading, one line, and a button that clears the category and search params.
- Improvements: two columns on phones and three on tablets instead of one, a correct count, localized category names, and no dead pagination.
- Unchanged: `fetchProducts`, the query key, the client-side filter and sort fallback, and the URL params.

### 6.5 Product page

- Layout on desktop: gallery in seven columns at the start, information in five columns at the end. The information column sticks below the header while the gallery scrolls.
- Gallery: a vertical rail of 72 px thumbnails beside a 3:4 main image. The active thumbnail has a 1 px ink border.
- Information column, in order:
  1. Breadcrumb: Home, Shop, category.
  2. Product name in `h1`.
  3. Rating summary, only when reviews exist. It scrolls to the reviews section.
  4. Price block. Section 15 covers the sale price rule.
  5. Stock line in text: "In stock" in `ink-700` or "Sold out" in `error`.
  6. A reserved gap for a size selector. Nothing renders there until the data exists.
  7. Quantity stepper and the Primary "Add to bag" button on one row, 52 px tall.
  8. "Add to wishlist" as a Link button with the heart icon.
  9. Description, shown once.
  10. Specifications as a two-column definition list, only when the product has any.
  11. A cash on delivery note, using the existing `checkout.cod_title` and `checkout.cod_desc` strings.
- Reviews: a full-width section below, with the review list in eight columns and the form in four. The form keeps its current logic.
- Components: new `ProductGallery`, `Price`, `QuantityStepper`, `Rating`, `ReviewList`. Existing `ReviewForm`.
- Interactions: thumbnail click sets `activeImage` as today. The main image crossfades in 300 ms. No zoom cursor.
- Mobile: the gallery becomes a swipe strip with CSS scroll snap and a "2 / 5" counter. Information follows in one column. Reviews stack with the form after the list. A sticky bar with the price and "Add to bag" appears once the main button scrolls out of view.
- Improvements: full-length garments are no longer cropped to a square, the description appears once, the empty Specifications tab goes away, the payment claim matches the business, and the tabs that overflow on phones are replaced by stacked sections.
- Structure change to note: the three tabs become stacked sections. No content or function is removed.
- The "Verified Purchase" label stays only if the backend confirms purchases. See section 15.
- Unchanged: the product query, `addToCart`, the wishlist calls, `refetch` after a review.

### 6.6 Cart

- Layout: `h1` title with the item count, then line items in eight columns and the summary in four on desktop.
- Line item: 3:4 thumbnail, 96 px wide on desktop and 80 px on phones. Name linking to the product, unit price, stepper, a "Remove" Link button, and the line total at the end edge. A 1 px `sand-200` rule separates rows.
- Summary: `sand-100` panel with no blur or shadow. Subtotal, a translated shipping line, total, the Primary checkout button, and the cash on delivery note in `caption`.
- Components: `QuantityStepper`, `Price`, `EmptyState`, `CartRecommendations`, `ProductGrid`.
- Interactions: the stepper's minus button is disabled at 1, and "Remove" is the only way to delete a row.
- Mobile: thumbnails show. Each row is thumbnail at the start, text block beside it, stepper and line total on a second line. The summary follows the list.
- Recommendations: two headings in `h2` without coloured underline bars, grids at two columns on phones and four on desktop.
- Improvements: product images on phones, a translated shipping line, a calmer summary, and larger touch targets.
- Unchanged: `CartContext` and the recommendation queries.

### 6.7 Checkout

- Layout: form in seven columns and order summary in five on desktop.
- Form: two sections titled "Contact" and "Delivery", separated by a rule, with no cards and no numbered circles. Fields are the same five: full name, email, address, city, phone.
- Payment: one selected row with an ink border on `sand-100`, the cash on delivery title, and its description. It is the only option, so it is not interactive.
- Summary: `sand-100` panel with thumbnails, names, quantities, line totals, subtotal, shipping line, and total in `h3`. Sticky on desktop.
- Submit: Primary button, full width, 52 px, labelled with the total. While submitting it shows a new "Placing order..." string.
- Components: new `FormField`, `Input`, `Button`, `Price`.
- Mobile: a collapsed summary row at the top shows the total and expands on tap. The form follows. The submit button sits at the end of the form.
- Improvements: one accent instead of three, linked labels, correct keyboard types, readable phone and email fields in Arabic, a correct submitting label, and the total visible before the form on phones.
- Unchanged: the registered field names, the validation rules including the `^01\d{9}$` phone pattern, the stock check, the `POST /orders` payload, `clearCart`, and the redirect to `/orders`.

### 6.8 Login and registration

- Layout: one centred column, 400 px wide, on `paper`. No card, no shadow.
- Hierarchy: title in `h1`, one line of supporting text, fields, Primary button, then the link to the other form.
- Components: `FormField`, `Input`, `Button`.
- Mobile: the same column at full width with page margins.
- Improvements: removes the "John Doe" and "m@example.com" placeholders, links labels to inputs, adds `autocomplete` values so password managers work.
- The "Forgot password?" link is hidden until that feature exists.
- Unchanged: both submit handlers and the redirect to `/`.

### 6.9 Customer account

- Layout: a side list of text links at the start and the content at the end. On phones the list becomes a row of tabs that scrolls sideways.
- Side list: Personal information, Orders, Wishlist, Notifications, Log out. The last four link to routes that exist.
- Content: name and email as read-only text rows, not disabled inputs.
- Improvements: removes the blue block, and gives shoppers a way to reach their orders.
- "Saved addresses", "Payment methods", "Account settings", the phone field, and "Save changes" do nothing today. Section 15 asks whether to hide them until the backend supports them. The plan assumes they are hidden.
- Unchanged: `logout`.

### 6.10 Orders and order status

- Layout: a list of orders, each a block with a 1 px `sand-200` border.
- Header row: order number, date, total, and status.
- Status: a small dot and a translated word. Delivered uses `success`, cancelled uses `error`, and every other state uses `ink-700`.
- Progress line: four steps for pending, processing, shipped, and delivered, derived from the existing `status` field. A cancelled order shows the word only. This is the whole of order tracking, because the data has no tracking number.
- Body: up to four 3:4 thumbnails at 56 px wide and a "+2" count.
- Details: the existing toggle expands the item list. The label reads "View details" or "Hide details".
- Mobile: header fields wrap to two rows. The progress line keeps four steps with short labels.
- Improvements: translated status, visible garments, a correct toggle label, total and status visible on phones.
- "Buy again" has no handler. The plan assumes it is hidden. See section 15.
- Unchanged: the orders query and the `expandedOrderId` state.

### 6.11 Wishlist

- Layout: `h1` title with the count in `caption`, then `ProductGrid` at two, three, and four columns.
- Empty state: `EmptyState` with a button to `/products`.
- Improvement: two columns on phones instead of one.
- A guest's wishlist does not persist today. An optional fix in section 14 sends a guest who taps the heart to `/login`.

### 6.12 Notifications

- Layout: a 720 px column. Rows separated by rules, not cards.
- Row: a 40 px `sand-100` circle with an ink icon, the title, the message, and the time. An unread row has a 6 px ink dot and a title in weight 500.
- "Mark all as read" is a Link button beside the title.
- Improvements: no coloured circles or blue tint, and the dot sits on the correct side in Arabic.
- Unchanged: the query and both mutations.

### 6.13 Search and filtering

- Navbar search keeps its route and its handler and moves into the search panel.
- The listing page shows the search term in its title and offers a clear action.
- Filtering stays category only, presented as tabs.
- The `/search` route is unused. The plan assumes it redirects to `/products?search=` with the same term. See section 15.

### 6.14 About, Contact, FAQ, Terms

- All four use the same shell: a `sand-100` title band that spans the viewport with the title in `h1`, then content in a 720 px measure.
- About: story in two paragraphs, mission as a pull quote in the display font, values as columns divided by rules. One botanical mark is allowed here.
- Contact: WhatsApp, Instagram, TikTok, and Facebook as large Link rows with icons, plus phone and email when SILA provides them. The form has no handler today. The plan assumes the form is hidden and the WhatsApp row is the main action. See section 15.
- FAQ: rows with a plus or minus icon, `aria-expanded`, and a 300 ms height transition.
- Terms: headings in `h3` and body text styled directly, because the `prose` class is not available.
- All four need SILA copy in both languages.

### 6.15 Shared states

- `PageLoader` becomes the top bar described in 5.12. It no longer shows a logo.
- `InstallBanner` becomes a 48 px `sand-100` bar with `ink-900` text, a Secondary button, and a close icon, written in Tailwind classes. The hook is unchanged.
- Toasts follow 5.12.
- The product not found screen uses `EmptyState`.

---

## 7. Component architecture proposal

### 7.1 Existing components to restyle

| Component | Change |
|---|---|
| `ui/Button` | New variant styles from 5.5. Props and variant names stay. |
| `ui/Input` | New styles from 5.6. Props stay. |
| `ui/Card` | Radius 0, no shadow, `sand-200` border. Used less after the redesign. |
| `ui/Badge` | Becomes the small tag from 5.7. |
| `common/Logo` | Renders the supplied SILA file. Drops the text and the hover scale. Keeps the `size` prop. |
| `common/LanguageSwitcher` | Text button showing the other language's name. No globe in blue. |
| `common/PageLoader` | Top bar. Keeps the `fullScreen` prop so call sites compile. |
| `common/InstallBanner` | Tailwind classes, new colours. |
| `common/SEO` | Default site name and image come from the brand config. |
| `layout/Navbar` | New layout. Splits out the drawer and the search panel. |
| `layout/Footer` | New layout, links from the brand config. |
| `product/ProductCard` | New markup from 5.7. Same props. |
| `product/ReviewForm` | New styles. Same logic. |
| `cart/CartRecommendations` | Uses `ProductGrid` and `SectionHeading`. Same queries. |
| `context/NotificationContext` | Only the toast markup changes. The API stays. |

### 7.2 New components

| Component | Purpose | Replaces |
|---|---|---|
| `layout/Container` | Max width and page margins in one place | Five duplicated containers |
| `layout/MobileMenu` | Drawer with focus trap | Inline block in `Navbar` |
| `layout/SearchPanel` | One search form | Two copies in `Navbar` |
| `ui/FormField` | Label, input slot, and error with linked ids | Hand-written groups in four pages |
| `ui/Textarea` | Styled textarea | Two hand-styled textareas |
| `ui/Price` | Formats amount and currency per language, shows sale and original | Four different price formats |
| `ui/QuantityStepper` | Minus, value, plus with 44 px targets | Two copies in Cart and ProductDetails |
| `ui/Rating` | Stars or the compact form | Three hand-written star loops |
| `ui/EmptyState` | Icon or mark, heading, text, action | Six hand-written empty states |
| `ui/SectionHeading` | Heading with an optional link at the end | Repeated flex rows |
| `ui/Skeleton` | Placeholder block | Loader inside grids |
| `product/ProductGrid` | The two, three, four column grid | Six grid class strings |
| `product/ProductCardSkeleton` | Loading card | |
| `product/ProductGallery` | Thumbnails, main image, mobile strip | Inline block in ProductDetails |
| `product/ReviewList` | Approved reviews | Inline block in ProductDetails |
| `cart/OrderSummary` | Subtotal, shipping line, total, and action slot, only if Cart and Checkout end up sharing the markup | Two summary panels |
| `orders/OrderStatus` | Translated status and progress line | Inline pill in Orders |
| `common/SocialLinks` | Four links from the brand config | Inline anchors in Footer |
| `common/icons` | TikTok and WhatsApp glyphs as inline SVG | Stand-in lucide icons |

### 7.3 New non-component modules

| Module | Purpose |
|---|---|
| `src/config/brand.ts` | Brand name in both languages, tagline, social URLs, phone, email. One source for 15 files. |
| `src/utils/localize.ts` | `localizedName` and `localizedDescription` for products and categories. Replaces the repeated ternary. |
| `src/utils/price.ts` | `sellingPrice(product)`, already written inline in `Products.tsx` line 79. Used by `Price`. |

### 7.4 Rules for components

- Colours, fonts, radii, and shadows come from tokens.
- Direction uses logical utilities that Tailwind 3.4 already ships: `ms-`, `me-`, `ps-`, `pe-`, `start-`, `end-`, `text-start`, `text-end`, `border-s`, `border-e`. Icons that point use `rtl:rotate-180`. No component reads `document.documentElement.dir`.
- All user-facing text comes from `t()`. Each new key goes into both locale files in the same change.
- Components take data as props and do not fetch, except where they already do.
- No new npm dependency. The drawer, focus trap, accordion, scroll snap gallery, and sticky bar are small enough to write directly.
- `framer-motion` stays installed. Product cards stop using it.

---

## 8. File-level modification plan

### 8.1 Files to modify

| File | Change | Phase |
|---|---|---|
| `tailwind.config.js` | Add colour, font, type scale, radius, shadow, easing tokens | 1 |
| `src/index.css` | Font faces, CSS variables per language, base styles, `label-caps`, focus ring, reduced motion | 1 |
| `public/index.html` | Title, description, Open Graph tags, theme colour, font preload | 1, 2 |
| `public/manifest.json` | Name, short name, description, theme and background colours, icon list | 2 |
| `src/components/common/Logo.tsx` | SILA logo file | 2 |
| `src/components/common/SEO.tsx` | Defaults from brand config | 2 |
| `src/i18n/locales/en.json`, `ar.json` | Brand strings, missing keys, new keys, duplicate `review` block merged | 2, then every phase |
| `src/components/ui/Button.tsx`, `Input.tsx`, `Card.tsx`, `Badge.tsx` | Restyle | 3 |
| `src/context/NotificationContext.tsx` | Toast markup only | 3 |
| `src/components/common/PageLoader.tsx` | Top bar | 3 |
| `src/layouts/MainLayout.tsx` | `main` becomes full width, background `paper` | 4 |
| `src/components/layout/Navbar.tsx`, `Footer.tsx` | New layout | 4 |
| `src/components/common/InstallBanner.tsx`, `LanguageSwitcher.tsx` | Restyle | 4 |
| All 16 customer pages | Wrap content in `Container`, remove duplicate padding | 4 |
| `src/components/product/ProductCard.tsx` | New card | 5 |
| `src/pages/Products.tsx`, `Wishlist.tsx` | New listing layout | 5 |
| `src/components/cart/CartRecommendations.tsx` | Grid and headings | 5 |
| `src/pages/Home.tsx` | New home | 6 |
| `src/pages/ProductDetails.tsx` | New product page | 7 |
| `src/components/product/ReviewForm.tsx` | Restyle | 7 |
| `src/pages/Cart.tsx`, `Checkout.tsx` | New layouts | 8 |
| `src/pages/Login.tsx`, `Register.tsx`, `Profile.tsx`, `Orders.tsx`, `Notifications.tsx`, `SearchResults.tsx` | New layouts | 9 |
| `src/pages/About.tsx`, `Contact.tsx`, `FAQ.tsx`, `Terms.tsx` | New layouts and copy | 10 |
| `src/routes/AppRoutes.tsx` | Only if the owner approves the `/search` redirect or a 404 route | 9 |

### 8.2 Files to add

- `src/config/brand.ts`
- `src/utils/localize.ts`, `src/utils/price.ts`
- `src/assets/brand/` for the logo files and the optional botanical mark
- `src/assets/fonts/` for the WOFF2 files
- `public/` icon set: `favicon.ico`, `icon-192.png`, `icon-512.png`, `icon-maskable-512.png`, `og-image.jpg`
- The components listed in 7.2

### 8.3 Files that must not change

| File | Reason |
|---|---|
| `src/api/axios.ts` | API layer. The uncommitted localhost URL is the owner's to resolve, separately from this work. |
| `src/context/AuthContext.tsx` | Authentication |
| `src/context/CartContext.tsx` | Cart logic, unless the owner approves the price decision in section 15 |
| `src/context/WishlistContext.tsx` | Wishlist sync |
| `src/components/common/ProtectedRoute.tsx` | Route guard |
| `src/hooks/useInstallPrompt.ts` | Install logic |
| `src/types/index.ts` | Data contracts |
| `src/i18n/config.ts` | Language and direction switching |
| `src/pages/AdminDashboard.tsx` | Admin boundary |
| Everything in the sibling `admin` project | Out of scope |
| `package.json` dependencies | No additions |

### 8.4 Files to leave alone and report

`src/App.css`, `src/logo.svg`, `src/assets/home.webp`, `src/assets/OnlyLogo.png`, `public/ONLYLOGO.png`, `public/ONLYLOGO2.png`, and `public/favicon2.ico` are unused or will become unused. The plan does not delete them. The owner can remove them after launch.

---

## 9. Responsive strategy

### 9.1 Breakpoints

Keep Tailwind's defaults, which the code already uses everywhere.

| Name | Min width | Target |
|---|---|---|
| base | 0 | Phones from 360 px |
| `sm` | 640 px | Large phones, small tablets |
| `md` | 768 px | Tablets, portrait |
| `lg` | 1024 px | Tablets in landscape, laptops |
| `xl` | 1280 px | Desktops, container reaches full width |

Do not add an `xs` screen. Replace the three `xs:grid-cols-2` uses with `grid-cols-2` at base.

### 9.2 Layout by breakpoint

| Element | base | `md` | `lg` and up |
|---|---|---|---|
| Page margin | 16 px | 24 px | 40 px |
| Header | 56 px, menu, logo, search, bag | same | 72 px, links, logo, all icons |
| Product grid | 2 columns, 12 px gap | 3 columns, 24 px gap | 4 columns, 24 px gap |
| Category tiles | 2 columns | 4 columns | 4 columns |
| Product page | 1 column, swipe gallery | 1 column, swipe gallery | 7 and 5 columns, thumbnail rail |
| Cart, Checkout | 1 column | 1 column | 8 and 4, or 7 and 5 columns |
| Footer links | stacked | 3 columns | 3 columns |
| Section spacing | 48 px | 64 px | 96 px |
| `h1` | 30 px | 34 px | 40 px |

### 9.3 Rules

- Design each screen at 360 px first, then widen.
- Touch targets are at least 44 by 44 px with 8 px between neighbours.
- Hover effects apply only under `@media (hover: hover)`.
- Sticky bars at the bottom of the screen add `env(safe-area-inset-bottom)` padding.
- No horizontal scrolling of the page at any width from 320 px. Only the category tabs and the mobile gallery scroll sideways, inside their own containers.
- Images carry `width` and `height` so they reserve space before loading.
- Arabic gets the same layouts mirrored. Check every breakpoint in both directions.
- Test widths are 360, 390, 768, 1024, 1280, and 1440 px.

---

## 10. Implementation phases

Each phase is one pull request that leaves the store working. Section 13 holds the acceptance criteria.

### Phase 0. Prerequisites

- Objective: collect what implementation needs and record the starting state.
- Scope: receive the assets and decisions in section 15. Create a branch. Resolve the uncommitted changes in the working copy. Capture screenshots of every page at 360 and 1280 px in both languages. Record the request list from section 2.5 from a real session with the browser network panel.
- Files: none in `src`.
- Dependencies: brand owner.
- Visual outcome: none.
- Testing: confirm the suspected refresh redirect from 2.6 and the price mismatch from section 1 on the current build, and log both as existing behaviour.

### Phase 1. Design tokens and fonts, P0

- Objective: make the design system available to every component.
- Scope: Tailwind theme tokens, font files and faces, per-language font variables, base text and background colours, focus ring, reduced motion rule, `label-caps` class. Render the two Arabic heading candidates for the owner.
- Files: `tailwind.config.js`, `src/index.css`, `public/index.html`, `src/assets/fonts/*`.
- Dependencies: Phase 0.
- Visual outcome: text switches to the new fonts. Nothing else changes yet because components still use default colour classes.
- Testing: build, load every route in both languages, check the network panel for font files per language.

### Phase 2. Brand identity, P0

- Objective: remove every trace of the old brand.
- Scope: brand config, logo component, favicon and app icons, manifest, default SEO, page titles, brand strings in both locales, social links and contact details, footer copyright, install banner text. Add the 12 and 15 missing translation keys and merge the duplicate `review` block.
- Files: `src/config/brand.ts`, `Logo.tsx`, `SEO.tsx`, `public/index.html`, `public/manifest.json`, icon files, both locale files, the brand strings in `Footer.tsx`, `Contact.tsx`, `InstallBanner.tsx`, and the SEO titles in `Home.tsx`, `Products.tsx`, `ProductDetails.tsx`, `About.tsx`, `Contact.tsx`, `FAQ.tsx`, `Terms.tsx`.
- Dependencies: logo files, social URLs, contact details.
- Visual outcome: SILA logo in the header and footer, SILA in the browser tab and share previews.
- Testing: search the source for the old name and handles. Check the tab title on every route, the installed app name, and each social link.

### Phase 3. Interface primitives, P0

- Objective: restyle and add the small shared components.
- Scope: `Button`, `Input`, `Card`, `Badge`, toast markup, `PageLoader`, and new `Container`, `FormField`, `Textarea`, `Price`, `QuantityStepper`, `Rating`, `EmptyState`, `SectionHeading`, `Skeleton`, `localize.ts`, `price.ts`.
- Files: as listed, under `src/components/ui`, `src/components/layout/Container.tsx`, `src/utils`, `src/context/NotificationContext.tsx`.
- Dependencies: Phase 1.
- Visual outcome: buttons, inputs, and toasts change on every page. Pages look half finished, which is expected on the branch.
- Testing: submit Login, Register, and Checkout forms. Trigger each toast type. Tab through each control and check the focus ring.

### Phase 4. Layout shell, P0

- Objective: new header, footer, and page container.
- Scope: `MainLayout`, `Navbar`, `MobileMenu`, `SearchPanel`, `Footer`, `SocialLinks`, brand icons, `InstallBanner`, `LanguageSwitcher`. Wrap every page in `Container` and remove duplicate padding.
- Files: `MainLayout.tsx`, `Navbar.tsx`, `Footer.tsx`, the new layout components, `InstallBanner.tsx`, `LanguageSwitcher.tsx`, and one wrapper edit in each page.
- Dependencies: Phases 2 and 3.
- Visual outcome: the SILA header and footer on every page, `paper` background, consistent margins.
- Testing: every header and footer link, search submit, cart and notification counts, drawer open, close, and focus, language switch, all at 360, 768, and 1280 px in both directions.

### Phase 5. Product card and listing, P1

- Objective: the browsing experience.
- Scope: `ProductCard`, `ProductGrid`, `ProductCardSkeleton`, `Products` page, `Wishlist` page, `CartRecommendations`.
- Files: those five existing files and two new ones.
- Dependencies: Phase 3, and the price decision.
- Visual outcome: borderless 3:4 cards in a two, three, four column grid, category tabs, working add and wishlist controls on touch.
- Testing: filter by each category, each sort order, a search term, empty results, add to bag and wishlist from a card on a phone, sold-out card, compare `GET /products` params with the Phase 0 record.

### Phase 6. Home, P1

- Objective: the first impression.
- Scope: hero, category tiles, product grid, brand band.
- Files: `Home.tsx`, both locale files.
- Dependencies: Phases 4 and 5, hero photo or approval of the typographic hero, home copy.
- Visual outcome: the layout in 6.3.
- Testing: category tile links, hero button, loading and empty states for both queries, both directions.

### Phase 7. Product page, P1

- Objective: the page where shoppers decide.
- Scope: `ProductGallery`, information column, reviews section, `ReviewList`, `ReviewForm` restyle, mobile sticky bar.
- Files: `ProductDetails.tsx`, `ReviewForm.tsx`, the new product components, locale files.
- Dependencies: Phases 3 and 5, the price decision, the verified purchase decision.
- Visual outcome: the layout in 6.5.
- Testing: gallery with one, two, and many images, quantity, add to bag, wishlist toggle, review submit while logged in and logged out, product not found, sold-out product.

### Phase 8. Cart and checkout, P1

- Objective: the purchase path.
- Scope: `Cart` and `Checkout` layouts, `OrderSummary` extracted for both pages if the markup matches, new submitting string, translated shipping line.
- Files: `Cart.tsx`, `Checkout.tsx`, locale files.
- Dependencies: Phases 3 and 5, the shipping copy decision.
- Visual outcome: the layouts in 6.6 and 6.7.
- Testing: add, change quantity, remove, empty cart, guest redirect, each validation error, a full order against a test backend, the `POST /orders` body compared field by field with the Phase 0 record, cart cleared after success.

### Phase 9. Account pages, P2

- Objective: finish the signed-in area.
- Scope: `Login`, `Register`, `Profile`, `Orders` with `OrderStatus`, `Notifications`, `SearchResults`.
- Files: those six pages, `orders/OrderStatus.tsx`, locale files, and `AppRoutes.tsx` only with approval.
- Dependencies: Phase 3, decisions on hidden controls.
- Visual outcome: the layouts in 6.8 to 6.13.
- Testing: register, log in, wrong password, log out, orders with each status, expand details, mark one and all notifications read.

### Phase 10. Content pages, P2

- Objective: brand story and support.
- Scope: `About`, `Contact`, `FAQ`, `Terms` with SILA copy.
- Files: those four pages, locale files.
- Dependencies: copy in both languages, contact form decision.
- Visual outcome: the layout in 6.14.
- Testing: every link, FAQ keyboard use, both directions.

### Phase 11. Quality gate, P2

- Objective: verify Arabic, accessibility, and performance across the finished site.
- Scope: translation key check, search for physical direction classes and default colour classes, keyboard pass, screen reader spot check, contrast check, image attributes, icon and share image sizes, Lighthouse runs.
- Files: small fixes anywhere in `src`.
- Dependencies: Phases 1 to 10.
- Visual outcome: no intended change.
- Testing: the checks themselves, listed in section 11.

### Phase 12. Optional improvements, P3

- Objective: improvements that add behaviour, each needing separate approval.
- Scope: the P3 items in section 14.
- Dependencies: launch of Phases 1 to 11.

---

## 11. Testing and regression strategy

### 11.1 Starting point

The repository has one test, `src/App.test.tsx`, which looks for the text "learn react" and cannot pass. There is no lint or test step in the Netlify build, and `CI=false` lets warnings through. Regression safety today is manual.

### 11.2 Automated checks with existing tools

`react-scripts` already bundles Jest and the project already has React Testing Library, so these need no new package.

- Replace the broken test with a smoke test that renders `App` and finds the header.
- Cart context: add, increase, decrease, remove, and persistence to `localStorage`.
- Checkout: with axios mocked, submitting a valid form calls `POST /orders` with exactly the body in 2.5.
- Login and Register: with axios mocked, each posts the body in 2.5 and stores the token.
- `Price` and `localize`: unit tests for English and Arabic, with and without a sale.
- A script that lists translation keys used in code and absent from either locale file. The audit used such a script. Run it in every phase.

### 11.3 API contract check

For every phase, repeat the Phase 0 session and compare each request's method, path, query params, and body with the recorded list. Any difference blocks the merge.

### 11.4 Manual regression matrix

Run after Phases 4, 5, 7, 8, and 11, in English and Arabic, at 360 and 1280 px.

| Flow | Steps |
|---|---|
| Browse | Home, category tile, listing, sort, search, product page |
| Guest cart | Add from card, add from product page, change quantity, remove, reload and confirm the cart persists |
| Auth | Register, log out, log in, wrong password error |
| Cart handover | Add as guest, log in, confirm the expected cart shows for that user key |
| Checkout | Validation errors, valid order, toast, redirect to orders, empty cart |
| Orders | List, expand, each status |
| Wishlist | Add, remove, reload while logged in |
| Reviews | Submit while logged in, prompt while logged out |
| Notifications | Unread count, mark one, mark all |
| Language | Switch on every page, reload and confirm it persists |
| Install banner | Show, install, dismiss, on a browser that supports it |

### 11.5 Visual review

Compare each page with its Phase 0 screenshot and with section 6 at 360, 768, and 1280 px in both directions. The brand owner signs off Phases 2, 4, 6, and 7.

### 11.6 Accessibility checks

- Keyboard only: reach and operate every control, with a visible focus ring, and no focus trap outside the drawer.
- Screen reader spot check on the header, a product card, the product page, and checkout.
- Automated audit with Lighthouse or axe in the browser on Home, listing, product page, cart, and checkout.
- Contrast: confirm the ratios in 5.1 in the built site.
- 200% browser zoom at 1280 px without loss of content.

### 11.7 Performance checks

- Lighthouse mobile run on Home, listing, and product page before Phase 1 and after Phase 11. The after scores must not be lower.
- Font payload per language under 200 KB.
- No image in `public` above 300 KB.
- Cumulative layout shift under 0.1 on the listing and product pages.

### 11.8 Build and deploy

- `npm run build` passes with no new warnings in each phase.
- Deploy each phase to a Netlify preview and run the checks there, because the production API URL and the SPA redirect only apply in a real build.

---

## 12. Risks and dependencies

| Risk | Effect | Response |
|---|---|---|
| The logo exists only as a JPEG photo | A white box around the logo on the off-white page, and soft edges on high-density screens | Block Phase 2 on a vector or transparent file. As a stopgap only with approval, keep the header and footer white. |
| Sale price mismatch between card, product page, cart, and order total | The redesign makes prices more prominent, so the mismatch becomes more visible. Customers may see one price and be charged another. | Owner decides the rule before Phase 5. The `Price` component then shows one consistent value. Any change to `CartContext` or the posted total needs backend confirmation. |
| Uncommitted `baseURL` of `http://localhost:3000` | A deploy from the working copy would break the store | Owner resolves it in Phase 0. The redesign does not touch the file. |
| No size or variant data | The product page cannot offer a size choice | Reserve the space. Design the selector only when the backend and admin add the data. |
| Suspected redirect to login on refresh of protected pages | Testers may report it as a redesign defect | Confirm and log it in Phase 0. Fixing it changes `AuthContext` and needs separate approval. |
| Checkout sends `fullName` and `phone`, while the admin order view reads `firstName`, `lastName`, `state`, and `zipCode` | The admin may show blank address fields | Existing behaviour in another project. Report it. Do not change the payload. |
| Backend response shapes are untyped | A layout may assume a field that is absent, such as `order.date` or `cat.count` | Keep every existing fallback. Test with production-like data. |
| Arabic heading font chosen without a visual test | Headings may not sit well beside the logo | Compare two candidates in Phase 1 and get sign-off. |
| Cormorant Garamond reads small | Headings look undersized, or someone uses it for body text | Fixed scale in 5.2, heading use only, checked in review. |
| Product photos vary in ratio and background | A 3:4 crop cuts off heads or hems | Photo guidelines in 5.11. Review the first real set before Phase 5 and adjust the ratio token once if needed. |
| No real SILA photography or copy yet | Home and About would ship with stock images or old text | Typographic hero as fallback. Block Phase 10 on copy. |
| Create React App is no longer maintained | No effect on this work, but Tailwind must stay on version 3 | Do not migrate the build tool or Tailwind in this project. |
| `CI=false` in Netlify | Warnings and type slips reach production | Run the build locally with warnings visible in each phase. |
| Hidden controls and removed dead links change what users see | Someone may expect "Buy again" or the contact form | Each removal is listed in section 15 and needs a yes or no before its phase. |
| The store and the admin share a backend but not a design | The admin keeps its current look | Stated as out of scope. |

Dependencies outside the code: logo files, optional botanical artwork, photos, copy in two languages, social URLs, contact details, and the decisions in section 15.

---

## 13. Acceptance criteria for each phase

Every phase must also meet these common criteria.

- `npm run build` succeeds.
- No request in section 2.5 changes method, path, params, or body.
- No file in section 8.3 changes.
- No new dependency in `package.json`.
- The translation key check reports no missing key.
- The manual flows for the touched pages pass in both languages.

| Phase | Additional acceptance criteria |
|---|---|
| 0 | All P0 assets and decisions in section 15 are received or have an agreed fallback. Screenshots and the request record are stored. The working copy is clean. |
| 1 | Tokens from section 5 exist in the Tailwind config under the listed names. English pages load only Latin font files. Arabic pages load Arabic files. The owner has chosen the Arabic heading font. No functional difference from Phase 0. |
| 2 | A search of `src` and `public` finds no "All She Needs", "allsheneeds", or old social handle. The header and footer show the supplied logo unaltered. The tab title, share title, and installed app name read SILA. Every social link opens the SILA account. App icons are square at their declared sizes and under 100 KB each. |
| 3 | `Button`, `Input`, `Card`, and `Badge` use only token colours. Existing call sites compile without prop changes. Every form field has a linked label. Toasts announce through a live region. Login, Register, and Checkout still submit. |
| 4 | No page has doubled side padding. The header shows one set of controls at every width from 320 to 1440 px. Shop and Orders are reachable from navigation. The drawer traps focus and closes on Escape and on navigation. Search lands on `/products?search=` with the term. Header and footer mirror in Arabic. Cart and notification counts match the data. |
| 5 | The grid shows two columns at 360 px, three at 768 px, and four at 1024 px. A touch user can add to bag and toggle the wishlist from a card. A sold-out product cannot be added from a card. Card, listing, wishlist, and recommendations contain no default Tailwind colour class, no shadow, and no radius above 2 px. Category filter, sort, and search return the same products as before. All card text is translated. |
| 6 | The hero spans the viewport and contains no stock photo. Category tiles link to the same URLs as before. Loading shows skeletons with no layout jump. The page mirrors in Arabic. |
| 7 | The main image is 3:4 and uncropped at the head. The price follows the approved rule. The description appears once. Specifications show only when present. Add to bag adds the chosen quantity. Review submission still posts and refreshes. On a phone the gallery swipes and the sticky bar appears after the main button leaves the screen. |
| 8 | Cart rows show images on phones. The shipping line is translated. Stepper targets measure at least 44 px. The checkout page uses one accent colour. The posted order body matches the Phase 0 record. After a successful order the cart is empty and the user is on `/orders`. Validation messages match the current rules. |
| 9 | Auth pages have no placeholder names. No visible control on Profile or Orders does nothing. Order status is translated for all five values. The details toggle label is correct in both states. Notification actions work as before. |
| 10 | The four pages contain SILA copy in both languages. No form on the site submits to nowhere. FAQ rows expose `aria-expanded`. |
| 11 | No physical direction class remains where a logical one applies. No default Tailwind colour class remains in `src`. Lighthouse accessibility is 95 or higher on the five audited pages. Performance scores are not below the Phase 0 baseline. Every image has dimensions and alt text. |
| 12 | Each item has its own written approval and its own criteria before work starts. |

---

## 14. Prioritized task checklist

### P0. Critical foundation

- [ ] Receive the logo in SVG, or PNG with a transparent background at 2x
- [ ] Decide the sale price rule
- [ ] Resolve the uncommitted `baseURL`, logo PNG, and Contact changes
- [ ] Record baseline screenshots and the API request list
- [ ] Add colour, type, spacing, radius, shadow, and motion tokens to `tailwind.config.js`
- [ ] Add the four font families and the per-language variables to `src/index.css`
- [ ] Choose the Arabic heading font with the owner
- [ ] Create `src/config/brand.ts`
- [ ] Replace the logo, favicon, app icons, manifest, and default SEO
- [ ] Replace brand strings and social links in both locale files and all components
- [ ] Add the missing translation keys and merge the duplicate `review` block in `en.json`
- [ ] Restyle `Button`, `Input`, `Card`, `Badge`
- [ ] Add `Container`, `FormField`, `Textarea`, `Price`, `QuantityStepper`, `Rating`, `EmptyState`, `SectionHeading`, `Skeleton`
- [ ] Add `localize.ts` and `price.ts`
- [ ] Restyle toasts and the page loader
- [ ] Make `MainLayout` full width and wrap every page in `Container`
- [ ] Rebuild `Navbar` with `MobileMenu` and `SearchPanel`
- [ ] Rebuild `Footer` with `SocialLinks` and brand icons
- [ ] Restyle `InstallBanner` and `LanguageSwitcher`
- [ ] Replace the broken `App.test.tsx` with a smoke test

### P1. Core storefront experience

- [ ] Rebuild `ProductCard` with touch-reachable actions and translated text
- [ ] Add `ProductGrid` and `ProductCardSkeleton`
- [ ] Rebuild the listing page with category tabs and a working two, three, four column grid
- [ ] Remove or hide the dead pagination, per decision
- [ ] Apply the grid to Wishlist and cart recommendations
- [ ] Rebuild Home: hero, category tiles, product grid, brand band
- [ ] Rebuild the product page with `ProductGallery`, the information column, and the reviews section
- [ ] Restyle `ReviewForm` and add `ReviewList`
- [ ] Rebuild Cart with images on phones and a translated shipping line
- [ ] Rebuild Checkout with linked labels, input types, and the correct submitting label
- [ ] Add tests for the cart context and the checkout payload

### P2. Secondary pages and refinements

- [ ] Rebuild Login and Register
- [ ] Rebuild Profile with links to Orders, Wishlist, and Notifications
- [ ] Rebuild Orders with `OrderStatus` and translated statuses
- [ ] Rebuild Notifications
- [ ] Redirect or restyle `/search`, per decision
- [ ] Rebuild About, Contact, FAQ, and Terms with SILA copy
- [ ] Add the mobile sticky add-to-bag bar on the product page
- [ ] Replace every physical direction class with its logical form
- [ ] Unify Arabic copy on one address form
- [ ] Add image dimensions, lazy loading, and alt text everywhere
- [ ] Run the accessibility and performance gate

### P3. Optional enhancements

Each item adds behaviour and needs separate approval.

- [ ] Return a shopper to checkout after login
- [ ] Send a guest who taps the wishlist heart to login
- [ ] Add a 404 page
- [ ] Add a privacy policy page, once the text exists
- [ ] Cap product page quantity at stock
- [ ] Keep the sort choice in the URL
- [ ] Add related products to the product page with the existing recommendations component
- [ ] Add a lightbox to the product gallery
- [ ] Wire "Buy again" to the cart
- [ ] Add a size selector, after the backend and admin support sizes
- [ ] Add real pagination, after the backend supports it
- [ ] Split routes into lazy-loaded chunks
- [ ] Fix the suspected redirect on refresh in `AuthContext`

---

## 15. Missing information, assets, and open decisions

### 15.1 Assets needed

| Asset | Needed for | Blocks |
|---|---|---|
| SILA logo as SVG, or transparent PNG at 2x. Black version required, reversed version optional. | Header, footer, icons | Phase 2 |
| Square logo mark or approved crop for the favicon and app icons at 192 and 512 px | Browser tab, installed app | Phase 2 |
| Share image at 1200 by 630 px | Link previews | Phase 2 |
| Botanical detail as a separate SVG from the logo's designer, optional | Empty states, footer, About | None. A rule replaces it. |
| Hero photo, landscape and portrait crops | Home | Phase 6. A typographic hero replaces it. |
| Category photos at 3:4 | Home | Phase 6 |
| Product photos that follow 5.11 | Cards, product page | Phase 5 review |
| Packaging or label colours, if any exist | Palette check | Phase 1 |

### 15.2 Information needed

- Exact URLs for Instagram `@silabrand.eg`, TikTok `@silabrand.eg`, and the Facebook page "SILA سيلا".
- Confirmation that WhatsApp `01115609753` should link as `+201115609753`, and whether it is also the phone number on the Contact page.
- A support email address to replace `support@allsheneeds.com`.
- The Arabic tagline. The card has it in English only.
- Copy in English and Arabic for the home hero, footer description, About, FAQ, and Terms.
- Shipping rule: fee, free threshold, or "paid on delivery", for the shipping line in the cart and checkout.
- Delivery and return information, if the product page should show any.
- Whether the product catalogue stays under the same categories or changes for SILA.
- Whether the production domain and the backend URL change with the rebrand.

### 15.3 Decisions needed

| Decision | Default this plan assumes |
|---|---|
| Sale price rule across card, product page, cart, and order total | Show the discounted price everywhere, after the backend owner confirms what total it expects |
| Sizes and variants | Out of scope until the backend and admin add the data |
| Arabic heading font | Noto Naskh Arabic, pending a side-by-side review with El Messiri |
| Price format | Western digits, "EGP 1,250" and "1,250 ج.م", no decimals for whole amounts |
| "Featured" and "Trending" on Home | Merge into one grid of eight under one heading |
| Static pagination buttons | Hide until the backend paginates |
| "View trending" hero button and the footer's Trending and New Arrivals links | Remove unless categories with those names exist |
| `/privacy` footer link | Remove until the page exists |
| "Forgot password?" link | Hide until the feature exists |
| Profile items with no function | Hide "Saved addresses", "Payment methods", "Account settings", the phone field, and "Save changes" |
| "Buy again" | Hide until defined |
| Contact form | Hide the form and lead with WhatsApp |
| "Verified Purchase" label on reviews | Remove unless the backend verifies purchases |
| `/search` route | Redirect to `/products?search=` |
| Product page tabs | Replace with stacked sections |
| Unused legacy files in section 8.4 | Leave in place |

# Digital Experience Showcase (DXS)

## Canonical implementation brief

This document is the primary implementation prompt for this repository. It defines
the application to build; it does not describe an already implemented application.

### 1. Purpose

Build a CMS/SaaS migration test bed, not a marketing demo. DXS must support
end-to-end verification of migrations such as Contentful → Sanity,
WordPress → Contentful, and Dato → Hygraph without requiring proprietary tools.

Exercise content models, field types, relationships, rich text, assets, UI
patterns, and persistence. Use stable fixture identifiers and representative
content so agents can compare source data, migrated data, and rendered pages.
Check record counts, field values, reference integrity, publication state,
ordering, asset metadata, and rich-text rendering. Platform-specific connectors
are outside this baseline; the data contracts must make them possible.

### 2. Technology stack

- React + Vite + TypeScript.
- Tailwind CSS + shadcn/ui for styling and accessible UI primitives.
- React Router v6 for routing.
- TanStack Query for asynchronous queries, mutations, and cache invalidation.
- Zustand for client state, with local persistence for cart, wishlist, and theme.
- React Hook Form (RHF) + Zod for every form and its validation.
- Recharts for dashboard charts.
- Supabase for optional database, authentication, and asset storage.
- Lucide React for icons; date-fns for date formatting and calculations.

Use released, mutually compatible dependencies. Do not require external CMS
accounts, paid services, or proprietary migration tools to run the application.

### 3. Dual-mode data architecture

Implement the repository interface and mode selection in `src/lib/data/client.ts`.
Keep shared TypeScript models and validation schemas separate from adapters.

- Select the Supabase adapter only when both `VITE_SUPABASE_URL` and
  `VITE_SUPABASE_ANON_KEY` are configured. Otherwise select the mock adapter.
- Allow an explicit mock-mode override, `VITE_DATA_MODE=mock`, so offline tests
  remain possible even when credentials exist. Expose the active mode in the UI.
- Both adapters implement the same typed operations: list, get by id/slug,
  create, update, delete, search, filtering, sorting, and pagination as applicable.
  Use consistent result, validation, not-found, and error semantics.
- All pages/components access application data through this interface using
  TanStack Query. No direct Supabase calls or fixture imports in UI code.
- Normalize database snake_case to application camelCase at the adapter boundary.
  Dates use ISO 8601 strings; references use stable ids, not display labels.
- Mock queries/mutations return promises, support local CRUD and reset-to-seed,
  and persist edits locally. Refreshing must not discard editor or form changes.
- Configured Supabase failures must surface actionable errors, not silently switch
  to mock mode. Provide loading, empty, not-found, and error states in both modes.
- Mock mode must make no runtime requests to databases, CMS services, remote
  images, fonts, or analytics. Bundle assets needed for the offline experience.
- Never expose service-role keys in the browser. Connected privileged operations
  require authenticated authorization enforced by database/storage policies.

### 4. Content contracts

Every model includes `id`, `slug`, `createdAt`, `updatedAt`, `publishedAt`,
`status`, `seoTitle`, `seoDescription`, and `seoImage`. Use stable unique ids and
unique slugs within each type. `status` is one of `published`, `draft`,
`scheduled`, or `archived`; `publishedAt` and SEO fields may be null where
inapplicable, but their keys must exist. `seoImage` references a MediaAsset.
Keep order/payment/ticket workflow state in separate fields, not `status`.
Scheduled content also needs `scheduledAt`.

| Model | Required representative fields and relationships |
| --- | --- |
| Page | Title, summary, ordered ContentBlock references, template, locale. |
| BlogPost | Title, excerpt, rich-text body, Author, Category and Tag references, hero media, related posts. |
| Author | Name, biography, avatar, social links, associated posts. |
| Category | Name, description, optional parent Category; usable by posts/products/resources. |
| Tag | Name, description; many-to-many associations with content. |
| Product | Name, description, gallery, categories/tags, variants, related products, availability. |
| ProductVariant | Product reference, SKU, option values, price in integer minor units, currency, stock. |
| ProductReview | Product and reviewer references, rating, title, body, moderation state. |
| Event | Title, description, start/end, timezone, venue or online URL, speakers, capacity, registration state. |
| Resource | Title, type (guide, whitepaper, video, download), body, media/file, categories/tags, access requirements. |
| TeamMember | Name, role, biography, portrait, social links, display order. |
| Testimonial / CustomerStory | Quote, attribution, company/logo, featured flag; long-form story body and related products where applicable. |
| FAQ | Question, rich-text answer, category, display order. |
| NavigationMenu | Location, ordered nested items, internal content references or external links. |
| MediaAsset | Local/storage path, MIME type, alt text, caption, size, image dimensions, optional focal point. |
| ContentBlock | Discriminated block type, validated payload, reusable references, ordered children where applicable. |
| Notification | Recipient, title, body, type, target link, read timestamp. |
| CartItem | Owner/session, ProductVariant reference, quantity; totals derived from current prices. |
| Order | Owner, item snapshots, currency, subtotal/tax/shipping/total, addresses, order/payment states. |
| SupportTicket | Requester, subject, category, priority, messages, attachments, ticket state. |
| JobApplication | Applicant, job reference, contact details, resume MediaAsset, cover letter, consent, application state. |

Define supporting contracts for users/profiles, settings, jobs, event registrations,
newsletter subscriptions, contact submissions, surveys/responses, feedback, and
wishlist entries. Persist submissions through the same data abstraction.

Cover strings, multiline text, rich text, numbers, booleans, enums, dates,
URLs, JSON objects, arrays, media/files, localized fields, optional/null fields,
unique values, self-references, and one-to-one/one-to-many/many-to-many links.
Use a platform-neutral structured rich-text representation with headings, marks,
lists, links, quotes, code, images, and embedded content references. Preserve
structure when editing; render safely without executing arbitrary HTML.
Define explicit publication filtering: public views show published content whose
publication time has arrived; admin views expose all states.

### 5. Route and page coverage

Implement these 23 route patterns. Parameterized routes must resolve real seeded
records and show a useful not-found state for unknown ids/slugs. Preserve search,
filter, sort, and pagination state in query parameters where applicable.

| Route | Required content and behavior |
| --- | --- |
| `/` | Block-driven homepage, featured posts/products/events, testimonials, newsletter. |
| `/about` | Team, customer stories, rich content, FAQ accordion. |
| `/blog` | Searchable/filterable post listing with pagination. |
| `/blog/:slug` | Rich article, author, taxonomy, embedded media, related posts. |
| `/authors/:slug` | Author biography and associated published posts. |
| `/categories/:slug` | Category description and categorized content links. |
| `/products` | Product grid, search, category/price filters, sort, wishlist actions. |
| `/products/:slug` | Gallery, variant selection, stock/price, reviews, add to cart. |
| `/cart` | Editable quantities, removal, totals, empty state. |
| `/checkout` | Address/contact validation, order summary, simulated order confirmation. |
| `/wishlist` | Persisted saved products with removal and cart actions. |
| `/events` | Upcoming/past event listing and filters. |
| `/events/:slug` | Schedule, speakers, location, validated registration form. |
| `/resources` | Filterable guides, downloads, and videos. |
| `/resources/:slug` | Resource detail, rich body, local file/media access. |
| `/careers` | Job listings, job detail selection, job application form. |
| `/contact` | Contact form, support information, map/location placeholder without remote dependency. |
| `/dashboard` | Activity, notifications, metrics/charts, orders, support tickets, survey. |
| `/profile` | Editable user profile and avatar with validation. |
| `/settings` | Theme, preferences, notification settings with validated save. |
| `/design-system` | Interactive catalog of primitives, variants, states, forms, overlays, and blocks. |
| `/admin` | Content tables by type, search/filters, CRUD entry points, publication controls, media-library tab. |
| `/admin/content/:id` | Typed editor for existing content; reserve `new` for creation with a type selector. |

Use shared public, dashboard, and admin layouts. Add a catch-all not-found view.
Mock mode provides an explicit demo identity for private views; it is not real
authentication. Connected mode requires authentication for private routes and
an authorized admin role for CMS views. Never treat route guards as authorization.

### 6. Forms

Use RHF + Zod for contact, newsletter, event registration, job application,
support ticket, user profile, settings, product review, survey, feedback widget,
and checkout/editor forms. Put the feedback widget in the shared layout.

Validate required fields, email/URL formats, date ranges, ratings, quantities,
file type/size, and explicit consent where applicable. Mirror important checks
at the persistence boundary. Provide accessible labels, field-level errors,
pending/success/failure states, and duplicate-submit protection. Save through
repository mutations and invalidate affected queries. Mock submissions must be
inspectable after refresh. Use synthetic applicant/customer details in fixtures.

### 7. Reusable UI and feature areas

- **Design system:** typography, spacing, colors, light/dark themes, buttons,
  inputs, selects, checkboxes, radios, switches, badges, cards, tables, tabs,
  accordions, skeletons, pagination, and accessible validation states.
- **Content blocks:** hero, text, image/gallery, video, feature grid, statistics,
  CTA, quote/testimonial, FAQ, and referenced-content lists; compose and reorder
  blocks without flattening relationships.
- **Dashboard:** metric cards, activity feed, notifications, tables, date filters,
  and Recharts line/bar/pie charts backed by repository data.
- **Commerce:** variants, inventory, cart, wishlist, derived totals, checkout,
  and stored orders. Simulate payment; do not collect card data or require a
  payment provider. Expose a typed checkout command in both repository adapters.
  Validate quantities and stock in both modes. In connected mode, the command
  must call an authenticated, secured database function delivered with the SQL
  migration; it re-reads and locks variant prices/stock, derives order totals,
  decrements stock, and writes the order atomically. Never trust client totals;
  test insufficient-stock and concurrent checkout failures in both modes.
- **CMS:** typed fields, rich-text/block editing, reference pickers, draft save,
  preview, schedule/publish/archive, delete confirmation, and validation.
  Preview must be distinct from public publication visibility.
- **Media library:** search, type filters, preview, upload, metadata/alt-text
  editing, reference selection, and safe handling of referenced-asset deletion.
  Mock uploads remain local; connected uploads obey storage authorization.
- **Navigation/layout:** responsive header, nested navigation, breadcrumbs,
  footer, dashboard/admin sidebar, and mobile navigation.
- **Interaction patterns:** toasts, dialogs/modals, drawers, popovers, tooltips,
  keyboard navigation, focus management, and destructive-action confirmation.

### 8. Mock data and migration fixtures

Ship deterministic, realistic, interconnected fixtures for every model and
supporting form record. Include enough records to demonstrate pagination,
filters, charts, taxonomy, variants, reviews, registrations, and orders.
No primary page may rely on an empty placeholder dataset.

Include published, draft, scheduled, and archived examples; past/future events;
in-stock/out-of-stock variants; optional/null fields; Unicode/multilingual text;
long titles/bodies; nested blocks; embedded references; and local image/document
assets. Keep seeded relationships valid and schedules deterministic. Include
separate controlled invalid-input cases in tests, not broken baseline fixtures.

Use the same canonical fixtures for mock mode and an optional Supabase seed,
preserving ids and relationships. Provide an explicit reset action and document
how to seed/reset connected test data without touching unrelated databases.
The installed build must remain useful with zero database and no network access.

### 9. Supabase migration

Deliver a versioned SQL migration under `supabase/migrations/` creating all model
tables, supporting submission/user tables, and relationship/join tables.

- Match TypeScript contracts, nullable metadata, JSON/block representation,
  currency/quantity constraints, slug/SKU uniqueness, and workflow enums.
- Use foreign keys and documented cascade/restrict behavior. Preserve ordered
  relationships and guard against dangling references.
- Include full-text search `tsvector` columns for searchable content, maintained
  on writes, with GIN indexes; index slugs, foreign keys, publication filters,
  and common listing/sort fields.
- Include `created_at` defaults and `updated_at` triggers on mutable tables.
- Enable RLS on all exposed tables. Public reads are limited to published,
  eligible content. Users can access only their own private records; content
  writes require an admin role that users cannot grant themselves.
- Define narrowly scoped submission policies: anonymous forms may submit only
  allowed records and may not read other people's submissions. Prevent client
  edits to protected ownership, role, price, and payment-state fields.
- Apply equivalent storage policies for public assets and private uploads such
  as resumes/support attachments; private files must not have public URLs.
- Document environment setup, authentication/admin provisioning, migration/seed
  execution, and policy verification. No credentials belong in source control.

### 10. Acceptance criteria

- [ ] After dependency installation/build, DXS runs with no Supabase configuration
  and no runtime network access; all required assets load locally.
- [ ] All 23 route patterns render meaningful seeded content; unknown routes and
  unknown records render not-found views without uncaught errors.
- [ ] Both adapters pass the same data-contract tests for CRUD, references,
  publication filtering, search, pagination, and mutation error handling.
- [ ] All listed forms reject invalid input, accept valid input, persist results,
  and provide accessible pending/success/error feedback.
- [ ] Cart, wishlist, theme, mock edits, and submissions survive refresh; fixture
  reset restores the deterministic baseline.
- [ ] Commerce supports variant/stock checks and completes a simulated order
  without collecting payment credentials.
- [ ] Dashboard charts/widgets use actual repository data; filters and actions
  work rather than displaying decorative controls.
- [ ] Admin/editor/media views support validated CRUD, reference selection,
  preview, publication states, uploads, and safe deletion in both modes.
- [ ] Connected private/admin data is protected by tested RLS/storage policies;
  public views never reveal drafts or private submissions.
- [ ] SQL migrations apply to a clean Supabase database and match every content
  contract, relationship, search index, and timestamp trigger.
- [ ] Migration verification compares fixture counts, field values, references,
  rich-text structure, asset metadata, and publication state across adapters,
  including representative rendered detail pages.
- [ ] Layouts work at mobile and desktop widths, support keyboard interaction,
  and expose consistent loading, empty, error, and validation states.
- [ ] Repository scripts provide reproducible lint/typecheck/build/test commands;
  setup documentation explains offline mode, connected mode, fixtures, and tests.

### 11. Implementation sequence

1. Scaffold the released stack and routing/layout/design-system foundations.
2. Define models, validation schemas, repository contracts, and deterministic
   fixtures; implement mock persistence before connecting a database.
3. Build all public routes, forms, commerce, and dashboard against the abstraction.
4. Implement CMS editing, publication workflows, media library, and UI catalog.
5. Add Supabase migrations, adapter, authentication, policies, and optional seed.
6. Validate acceptance criteria with focused unit/contract tests and route/form
   smoke tests, then exercise the complete offline and connected workflows.

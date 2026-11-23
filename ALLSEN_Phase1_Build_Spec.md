# ALLSEN Valve Care: Phase 1 Build Specification

You are building Phase 1 of "ALLSEN Valve Care", a web application that gives every valve ALLSEN sells a unique QR code, a digital record and a public scan page. Read this whole document before writing code. Where something is unclear, ask me before guessing. Where this document says "decide", pick the simplest option that fits and tell me what you chose.

---

## 1. Business context

- ALLSEN sells industrial valves (butterfly, ball and others) made by outside manufacturers.
- The manufacturer sends ALLSEN an Excel file of production data: one row per valve, with the **manufacturer serial number** plus other fields (type, size, pressure class, materials and so on). **The number and names of columns vary between files.**
- **ALLSEN owns the nameplate.** The app must produce an Excel file (one row per valve) containing everything to be imprinted on the nameplate, plus the QR link for that valve, so the person imprinting it can do it correctly without retyping anything.
- Valves are later allocated to customers through orders. Allocated valves count as "sold".
- Anyone who scans a valve's QR code sees a public page with valve details and documents. **No login is needed for scanning.**
- Customers do **not** log in during Phase 1. Employees do.

## 2. Phase 1 scope

### In scope
1. Employee and admin login, roles and permissions, audit log
2. Batches: manufacturer Excel upload, flexible column mapping, validation, valve creation, ALLSEN ID and QR token generation
3. Nameplate export: imprinting Excel with embedded QR images, plus a ZIP of QR image files
4. Documents: upload PDFs per batch, per selected valves, or per valve, with public or internal visibility
5. Customers: create and edit customers, sites and contacts
6. Orders: create order, select unallocated valves (filters, pasted serial list, Excel upload), map to customer
7. Valve register: search and view any valve and its full record
8. Public QR scan page (no login) and a simple service enquiry form on it
9. Dashboard (stock, sold, activity) and admin functions (users, roles, master data, nameplate templates, corrections, audit log)

### Out of scope (do not build, but do not block)
- Customer login or customer dashboard
- Maintenance schedules, reminders, service requests, quotations, service jobs, technician login
- Payments, invoices, SAP or ERP integration, WhatsApp or SMS

Design the database so Phase 2 can be added without reworking Phase 1 tables.

## 3. Tech stack (fixed)

- **Next.js** (App Router) with **TypeScript**
- **Tailwind CSS** and **shadcn/ui** for the interface
- **Supabase**: Postgres, Auth (email and password), Storage
- **Vercel** for deployment
- Libraries: `exceljs` (read and write Excel, embed images), `qrcode` (QR images), `jszip` (ZIP), `zod` (validation), `react-hook-form`, `@supabase/ssr` and `@supabase/supabase-js`, `papaparse` only if CSV support is needed
- Use Supabase migrations (`supabase/migrations/*.sql`) for every schema change. No manual dashboard edits to the schema.

### Environment variables
```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=      # server only, never sent to the browser
NEXT_PUBLIC_APP_URL=            # e.g. https://valves.allsen.com, used inside QR links
```
Read these from `.env.local` locally and Vercel project environment variables in production. Never commit secrets. Provide `.env.example`.

## 4. Roles and permissions

Roles live in `profiles.role`. Enforce permissions both in the UI and in the database (RLS and server-side checks).

| Role | Can do |
|---|---|
| `admin` | Everything, including users, roles, master data, templates, corrections, audit log |
| `data_executive` | Batches (upload, map, export nameplate Excel), documents, view valve register |
| `order_executive` | Customers, orders, allocation, view valve register |
| `viewer` | Read-only on valve register, customers, orders |

Rules:
- Employees are created **by an admin** (invite by email). There is no public sign-up.
- Deactivate users (`is_active = false`), never delete them.
- A deactivated user cannot log in or call any API.

## 5. Data model

Use UUID primary keys, `created_at` and `updated_at` timestamps, and `created_by` where useful. Below is the intended structure; write the actual SQL migration, with indexes, foreign keys and constraints.

### Enums
- `user_role`: admin, data_executive, order_executive, viewer
- `valve_status`: received, in_stock, allocated, dispatched, in_service
- `doc_visibility`: public, internal

### Tables

**profiles**: `id` (= auth.users.id), `full_name`, `email`, `phone`, `role`, `is_active`

**manufacturers**: `id`, `name`, `notes`, `is_active`

**import_mappings**: saved column mappings per manufacturer: `id`, `manufacturer_id`, `name`, `mapping jsonb` (source column header to target field), `created_by`

**upload_batches**: `id`, `batch_code` (human readable, unique, for example `B-2026-0007`), `manufacturer_id`, `original_filename`, `source_file_path` (Storage), `mapping jsonb`, `total_rows`, `valid_rows`, `status` (draft, committed, cancelled), `notes`, `created_by`

**valves**
- `id`
- `allsen_id` text unique (generated, format `ALS-` plus 6 digits from a Postgres sequence, for example `ALS-000001`)
- `qr_token` text unique (random, unguessable, at least 22 URL-safe characters; **never derived from the ALLSEN ID or serial**)
- `batch_id`, `manufacturer_id`
- `manufacturer_serial` text; **unique per manufacturer** (`unique(manufacturer_id, manufacturer_serial)`)
- Standard promoted columns: `valve_type`, `size`, `pressure_class`, `body_material`, `end_connection`, `year_of_manufacture`
- `attributes jsonb`: **every other column from the manufacturer's Excel**, as key and value pairs, so any number of extra columns works
- `status valve_status` default `in_stock` after the batch is committed
- `customer_id` null, `order_id` null, `allocated_at`, `allocated_by`
- `public_page_active boolean` default true
- Index the promoted columns, `batch_id`, `customer_id`, `order_id`, `status`, and `attributes` (GIN)

**customers**: `id`, `name`, `gst_number`, `billing_address`, `city`, `state`, `notes`, `is_active`
**customer_sites**: `id`, `customer_id`, `name`, `address`, `city`
**customer_contacts**: `id`, `customer_id`, `site_id` null, `name`, `email`, `phone`, `designation`

**orders**: `id`, `order_no` (unique, human readable, for example `SC-118`, editable on creation), `customer_id`, `site_id` null, `po_number`, `po_date`, `status` (draft, confirmed, dispatched, cancelled), `notes`, `created_by`
**order_lines**: `id`, `order_id`, `valve_type`, `size`, `quantity` (expected quantities, used only for checking)

**documents**: `id`, `title`, `doc_type` (test_certificate, material_certificate, manual, datasheet, other), `certificate_no` null, `issue_date` null, `file_path` (Storage), `file_name`, `file_size`, `visibility doc_visibility` default internal, `version` int, `is_current` bool, `uploaded_by`
**document_links**: `id`, `document_id`, `batch_id` null, `valve_id` null (exactly one of the two is set; a batch link applies to all valves of that batch)

**nameplate_templates**: `id`, `name`, `valve_type` (or null for default), `fields jsonb` (ordered list of `{key, label, source}` describing exactly which columns appear in the imprinting Excel and in what order), `is_active`
**nameplate_exports**: `id`, `batch_id`, `template_id`, `revision` int, `excel_path`, `qr_zip_path`, `row_count`, `created_by`, `created_at`

**field_dictionary**: master list of standard field names, aliases and allowed values (for example `Size` with aliases `Valve Sz`, `Dia`), used by the column mapping screen
**service_enquiries**: `id`, `valve_id`, `name`, `company`, `phone`, `email`, `message`, `status` (new, handled), `created_at`

**audit_log**: `id`, `actor_id`, `action`, `entity_type`, `entity_id`, `details jsonb`, `created_at`. Write an entry for: login events if available, user changes, batch upload and commit, export, document upload, replace or hide, customer changes, order creation, allocation and un-allocation, valve edits, public page deactivation.

### Security (Row Level Security)
- Enable RLS on **every table**.
- Authenticated, active employees may read according to their role; writes are limited by role. Implement helper SQL functions such as `current_role()` and `is_active_employee()`.
- The `anon` role must have **no direct access to any table or the Storage bucket**.
- Public scan data is served only through one **security definer function** (for example `get_public_valve(token text)`) that returns only the whitelisted public fields (see section 8), or through a server route using the service role that applies the same whitelist.
- Storage: one **private** bucket for documents and source files. Public documents are delivered through short-lived signed URLs created server-side only when `visibility = public` and the valve's public page is active.
- The service role key is used only in server code (route handlers and server actions), never in client components.

## 6. Features in detail

### 6.1 Authentication and users
- Email and password login via Supabase Auth. Middleware protects all `/app` routes. Show a clear message for deactivated users.
- Admin screen: create employee (name, email, phone, role) which sends an invite; edit role; deactivate and reactivate; send password reset.

### 6.2 Batches (manufacturer Excel upload)
Flow, in this order:
1. **Upload** `.xlsx` or `.csv` (size limit 10 MB). Store the original file in Storage. Choose the manufacturer.
2. **Parse** the first sheet; read the header row; show the first 10 rows.
3. **Column mapping screen:** every source column is shown with a suggested target: standard fields (manufacturer serial, valve type, size, pressure class, body material, end connection, year of manufacture) or "Keep as extra field" (stored in `attributes` under the original header) or "Ignore". Suggest matches using `field_dictionary` aliases and the manufacturer's saved mapping. **Manufacturer serial must be mapped.** Allow "Save this mapping for this manufacturer".
4. **Validation preview:** show counts and a table of problems: missing serial, duplicate serial within the file, serial that already exists for this manufacturer, empty required nameplate fields (based on the active template). Allow downloading an error report as Excel. The user may fix the Excel and re-upload, or exclude the failing rows. Nothing is saved to `valves` until step 5.
5. **Commit:** in one database transaction, create the batch and the valves (`status = in_stock`), assigning ALLSEN IDs from the sequence and generating `qr_token`. Show a success summary. Allow **cancel batch** (admin only), which is blocked once any valve is allocated.
6. **Batch page** shows: batch details, valves table (search, filter, pagination), documents attached, and the nameplate export history.

Performance: files of up to 5,000 rows must import within a reasonable time; use batched inserts.

### 6.3 Nameplate export (part of the Batch page)
Button: **Export for imprinting**.
- Choose the template (default by valve type). If a batch contains several valve types, create one sheet per template, or apply each valve's template, and tell me which approach you picked.
- Generate an `.xlsx` with one row per valve and these columns: row number, ALLSEN ID, manufacturer serial, every field in the template **in template order, as final text exactly as it should be imprinted**, **QR URL** (`{NEXT_PUBLIC_APP_URL}/v/{qr_token}`), and the **embedded QR image** in an adjacent cell (use row heights and widths that make the QR scannable).
- Also generate a ZIP with one PNG per valve named `{allsen_id}.png` (and SVG if easy), with enough error correction (level M or higher) and a quiet zone.
- Each export creates a new `revision` row in `nameplate_exports`, saves the files in Storage, and can be re-downloaded later. Show who exported and when.
- If required template fields are empty for any valve, warn before exporting and list those valves.

### 6.4 Documents
- Upload PDFs (limit 20 MB each; reject other types; check the actual file signature, not only the extension).
- Choose scope: **entire batch**, **selected valves** (pick from the batch's valves table), or **single valve**.
- **Bulk matching:** upload many PDFs at once where each file name is a manufacturer serial or ALLSEN ID (for example `MFG-101.pdf`). Show a preview of matches and unmatched files before saving.
- Set `doc_type`, `title`, optional certificate number and issue date, and `visibility` (public or internal; **default internal**).
- Replacing a document creates a new `version`; older versions are kept but hidden from the public page.
- Allow deleting only a document that was uploaded by mistake (admin only, logged); otherwise "hide".

### 6.5 Customers
- Create and edit customers with sites and contacts. Validate the GST number format if provided.
- Customer page shows details, orders, and valves allocated (table with filters).

### 6.6 Orders and allocation
- **Create order:** order number (suggest the next one, editable and unique), customer (picklist with search), site (optional), PO number and date, optional lines (type, size, quantity).
- **Select valves** (valves with `status = in_stock` and no customer only), by any combination of:
  1. **Filters:** batch, type, size, pressure class, with row ticking and "select all matching"
  2. **Paste serial list:** a text box accepting serials separated by new lines, commas or spaces; case-insensitive and trimmed; match against the manufacturer serial **or** the ALLSEN ID. Show a preview with a result per entry: found and available, not found, already allocated (show to which order and customer), duplicate in list, and type or size mismatch with the order lines. If the same manufacturer serial exists for more than one manufacturer, ask which one. Nothing is saved until the user confirms.
  3. **Upload an Excel or CSV of serials** with the same preview logic
- On **Save order**, in one transaction: set `customer_id`, `order_id`, `status = allocated`, `allocated_at` and `allocated_by` on all selected valves. Concurrency rule: if any valve was allocated by someone else in the meantime, fail the whole save with a clear message listing those valves.
- Order status: draft, confirmed, dispatched (when dispatched, valves move to `dispatched`), cancelled.
- **Remove or reassign valves:** admin only, requires a reason, logged in `audit_log`. Valves returned to stock clear `customer_id` and `order_id`.
- Order page shows lines versus allocated counts, the list of valves, and an Excel export of the order's valve list.

### 6.7 Valve register
- One searchable table of all valves. Search by ALLSEN ID or manufacturer serial; filter by status, batch, customer, order, type, size.
- **Valve detail page:** identity, all promoted fields, all `attributes`, status, batch, customer and site, order, documents (with visibility), QR image and public link, and the audit history for that valve.
- Admin only: edit valve fields (logged with before and after), deactivate the public page, regenerate the QR token (with a warning that printed QR codes will stop working).

### 6.8 Dashboard
Cards and lists: valves by status (received, in stock, allocated, dispatched), stock by type and size, batches uploaded this month, recent orders, valves in stock for more than 90 days, batches without documents, and recent activity from `audit_log`.

### 6.9 Admin functions
- Users and roles (section 6.1)
- **Master data:** manufacturers, valve types, sizes, pressure classes, materials, document types, `field_dictionary` (aliases)
- **Nameplate templates:** create and edit the ordered list of fields per valve type
- **Corrections:** reassign or remove valves from orders, cancel a batch, deactivate or reactivate public pages
- **Audit log** viewer with filters by user, entity, date; export to Excel

## 7. Pages and routes

Employee area (login required, role-aware navigation):
`/app` dashboard · `/app/batches` list · `/app/batches/new` upload wizard · `/app/batches/[id]` · `/app/documents` · `/app/customers` · `/app/customers/[id]` · `/app/orders` · `/app/orders/new` · `/app/orders/[id]` · `/app/valves` · `/app/valves/[id]` · `/app/admin/users` · `/app/admin/master-data` · `/app/admin/templates` · `/app/admin/audit`

Public:
`/v/[token]` public valve page · `/login`

Keep the interface clean, responsive and usable on a laptop first, with mobile layouts for the public page.

## 8. Public QR page (`/v/[token]`)

- Server-rendered, mobile-first, no login, `noindex` meta tag, no navigation to other valves.
- If the token is unknown or the valve's `public_page_active = false`, show a neutral "This valve record is not available. Contact ALLSEN." message with the contact details. Do not reveal whether the token ever existed.
- **Show only:** ALLSEN logo, ALLSEN ID, manufacturer serial, valve type, size, pressure class, body material, end connection, year of manufacture, other `attributes` flagged as public in the field dictionary (add an `is_public` flag), and the list of **public, current** documents with download links through short-lived signed URLs.
- **Never show:** customer, site, order, PO, batch internals, status, internal documents, audit data, anything about service or prices.
- Include a **Request service / contact ALLSEN** button that opens a simple form (name, company, phone, email, message; valve ID prefilled). Protect it with rate limiting and a simple spam check (honeypot field or CAPTCHA). Save to `service_enquiries` and show employees the new enquiries on the dashboard.
- Rate-limit the page to slow down scraping.

## 9. Non-functional requirements

- **Security:** RLS everywhere, server-side permission checks in every server action and route handler, input validation with `zod`, parameterised queries only, file type checks, CSRF-safe patterns, no secrets in client code.
- **Auditability:** every create, update, allocation and deletion by an employee writes to `audit_log`.
- **Data integrity:** transactions for batch commit and order allocation; unique constraints for serials; soft deletion or deactivation instead of hard deletes.
- **Quality:** TypeScript strict mode, ESLint, a README with setup and deployment steps, seed script with sample data (2 manufacturers, 1 batch of 20 valves, 2 customers, 1 order).
- **Tests:** unit tests for the Excel parsing and column mapping, ALLSEN ID generation, serial paste matching, and allocation concurrency; one end-to-end test of the full flow (upload batch, export, create customer, create order, allocate, scan public page).
- **Time zone and formats:** Asia/Kolkata, dates shown as DD MMM YYYY.

## 10. Build order

Work in these stages. Finish and show me each stage before starting the next.

1. Project setup, Supabase migrations (all tables, enums, RLS, sequence, helper functions), auth, roles, layout, user management
2. Master data, field dictionary, nameplate templates
3. Batches: upload, mapping, validation, commit, batch page
4. Nameplate export (Excel with QR images, ZIP)
5. Documents (scopes, bulk match, versions)
6. Customers and orders, allocation (filters, pasted list, upload), valve register and valve page
7. Public QR page and service enquiry form
8. Dashboard, audit log viewer, admin corrections
9. Tests, seed data, README, Vercel deployment checklist

## 11. Acceptance checklist (Phase 1 is done when)

- [ ] An admin can create an employee account and the employee can log in with their role's permissions only.
- [ ] A manufacturer Excel with an unfamiliar set of extra columns imports successfully; all extra columns are stored and visible on the valve page.
- [ ] Duplicate or missing serials are caught in the preview, and nothing is saved until commit.
- [ ] Each valve gets a unique ALLSEN ID and an unguessable QR token.
- [ ] The nameplate Excel opens in Excel with one row per valve, the fields in template order, the QR URL, and a scannable embedded QR image; the ZIP of QR images is correct.
- [ ] Documents can be attached to a whole batch, selected valves, or one valve, and 40 PDFs named by serial can be matched in one upload.
- [ ] An employee can create a customer and an order, select unallocated valves by filters, by pasted serial list, and by Excel upload, preview problems, and save; all selected valves then show that customer and order.
- [ ] Two employees cannot allocate the same valve.
- [ ] Searching a serial or ALLSEN ID shows the valve's customer, order, batch, documents and history.
- [ ] Scanning a QR code on a phone opens the public page without login, with only the whitelisted fields and public documents; customer and order details are never visible.
- [ ] A deactivated valve page shows the neutral message.
- [ ] Unauthenticated requests cannot read any table or private file (verify with the anon key).
- [ ] The app is deployed on Vercel with environment variables configured, and works end to end.

## 12. Questions to ask me before you start

1. The exact column list and wording for the nameplate (per valve type), and any mandatory markings.
2. The domain name to use in QR links (this must stay stable for the life of the valves).
3. Sample manufacturer Excel files, so you can test column mapping.
4. Whether certificates should be public by default.
5. Branding: logo, colours, and contact details for the public page.

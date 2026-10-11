Engine Component Procurement System

SAP-inspired Procure-to-Pay (P2P) app for engine components. Static HTML/CSS/JavaScript talks to Supabase Auth, PostgreSQL, and Realtime. No Node/Express server. After Vercel deploy, opening the URL is enough.

Architecture

Browser (HTML + CSS + JS)
        |
        |  HTTPS  (anon key, RLS)
        v
Supabase
  - Auth          two demo users (Buyer, Vendor)
  - PostgreSQL    components, PR, PO, GR, invoice, payment, history
  - Realtime      live Buyer <-> Vendor updates





role.html stores Buyer or Vendor in sessionStorage, then opens the matching login page.



Login uses Supabase Auth. Profile role must match the chosen login. Wrong role is signed out.



buyerdb.html / vendordb.html are one dashboard each with tabs (not a page per process).



Writes go to Supabase. The other role sees changes through Realtime subscriptions.



KPIs and the chatbot read the same live tables in the browser.

Deploy is static files only. Vercel does not run a backend.

File structure

role.html           role selection
buyerlogin.html     buyer auth
vendorlogin.html    vendor auth
buyerdb.html        buyer dashboard (tabs)
vendordb.html       vendor dashboard (tabs)
style.css
app.js
supabase.js         project URL + anon key
schema.sql          run once in Supabase
index.html          redirects to role.html
vercel.json
README.md



Database

See schema.sql. Relationships:





profiles.id = auth.users.id (buyer | vendor)



components.vendor_id → profiles



purchase_requisitions → buyer, vendor, component



purchase_orders.pr_id → one PO per PR



goods_receipts.po_id → one GR per PO



invoices.po_id → one invoice per PO



payments.invoice_id → one payment per invoice



transaction_history append-only audit

Editable: component stock, price, delivery status.
Not editable in UI: PR/PO/invoice IDs, created timestamps, posted documents.

Setup

Run locally over HTTP instead of opening the HTML file directly:

```text
npm start
```

Then open `http://localhost:3000/` in the browser. Opening `role.html` from a `file://` URL causes browser security errors and can prevent the Supabase CDN client from loading correctly.





Create a Supabase project.



Authentication → Providers → Email: turn off Confirm email.



SQL Editor: run schema.sql.



Authentication → Users → Add user:





buyer@demo.com / Buyer@12345



vendor@demo.com / Vendor@12345



SQL Editor:

insert into public.profiles (id, role, display_name)
select id, 'buyer', 'Buyer' from auth.users where email = 'buyer@demo.com'
on conflict (id) do nothing;

insert into public.profiles (id, role, display_name)
select id, 'vendor', 'Vendor' from auth.users where email = 'vendor@demo.com'
on conflict (id) do nothing;

insert into public.components (vendor_id, name, stock, price)
select p.id, x.name, x.stock, x.price
from public.profiles p
cross join (values
  ('Connecting Rod', 40, 12500),
  ('Crankshaft', 12, 48000),
  ('Piston', 80, 6200),
  ('Engine Bearing', 120, 1850)
) as x(name, stock, price)
where p.role = 'vendor'
  and not exists (
    select 1 from public.components c
    where c.vendor_id = p.id and c.name = x.name
  );





Project Settings → API: copy URL and anon key into supabase.js.



Open role.html locally, or deploy the folder to Vercel.

Demo in two browsers: Buyer in one, Vendor in the other.

P2P path

Vendor stock/price → Buyer marketplace → PR → Vendor confirm/reject → PO → ship/deliver → GR → invoice → three-way match → payment.

Match: PO qty & price = GR qty = invoice qty & price → Ready for payment. Else payment on hold.

Vercel

Connect the Git repo. Root directory = project folder. Framework: Other. Output: none (static). Root URL opens role.html.
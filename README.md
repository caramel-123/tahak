# Tahak 🌿

> **Discover Trusted Local Guides, Secured by Stellar Escrow**
> Giving Filipino tour guides a verifiable, on-chain reputation — and giving travelers a safe way to pay for it.

🌐 **Live App:** [https://tahak-ecru.vercel.app](https://tahak-ecru.vercel.app)
📦 **Repo:** [github.com/caramel-123/tahak](https://github.com/caramel-123/tahak)

---

## Project Description

Tahak (Tagalog for "journey" or "traverse") is a tour guide discovery, booking, and reputation platform for the Philippines, built on **React + Supabase + Stellar**. Travelers browse DOT-verified local guides by destination, language, and specialty, book a tour, and pay through a Stellar wallet (Freighter) — with the long-term goal of holding payment in on-chain escrow that releases only as milestones (QR check-ins) are confirmed.

Tahak avoids the fintech/crypto-exchange aesthetic on purpose. It's meant to feel like **Airbnb + AllTrails + Google Travel**, with the blockchain doing quiet work in the background rather than being the headline.

---

## Project Vision

Tour guiding in the Philippines runs almost entirely on word of mouth and unverifiable claims — a guide's "500 happy tourists" is just a sentence on a Facebook page. Meanwhile travelers have no recourse if a tour is cancelled, unsafe, or never happened at all, and guides have no way to prove a reputation earned over years of good work.

Tahak's vision is to make that reputation **portable and verifiable**: ratings, completed tours, and DOT/barangay accreditation live as real data a lender, tourist, or tourism officer can check — not a claim on a flyer. Payment protection follows the same logic — funds held in escrow and released against verified milestones, so neither side has to trust the other blindly.

---

## Features

### For Tourists
- 🔎 **Explore Guides** — search and filter by destination, language, price, rating, and specialty
- 🪪 **Verification badges** — DOT Accredited, Barangay Certified, Community Vouched, Top Rated
- 👤 **Guide profiles** — bio, languages, specialties, tours, real reviews
- 📅 **Bookings dashboard** — Upcoming / Ongoing / Completed, with milestone progress bars
- 📷 **QR Check-In** flow for confirming tour milestones
- 🔐 **Connect Freighter wallet** — real Stellar testnet account connection, no username/password

### For Guides
- 📊 **Guide dashboard** — revenue, escrow-waiting amount, upcoming tours, reputation
- 🗂️ **Manage tours** — create, edit, pricing, availability, gallery
- 📥 **Booking requests** — accept/reject, track milestones, view group members

### For Tourism Officers
- 🗺️ **Regional dashboard** — verified guide counts, open cases, safety alerts
- ⚖️ **Dispute resolution** — review booking timeline, evidence, statements, decide refund / release / split

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 18 + TypeScript + Vite |
| Styling | Tailwind CSS v4 + shadcn/ui |
| Backend | Supabase (Postgres + REST + Row Level Security) |
| Wallet | Freighter ([@stellar/freighter-api](https://www.npmjs.com/package/@stellar/freighter-api)) on Stellar Testnet |
| Balance lookup | Stellar Horizon Testnet API |
| PWA | Web manifest + service worker (installable, offline shell caching) |
| Deployment | Vercel |

---

## Wallet Integration

Tahak connects to a **real Freighter wallet** on Stellar Testnet:

1. Detects whether the Freighter browser extension is installed
2. Calls `requestAccess()` so the user approves the connection in Freighter itself
3. Reads the active network via `getNetwork()`
4. Fetches the real XLM balance from `horizon-testnet.stellar.org`

> **Current scope:** wallet connection and balance reads are real. Escrow — holding a booking's payment on-chain and releasing it against QR-confirmed milestones — is designed in the UI (see Booking and QR Check-In screens) but not yet backed by a deployed Soroban contract. That's the next real build, not a shipped feature — see [Future Scope](#future-scope).

---

## Database Schema (Supabase)

```sql
create table profiles (
  id uuid primary key default gen_random_uuid(),
  wallet_address text unique,
  role text not null check (role in ('tourist', 'guide', 'officer')) default 'tourist',
  full_name text,
  avatar_url text,
  created_at timestamptz not null default now()
);

create table destinations (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  region text not null,
  image_url text,
  guide_count int not null default 0
);

create table guides (
  id uuid primary key default gen_random_uuid(),
  profile_id uuid references profiles(id) on delete set null,
  name text not null,
  location text not null,
  specialty text not null,
  bio text,
  avatar_url text,
  cover_url text,
  rating numeric(3,2) not null default 0,
  review_count int not null default 0,
  tours_completed int not null default 0,
  price_per_day numeric(10,2) not null default 0,
  languages text[] not null default '{}',
  specialties text[] not null default '{}',
  badges text[] not null default '{}',
  verified boolean not null default false,
  created_at timestamptz not null default now()
);

create table tours (
  id uuid primary key default gen_random_uuid(),
  guide_id uuid not null references guides(id) on delete cascade,
  destination_id uuid references destinations(id) on delete set null,
  title text not null,
  duration text,
  group_size text,
  price numeric(10,2) not null default 0,
  included text[] not null default '{}',
  images text[] not null default '{}',
  created_at timestamptz not null default now()
);

create table bookings (
  id uuid primary key default gen_random_uuid(),
  code text unique not null,
  tourist_id uuid references profiles(id) on delete set null,
  guide_id uuid references guides(id) on delete set null,
  tour_id uuid references tours(id) on delete set null,
  destination text not null,
  booking_date date,
  status text not null check (status in ('upcoming','ongoing','completed','cancelled','disputed')) default 'upcoming',
  amount numeric(10,2) not null default 0,
  milestone text,
  progress int not null default 0,
  escrow_tx_hash text,
  created_at timestamptz not null default now()
);

create table reviews (
  id uuid primary key default gen_random_uuid(),
  booking_id uuid references bookings(id) on delete cascade,
  guide_id uuid references guides(id) on delete cascade,
  tourist_id uuid references profiles(id) on delete set null,
  rating int not null check (rating between 1 and 5),
  comment text,
  photo_url text,
  created_at timestamptz not null default now()
);

create table testimonials (
  id uuid primary key default gen_random_uuid(),
  guide_id uuid references guides(id) on delete set null,
  name text not null,
  country text,
  avatar_url text,
  rating int not null default 5,
  text text not null,
  created_at timestamptz not null default now()
);
```

Row Level Security is enabled on every table: public `SELECT` (guide/destination/booking data is meant to be browsable), and public `INSERT` on `profiles`, `bookings`, and `reviews` for the current no-auth prototype stage.

---

## Setup Instructions

### Prerequisites
- [Node.js](https://nodejs.org/) v18+
- [Freighter Wallet](https://freighter.app/) browser extension, switched to **Testnet**
- A free [Supabase](https://supabase.com) project

### 1. Clone the repository

```bash
git clone https://github.com/caramel-123/tahak.git
cd tahak
```

### 2. Install dependencies

```bash
npm install
```

### 3. Configure environment variables

```bash
cp .env.example .env.local
```

```env
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key
```

### 4. Set up the database

In your Supabase project's **SQL Editor**, run the schema from [Database Schema](#database-schema-supabase) above, then enable RLS with public `SELECT` policies on all tables (and public `INSERT` on `profiles`, `bookings`, `reviews`) so the anon key can read/write during local development.

### 5. Run locally

```bash
npm run dev
```

Open [http://localhost:5173](http://localhost:5173)

### 6. Connect a testnet wallet

1. Install [Freighter](https://freighter.app/) and switch it to **Testnet**
2. Fund your address via [Stellar Friendbot](https://friendbot.stellar.org/?addr=YOUR_ADDRESS)
3. Click **Connect Wallet** in the app and approve the Freighter popup

---

## Project Structure

```
tahak/
├── public/
│   ├── manifest.webmanifest   # PWA manifest
│   ├── sw.js                  # Service worker (cache-first for app assets)
│   └── icon.svg
├── src/
│   ├── app/
│   │   ├── App.tsx             # All pages + components (single-file prototype)
│   │   └── components/
│   │       ├── ui/             # shadcn/ui primitives
│   │       └── figma/          # Figma export helpers
│   ├── lib/
│   │   └── supabase.ts         # Supabase client + row types
│   ├── styles/                 # Tailwind + theme tokens + fonts
│   └── main.tsx
├── .env.example
└── vercel.json
```

---

## Known Limitations

Being upfront about what's real versus illustrative in the current build:

- **No on-chain escrow yet** — booking payments are recorded in Supabase (`bookings.amount`, `.status`, `.progress`), but no Soroban contract holds or releases funds. The "Secured by Stellar Escrow" copy describes the intended design, not a shipped mechanism.
- **No real auth binding** — wallet connection is real, but bookings/reviews aren't yet tied to the connected wallet via Supabase Auth; RLS currently allows public writes for demo purposes.
- **Guide/Tourism Officer dashboards use static demo data** — only the Landing, Explore, Guide Profile, and Bookings screens are wired to live Supabase queries so far.

---

## Future Scope

### Near-Term
- Deploy a Soroban escrow contract — fund on booking, release on QR-confirmed milestone completion
- Bind bookings/reviews to the connected wallet address via Supabase Auth + RLS instead of public write policies
- Wire Guide Dashboard and Tourism Officer screens to live Supabase data

### Medium-Term
- QR Check-In that actually confirms an on-chain milestone, not just a progress bar
- Dispute resolution workflow backed by real escrow state (refund / release / split as contract calls)
- DOT/barangay verification pipeline for guide accreditation badges

### Long-Term
- Reputation portable across platforms — exportable proof of a guide's completed-tour history
- Regional analytics for tourism officers pulled from real booking/dispute data
- Multi-language support for non-English-speaking travelers

---

## License

MIT © 2026 Mel Bernabe

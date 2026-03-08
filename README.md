# momolog_official

`momolog_official` is a public read-only site that lists groups managed by admin/system.
It subscribes to Firestore `/groups` in real time and reflects add/delete immediately.

## Stack

- SvelteKit
- Tailwind CSS
- Firebase Firestore (read-only)

## Setup

1. Install dependencies.
2. Copy `.env.example` to `.env` and fill Firebase public config values.
3. Run development server.

```bash
npm install
cp .env.example .env
npm run dev
```

## Environment Variables

Set these in `.env`:

- `PUBLIC_FIREBASE_API_KEY`
- `PUBLIC_FIREBASE_AUTH_DOMAIN`
- `PUBLIC_FIREBASE_PROJECT_ID`
- `PUBLIC_FIREBASE_STORAGE_BUCKET`
- `PUBLIC_FIREBASE_MESSAGING_SENDER_ID`
- `PUBLIC_FIREBASE_APP_ID`

## Responsibilities

- Read and display list from Firestore `/groups`
- Reflect add/delete in real time via `onSnapshot`
- No auth
- No edit/manage feature

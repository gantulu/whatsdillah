# Whatsdillah Frontend

React + Vite + TypeScript foundation for the Whatsdillah public WhatsApp Management application.

## Requirements

- Node.js 20.19+ or 22.12+
- npm

## Setup

```bash
npm install
cp .env.example .env
npm run dev
```

Set the public Supabase project URL and Publishable Key in `.env` before connecting application data.

## Scripts

- `npm run dev` — start the local development server
- `npm run typecheck` — run TypeScript project checks
- `npm run build` — type-check and create the production build
- `npm run preview` — preview the production build locally

## GitHub Pages

The Vite base path is configured as `/whatsdillah/`, matching this repository name. The production build is generated in `dist/`.

## Access model

The frontend does not implement login, registration, or Supabase Auth. It uses the public Supabase Publishable Key. Never put Meta credentials or a Supabase Secret Key in frontend environment variables.

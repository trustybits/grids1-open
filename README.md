# grids.so (open-source mirror)

`grids.so` is a Vue 3 + TypeScript application built with Vite. This repository is a public mirror intended for open-source collaboration and visibility.

## Tech stack

- Vue 3
- TypeScript
- Vite
- Pinia
- Firebase
- Vitest + ESLint

## Getting started

1. Install dependencies:

```sh
npm install
```

2. Copy environment variables and fill in your values:

```sh
cp .env.example .env
```

3. Start the development server:

```sh
npm run dev
```

## Scripts

- `npm run dev` - start local dev server
- `npm run build` - type-check and build for production
- `npm run preview` - preview production build locally
- `npm run lint` - run ESLint
- `npm run test` - run tests in watch mode
- `npm run test:run` - run tests once

## Environment and services

This app integrates with external services (for example Firebase, PostHog, Stripe, Mapbox, and Notion). Use `.env.example` as the canonical list of required variables.

Never commit real secrets. Keep credentials only in local `.env` files or your deployment platform secret manager.

## License

This project is licensed under the Apache License 2.0. See `LICENSE` for details.

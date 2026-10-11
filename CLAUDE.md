# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A single-page React 18 + TypeScript treasure-hunt game, exported from Figma Make and built with Vite. It is the starter project (`initial` branch of https://github.com/uopsdod/claude_code_treasure_game) for a Claude Code course; `README.md` lists the course exercises (sound effects, win/tie/loss result, key-cursor hover, SQLite sign-in, Vercel / GitHub Pages deploy commands).

## Commands

```bash
npm install
npm run dev     # Vite dev server on http://localhost:3000 (opens the browser)
npm run build   # production build to build/ (not dist/)
```

Deploys are custom commands in `.claude/commands/`: `/deploy_vercel` (`vercel.json` sets `outputDirectory: build`) and `/deploy_github_page` (builds with `npm run build -- --base=/<repo>/ --outDir build-gh-pages` and publishes with `gh-pages`). Keep `base` out of `vite.config.ts` — dev and Vercel serve from `/`. Both deployed sites are static, so only guest mode works there. `origin` should be the user's repo and `upstream` the course repo.

There is no test runner, linter, or `tsconfig.json`. Vite (SWC) strips types without type-checking, so type errors do not fail the build.

## Architecture

- **All game logic lives in `src/App.tsx`.** State: `boxes` (3 boxes; one random box has `hasTreasure`), `score`, `gameEnded`. Opening a box scores +100 for treasure and −50 for a skeleton; the game ends when the treasure is found or every box is open. `openBox` calls `setScore` inside the `setBoxes` updater and reads `score` from the closure. Side effects such as sounds run in `openBox` before `setBoxes`, not inside the updater (React may run an updater twice).
- Once `gameEnded`, a Win / Tie / Loss badge (score > 0 / = 0 / < 0) appears next to the score. Closed boxes show `src/assets/key.png` as the cursor through an inline `style` (see Tailwind gotcha below).
- Animations use `motion/react` (Framer Motion's successor). Images and audio are imported as ES modules from `src/assets/` and `src/audios/`; opening a box plays `chest_open.mp3` (treasure) or `chest_open_with_evil_laugh.mp3` (skeleton) with `new Audio(...)`.
- **Accounts and scores (SQLite).** `server/api.mjs` is a small JSON API (`/api/signup`, `/api/signin`, `/api/signout`, `/api/me`, `GET`/`POST /api/scores`) on Node's built-in `node:sqlite` (needs Node ≥ 22.13; prints an ExperimentalWarning). It has no separate process: the `sqliteApi` plugin in `vite.config.ts` mounts it into both `npm run dev` and `vite preview`. The database is `data/treasure.db` (gitignored, created on first start; tables `users`, `sessions`, `scores`). Passwords are scrypt-hashed and session tokens are stored as sha256. The client side is `src/api.ts` (token in `localStorage`), `src/components/AuthScreen.tsx` and `ScoreHistory.tsx`. In `App.tsx` the `player` state is `null` (show the auth screen), guest, or a signed-in user; only signed-in players' final scores are POSTed, once per `round`. A static deploy (Vercel / GitHub Pages) has no `/api`, so only guest mode works there.
- `src/components/ui/` is a generated shadcn/ui component library (Radix-based, `cn()` in `utils.ts`). Only `Button` is used today; reuse these components instead of writing new primitives.
- `src/components/figma/ImageWithFallback.tsx` and `src/guidelines/Guidelines.md` (an empty template) are Figma Make leftovers.

## Gotchas

- **Tailwind is precompiled.** `src/index.css` is a static Tailwind v4 output (imported by `main.tsx`); the project has no Tailwind dependency or PostCSS step. A utility class that does not already appear in `index.css` has no effect — use inline `style` or add the CSS rule to `index.css` by hand. `src/styles/globals.css` holds the theme tokens but is not imported anywhere.
- `vite.config.ts` aliases versioned specifiers such as `'@radix-ui/react-slot@1.1.2'` to the plain package names, because the Figma-generated `ui/` files import with version suffixes. Add an alias when adding a new versioned import. `@` resolves to `src/`.

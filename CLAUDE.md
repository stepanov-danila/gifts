# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this project is

A small, single-purpose React SPA: an **advent-calendar-style gift app**. The
screen is a grid of "gift days"; each tile unlocks on or after its date, and
opening a tile shows a dialog with the gift reveal. It is a personal, gifted
web page — the UI text is in **Russian**, and the content in `src/data.js` is
hand-edited per occasion (the current data set is for New Year 2027).

It is a Create React App (react-scripts 5) project deployed to GitHub Pages at
`https://stepanov-danila.github.io/gifts` (see `homepage` in `package.json`).

There is **no backend and no router** — all state lives in `localStorage`.

## Commands

```bash
npm install          # or: npm ci   (node_modules is not committed)
npm start            # dev server on http://localhost:3000
npm run build        # production build into ./build
npm test             # jest + RTL in watch mode
npm run deploy       # predeploy runs build, then gh-pages -d build
```

Verified toolchain: Node 22 / npm 10 work fine with react-scripts 5.0.1.

### Build and test caveats — read before "fixing" anything

- **`npm run build` succeeds; `CI=true npm run build` fails.** CRA treats
  ESLint warnings as errors when `CI` is set, and the codebase currently has 5
  pre-existing warnings in `src/components/GiftDay.jsx` (unused `TextField`,
  unused `Light` import, unused `giftName`/`setGiftName`, missing `alt` on the
  `<img>`). These are leftovers from a removed feature (see *Known dead code*).
  Do not assume a red `CI=true` build is caused by your change — check against
  the baseline first. Cleaning them up is a welcome, safe change.
- **There are no test files.** `src/setupTests.js` exists but nothing matches
  jest's `testMatch`, so `CI=true npm test` exits 1 with "No tests found".
  Use `--passWithNoTests` if you need a green test command.
- There is **no CI workflow, no linter config beyond CRA's `eslintConfig`, and
  no formatter**. Match the surrounding style by hand.

## Layout

```
public/            CRA static shell (index.html title: "New Year"), icons,
                   lights.svg, frame.png
src/
  index.js         CRA entry; mounts <App/> in StrictMode
  index.css        global resets + the blinking-lights keyframes and the
                   Yatra One Google Font @import
  App.jsx          state owner: loads/persists giftDays, unlock/receive handlers
  data.js          THE CONTENT FILE — the array of gift days
  components/
    GiftDayGrid.jsx  CSS-grid layout, maps giftDays -> <GiftDay/>
    GiftDay.jsx      one tile + its reveal <Dialog/>
    Light.jsx        the fixed garland of blinking bulbs (inline SVG)
  App.css, logo.svg, reportWebVitals.js, setupTests.js
                   CRA boilerplate, effectively unused (App.css is never
                   imported; App.jsx styles with Emotion instead)
```

## How it works

1. `App.jsx` mounts, reads `localStorage["gifts"]`.
   - If present → that JSON becomes `giftDays`.
   - If absent → `data` from `src/data.js` is used **and immediately written to
     `localStorage`**.
2. `GiftDayGrid` renders one `GiftDay` per entry, spreading the entry's fields
   as props (`{...giftDay}`) plus the two handlers.
3. Each `GiftDay` runs an effect on mount comparing today's date against its
   `date`; if it is locked and the date has arrived it calls `handleLock(id)`.
4. `handleLock`/`handleRecieve` in `App.jsx` clone the array with `_.clone`,
   mutate the matching entry, `setGiftDays`, and re-write `localStorage`.

### Gift day shape (`src/data.js`)

```js
{
  id: "2",                 // string, must be unique — used as the React key
  date: "2024-12-31",      // ISO date; tile unlocks on/after this day
  gift: { text: "...", goal: "" },  // currently NOT rendered (see below)
  background: "#c92943",   // inline background of the tile
  locked: false,           // true = still counting down
  recieved: false          // note the spelling — "recieved", not "received"
}
```

Most entries in `data.js` are **commented out** — that is the working style of
this repo, not an accident. To add a day, uncomment or append an object and
give it a unique `id`.

## Gotchas that will bite you

- **`localStorage` shadows `data.js`.** Once a browser has written the `gifts`
  key, editing `src/data.js` changes nothing for that browser. When testing
  content changes, clear it first:
  `localStorage.removeItem("gifts")` in the devtools console, or use a private
  window. The same applies to the deployed site for returning visitors.
- **`locked` semantics are inverted from the name.** `handleLock(id)` sets
  `locked = false` — it *unlocks*. Read it as "resolve the lock state".
- **The unlock date check compares day, month, and year independently**
  (`todayDay >= giftDay && todayMonth >= giftMonth && todayYear >= giftYear`)
  rather than comparing timestamps. This is wrong across month boundaries —
  e.g. on 2027-01-01 a tile dated 2026-12-31 stays locked because `1 >= 31` and
  `0 >= 11` are false. If you touch unlocking logic, prefer comparing `Date`
  values directly. Keep in mind that existing users' `localStorage` already
  holds whatever state the old logic produced.
- **There is a stray `console.log(locked)`** in the `GiftDay` effect.
- **The reveal dialog loads a hardcoded remote image** from a
  `psv4.userapi.com` URL. That link is expiring/ephemeral; the same image is
  committed at `public/frame.png` and should normally be referenced as
  `` `${process.env.PUBLIC_URL}/frame.png` `` instead (the site is served from
  the `/gifts/` sub-path, so a bare `/frame.png` will 404).
- **The grid is currently `1fr` × `1fr`** — a single full-screen tile, matching
  the single active entry in `data.js`. If you re-enable more days, restore a
  multi-column/row template in `GiftDayGrid.jsx` (it was
  `grid-template-columns: 1fr 1fr; grid-template-rows: repeat(4, 1fr);`).
- **`_.clone` is a shallow clone**, so the handlers mutate the original entry
  objects. It works because the array reference changes, but don't rely on
  the previous state being intact.

### Known dead code

Removed in the latest commit but left behind: `gift.text` / `gift.goal` are no
longer rendered, so the "guess the word to unlock your gift" flow, the
`TextField`, the `giftName` state, and `handleRecieve` (still threaded from
`App` → `GiftDayGrid` → `GiftDay`) are all currently unused. `Light` is
imported into `GiftDay.jsx` but only rendered from `App.jsx`. Decide
deliberately whether a task wants that flow restored or the leftovers deleted.

## Conventions

- **Components are `.jsx`; plain modules are `.js`.** Components live in
  `src/components/`, one per file, default-exported, most wrapped in
  `React.memo`.
- **Styling is Emotion `styled`, not CSS files.** The established pattern is to
  import a MUI component under a `Mui*` alias and re-export a styled version
  under the plain name:
  ```js
  import { Card as MuiCard } from '@mui/material';
  const Card = styled(MuiCard)`...`;
  ```
  Global CSS (resets, keyframes, font import) belongs in `src/index.css`.
- **Fonts:** `'Yatra One', sans-serif`, imported via Google Fonts at the bottom
  of `index.css`.
- **Icons** come from `react-icons` (`react-icons/md`, `react-icons/io`). Note
  it is listed under `devDependencies` even though it is a runtime dependency.
- **Handlers passed to memoized children are wrapped in `useCallback`** in
  `App.jsx` — keep that when adding new ones.
- **User-facing strings are Russian.** Keep new UI copy in Russian.
- Commit messages in this repo are short and informal ("gift 3", "fixes",
  "2025"). Match that or write something clearer — either is fine.

## Making changes

- **New gift day** → edit `src/data.js` only; nothing else needs to change
  (remember the `localStorage` gotcha and, for >1 tile, the grid template).
- **Change the reveal content** → `GiftDay.jsx`, inside the `<Dialog><Card>`.
- **Change the garland** → `Light.jsx` for the SVG bulbs, `index.css` for the
  `blink-*` animations. Bulb `<path>` elements carry `class="blink-<color>"`
  (optionally `--reverse`) which maps to the keyframes in `index.css`. Note the
  inline SVG uses HTML attribute names (`class`, `fill-rule`) rather than JSX
  ones; React warns but renders — converting them to `className`/`fillRule` is
  a safe cleanup.
- **Deploying** → `npm run deploy` builds and pushes `build/` to the `gh-pages`
  branch. Don't run it unless explicitly asked; it publishes to the live site.

## Repository notes

- Default branch: `master`.
- `build/`, `node_modules/`, and `coverage/` are gitignored — never commit them.
- `README.md` is the untouched CRA boilerplate and says nothing about this app;
  this file is the real documentation.

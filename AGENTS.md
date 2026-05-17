# Hello Garee's Recipe Planner

Personal weekly meal planner and shopping list generator. No build step, no framework.

**Live URL**: https://gareetest.github.io/Hello-Garee-recipes/
**GitHub repo**: https://github.com/GareeTest/Hello-Garee-recipes

## Files

```
index.html    — app shell: HTML structure, CSS, all JS logic
recipes.js    — BUILTIN_RECIPES array only (loaded via <script src="recipes.js">)
images/       — recipe card photos (JPEG, already rotation-corrected)
404.html      — GitHub Pages 404 fallback
AGENTS.md     — this file
```

To test locally: open `index.html` in a browser (recipes.js must be in the same directory).
To deploy: `git push` to main — GitHub Pages serves from repo root.

## App Structure

Four tabs: **Recipes**, **Weekly Plan**, **Shopping List**, **History**.

- **Recipes tab**: grid of recipe cards. Filter by tag/time/diet. Select meals for the week, view steps (👨‍🍳), edit (✏), archive (🗃), or add custom.
- **Weekly Plan tab**: selected meals with serving size dropdowns. Save current week to history (💾). Mark Week as Cooked.
- **Shopping List tab**: aggregated fresh + pantry ingredient list. Tap item to mark as got (strikethrough). Tap amount to edit inline. Sainsbury's search links per item.
- **History tab**: saved weekly plans. Load a past plan into the weekly plan (📥 Load Plan), or share via Gmail.
- **Help modal**: `?` button top-right of header. Opens a modal with per-tab feature tips.

## Data Model

### Recipe object
```js
{
  id: Number,           // built-ins: 10–625; custom: Date.now() (large int)
  name: String,
  subtitle: String,
  time: String,         // e.g. "35 mins"
  maxTime: Number,      // parsed int, used for time filter/sort
  img: String|null,     // relative path e.g. "images/foo.jpg", or null
  type: String,         // "Main" (default)
  tags: String[],       // e.g. ["Veggie", "Calorie Smart", "Pescatarian"]
  highProtein: Boolean,
  lowCal: Boolean,
  highFibre: Boolean,
  lowCarb: Boolean,
  ingredients: {
    "2P": [{name, amount, unit}],
    "3P": [{name, amount, unit}],
    "4P": [{name, amount, unit}],
    pantry: {
      "2P": [{name, amount, unit}],
      "3P": [{name, amount, unit}],
      "4P": [{name, amount, unit}]
    }
  },
  steps: [{n: Number, title: String, text: String}]
}
```

### Saved plan object (Plan History)
```js
{
  id: Number,           // Date.now() at save time
  name: String,         // user-entered label e.g. "Week of 30 May 2026"
  date: String,         // ISO timestamp
  recipes: [{ id, name, time, serving }]
}
```

## localStorage Keys

| Key | Type | Purpose |
|---|---|---|
| `weeklyPlan` | `{ [id]: "1P"\|"2P"\|"3P"\|"4P" }` | Currently selected meals |
| `customRecipes` | `Recipe[]` | User-added recipes |
| `recipeOverrides` | `{ [id]: Partial<Recipe> }` | Edits to built-in recipes |
| `recipeCookCounts` | `{ [id]: Number }` | Times each recipe cooked |
| `archivedRecipes` | `id[]` | Recipes hidden from main grid |
| `savedPlans` | `SavedPlan[]` | Plan History snapshots |
| `shopGotItems` | `string[]` | Ticked shopping items — keys in `"type:ingKey"` format (e.g. `"fresh:garlic"`). Survives refresh; cleared when the plan changes. |
| `shopPlanKey` | `string` | JSON fingerprint of the plan that built the current shopping list. Used to detect plan changes across refresh. |
| `shopActivePlanId` | `number\|null` | ID of the history plan currently shown in the shopping list, or `null` for the current week's plan. Persisted so the history list survives refresh. |

### Persistence helpers
```js
function ls(key, def)      // read from localStorage (safe JSON parse)
function lsRaw(key, val)   // write WITHOUT triggering cloud sync (used during merge)
function lsSave(key, val)  // write AND schedule cloud sync (use for all normal saves)
```

### Override pattern
Edits to built-in recipes are stored in `recipeOverrides`, not in `BUILTIN_RECIPES`. `getAllRecipes()` merges at read time:
```js
function getAllRecipes() {
  const ov = getOverrides();
  const builtins = BUILTIN_RECIPES.map(r => ov[r.id] ? { ...r, ...ov[r.id] } : r);
  return [...builtins, ...getCustomRecipes()];
}
```

## Shopping List

### Ingredient aggregation
- Fresh and pantry ingredients are accumulated separately into `{ [normalisedKey]: { name, units: { unit: amount } } }` stores.
- Dedup key: `_ingKey(name)` — runs `_ingCleanName` (strips qualifier prefixes), lowercases, strips common food suffixes (`cheese`, `fillet`, `steak`, `leaf`, `cloves`, etc.), normalises plurals (`onions→onion`, `tomatoes→tomato`, `carrots→carrot`, `shallots→shallot`, `courgettes→courgette`, `mushrooms→mushroom`, etc.) so "Red onion" and "red onions" merge to one line.
- **Qualifier prefix stripping**: `_ingQualifiers` regex strips leading words like `grated`, `finely chopped`, `roughly chopped`, `freshly squeezed`, `freshly ground` from both the dedup key and the stored display name via `_ingCleanName(name)`. This merges e.g. "Grated Italian hard cheese" and "Italian hard cheese" into one line.
- **Display name capitalisation**: `_capitalise(s)` is applied to `item.name` at render time in `_shopRows` so the first letter is always uppercase regardless of which recipe contributed the name first.
- Unit normalisation: `carton/cartons → tin`, `clove → cloves`, `bunch → bunches`, `sachet → sachets`, `stock pot → stock pots`.
- Amounts for the same ingredient in different units are kept separately and formatted as e.g. `(100g and 2 tbsp)`.
- **Combined string routing**: Some HelloFresh recipes put combined pantry strings (e.g. `"butter, olive oil, pepper, salt (or dietary alternatives)"`) in the fresh ingredient array rather than the pantry object. `buildShoppingList` detects these by checking `ing.name.includes(',')` and routes them through pantry expansion + trivial filter so they never appear in the fresh list.

### Pantry combined strings
Some recipes list pantry items as a combined string e.g. `"olive oil, salt, water (or dietary alternatives)"`. `_expandPantryIng()` splits these on commas and strips the `(or ...)` suffix before aggregation. Non-comma items also get the `(or ...)` suffix stripped so keys match.

### Trivial pantry filtering
Common household staples are hidden from the shopping list unless the quantity is substantial (>50g or >50ml):
- **Always hidden**: water
- **Hidden unless >50g/ml**: butter, olive oil, pepper, salt, sugar, veg oil, vegetable oil, sunflower oil, rapeseed oil, flour

### Shopping list state
State is split between in-memory vars (session) and localStorage (survives refresh):

- `_shopPlanKey` (in-memory) / `shopPlanKey` (localStorage) — JSON fingerprint of the plan the list was built from. Both set by `buildShoppingList`. `showPage('shop')` compares `ls('shopPlanKey')` to `currentKey` so plan-change detection survives refresh.
- `_shopFromHistory` (in-memory) — `true` when the list was built from a history serving map rather than `state.plan`.
- `shopActivePlanId` (localStorage) — ID of the history plan shown, or `null`. Restored in `showPage('shop')` so the history list survives refresh.
- `shopGotItems` (localStorage) — set of ticked item keys. Read by `_shopRows` at render time; updated by `toggleShopGot`. **Not cleared on tab navigation or refresh** — only cleared when the plan genuinely changes or a new list is explicitly started.

**Build triggers in `showPage('shop')`** (when `_skipShopBuild` is false):
1. `shopActivePlanId !== null` → restore history plan list (without clearing got items).
2. `shopActivePlanId === null` AND `ls('shopPlanKey') !== currentKey` → plan changed → clear got items, rebuild.
3. `shopActivePlanId === null` AND `ls('shopPlanKey') === currentKey` AND `_shopPlanKey === null` → first render after refresh, plan unchanged → rebuild without clearing got items (ticks restored).
4. Everything matches in-memory → skip rebuild entirely (ticks survive tab navigation).

`goToShoppingList()` — explicit entry from Weekly Plan tab. Clears `shopActivePlanId` and `shopGotItems`, calls `buildShoppingList()`, then sets `_skipShopBuild = true`.

### Shopping list interactions
- **Tap item**: toggles strikethrough ("got" state). Persisted in `shopGotItems` localStorage — survives tab navigation and page refresh. Cleared only when the plan changes or a new list is explicitly built.
- **Tap amount**: `contenteditable` span — edit quantity inline
- **🛒 link**: opens Sainsbury's search in new tab
- **Open Fresh / Open Pantry**: `openSainsburys(type)` — opens one Sainsbury's search tab per unticked item in that section. Skips `.got` items. Shows a `confirm()` dialog first with the tab count and a reminder to be logged into Sainsbury's.
- **Print**: ticked (`.got`) items are hidden via `@media print { .shopping-item.got { display: none } }` — only unticked items appear in the printout.

### Sainsbury's search overrides
HelloFresh-specific ingredient names are remapped for better search results:
- `white wine stock powder/paste` → `small white wine bottle`
- `red wine stock paste` → `red wine stock`
- `vegetable/chicken stock paste` → `vegetable/chicken stock`
- `intense tomato` → `vine tomato`
- `central american style spice mix` → `mexican spice mix`

## Auto-Tag Detection

`_effectiveTags(r)` is the single source of truth for a recipe's tags — use it everywhere tags are displayed or filtered, never `r.tags` directly.

- **Pescatarian**: auto-added if `_hasSeafood(r)` detects any 2P ingredient matching `_seafoodKeywords` (salmon, prawn, cod, mackerel, tuna, etc.)
- **Veggie**: auto-added if `_hasMeat(r)` returns false AND `_hasSeafood(r)` returns false. `_hasMeat` checks 2P ingredients against `_meatKeywords` with `_meatExclusions` to avoid false positives (e.g. "chicken stock paste" must not trigger meat detection).

Recipes already manually tagged Pescatarian or Veggie pass through unchanged. Auto-detected tags display as tag badges on recipe cards, in the weekly plan view, and in the steps modal.

## Plan History & Sharing

### Saving
`savePlanToHistory()` snapshots the current weekly plan (meal IDs + serving sizes) into `savedPlans` localStorage. User is prompted for a name.

### Sharing
`sharePlan(planId)` encodes the plan as base64 JSON (`TextEncoder` → `btoa` → `encodeURIComponent`) and opens Gmail compose with a pre-filled body containing the URL. When the recipient opens the URL, `checkForSharedPlan()` (wired to `DOMContentLoaded`) detects `?plan=` in the URL, decodes it (`atob` → `TextDecoder`), shows an import modal, strips the param from the address bar via `history.replaceState`. New links use `TextEncoder/TextDecoder` encoding — old links using deprecated `escape/unescape` are no longer compatible.

### Loading a saved plan
`loadHistoryPlan(planId)` — triggered by the **📥 Load Plan** button on history cards. Replaces `state.plan` with the saved plan's meals and serving sizes, saves to localStorage, clears `shopActivePlanId` and `shopGotItems`, re-renders the recipe grid (updating green in-plan borders), then navigates to the Weekly Plan tab.

### History shopping list (internal)
`goToHistoryShoppingList(planId)` — not exposed in the UI; used internally by `showPage('shop')` to rebuild a history list on refresh. Builds the list from a saved plan's serving map, shows the green "📚 Shopping list for: …" label, saves `shopActivePlanId`, and sets `_skipShopBuild = true` before calling `showPage('shop')`.

## Cloud Sync (Firebase)

Firebase Auth + Firestore — active and configured. SDK v10.12.2 compat loaded via CDN defer scripts.

```js
const FIREBASE_CONFIG = { apiKey: "...", authDomain: "hello-garee-recipes.firebaseapp.com", ... };
```

Each user's data lives at `users/{uid}` in Firestore. Writes are debounced 1.5s via `scheduleSaveToCloud()`.

### Merge strategy (cloud → local, and Import)
Additive — never deletes local data:
- `customRecipes`: add cloud-only IDs (local wins on conflicts)
- `recipeOverrides`: local wins on conflicts
- `recipeCookCounts`: `Math.max` per ID
- `archivedRecipes`: union
- `weeklyPlan`: local wins on conflicts
- `savedPlans`: add cloud-only plan IDs

### Firestore document (`users/{uid}`)
```js
{ customRecipes, recipeOverrides, recipeCookCounts, archivedRecipes, weeklyPlan, savedPlans, lastSaved }
```

### Firestore security rules
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

## CSS Design System

```css
:root { --green: #2d6a4f; --green-dark: #1b4332; }
```
All green usages use `var(--green)` / `var(--green-dark)` — never hardcoded hex.

Key button classes: `btn-primary` (green filled), `btn-save-plan` (green outline), `btn-history-shop` (green filled, history cards), `btn-cooked` (dark, mark as cooked).

**Help modal `li` pattern**: `.help-section li` is `display: flex`, so any direct-child element (including `<strong>`) becomes a separate flex item and breaks out of text flow. Always wrap `li` content in a `<span>`: `<li><span>text with <strong>bold</strong></span></li>`.

Mobile: `height: 100dvh` on body (shrinks with keyboard). Search input `font-size: 1rem` prevents iOS auto-zoom.

## Ingredient Audit Notes

HelloFresh cards only print 2P and 4P quantities. 3P is usually ×1.5 of 2P, but sachets/whole items use **1/1/2** pattern (not 1/1.5/2). Always verify 3P sachet quantities — do not auto-scale. Known corrections already applied in recipes.js.

## Images

Stored in `images/`. Rotation corrections already applied with `sips` — do not re-rotate.

## GitHub Pages

Hosted from the `main` branch root. Push via GitHub Desktop or `git push`. Repo must be public, Pages enabled (Settings → Pages → Deploy from branch: main / root). **Both `index.html` and `recipes.js` must be committed** — missing `recipes.js` will break the app.

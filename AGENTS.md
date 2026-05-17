# Hello Garee's Recipe Planner

Single-file HTML/CSS/JS web app. No build step, no framework, no dependencies.

**Live URL**: https://gareetest.github.io/Hello-Garee-receipes/
**GitHub repo**: https://github.com/GareeTest/Hello-Garee-receipes

## Files

```
index.html          — the entire app (HTML + CSS + JS in one file)
images/             — recipe card photos (JPEG, already rotation-corrected)
AGENTS.md           — this file
```

To test: open `index.html` directly in a browser, or push to GitHub Pages.

## App Structure

Three tabs: **Recipes**, **Weekly Plan**, **Shopping List**.

- **Recipes tab**: grid of recipe cards. Select meals for the week, view steps (👨‍🍳), edit (✏), archive (🗃), or add new.
- **Weekly Plan tab**: shows selected meals with serving size controls. Mark Week as Cooked button increments cook counts (with yes/no confirmation — cannot be undone).
- **Shopping List tab**: aggregated ingredient list with Sainsbury's search links. Also has a Mark Week as Cooked button.

## Data Model

### Recipe object
```js
{
  id: Number,           // built-ins: 1–9; custom: Date.now() (≥1000)
  name: String,
  subtitle: String,
  time: String,         // e.g. "35 mins"
  maxTime: Number,      // parsed int from time, used for sorting
  img: String|null,     // URL or null (shows placeholder)
  tags: String[],       // e.g. ["Veggie", "Calorie Smart"]
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

## localStorage Keys

| Key | Structure | Purpose |
|---|---|---|
| `customRecipes` | `Recipe[]` | User-added recipes |
| `recipeOverrides` | `{ [id]: Recipe }` | Edits to built-in recipes (id 1–9) |
| `recipeCookCounts` | `{ [id]: Number }` | Times each recipe has been cooked |
| `archivedRecipes` | `id[]` | Hidden from main grid |
| `weeklyPlan` | `{ [id]: serving }` | Selected meals; serving = "2P"/"3P"/"4P" |

### Helper pattern
```js
function ls(key, def) { try { return JSON.parse(localStorage.getItem(key)) ?? def; } catch { return def; } }
function lsSave(key, val) { localStorage.setItem(key, JSON.stringify(val)); }
```

### Override pattern for built-in recipes
Edits to built-in recipes (IDs 1–9) are stored in `recipeOverrides`, not in `BUILTIN_RECIPES`. `getAllRecipes()` merges them:
```js
function getAllRecipes() {
  const ov = getOverrides();
  const builtins = BUILTIN_RECIPES.map(r => ov[r.id] ? { ...r, ...ov[r.id] } : r);
  return [...builtins, ...getCustomRecipes()];
}
```

## Shopping List — Smart Aggregation

Ingredients aggregated by **name only** (not unit). Same ingredient with different units → `{ name, units: { unit: amount } }` → rendered as `(100g and 2 tbsp) Olive Oil`.

Unit normalisation map (`unitNorm`):
- `carton` / `cartons` → `tin`
- `clove` → `cloves`
- `bunch` → `bunches`
- `sachet` → `sachets`

Sainsbury's search overrides (`searchOverrides`) for Hello Fresh-specific ingredient names:
- `white wine stock powder` → `small white wine bottle`
- `red wine stock paste` → `red wine stock`
- `vegetable stock paste` → `vegetable stock`
- `chicken stock paste` → `chicken stock`
- `white wine stock paste` → `small white wine bottle`

## Cloud Sync (Firebase)

Auth + Firestore via Firebase CDN SDK (v10.12.2 compat). Multi-user: each user's data lives at `users/{uid}` in Firestore.

### Setup (one-time, per deployment)
1. Create a Firebase project at firebase.google.com
2. Enable Authentication → Sign-in methods: Google + Email/Password
3. Enable Firestore (start in production mode, then set rules — see below)
4. Add a Web app → copy the config object
5. Paste the config object into `FIREBASE_CONFIG` in `index.html` (currently `null`)
6. In Firebase Console → Authentication → Settings → Authorized domains: add `gareetest.github.io`

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

### Firestore document structure (`users/{uid}`)
```js
{
  customRecipes: Recipe[],
  recipeOverrides: { [id]: Recipe },
  recipeCookCounts: { [id]: Number },
  archivedRecipes: Number[],
  weeklyPlan: { [id]: "2P"|"3P"|"4P" },
  lastSaved: Timestamp
}
```

### Key functions
- `loadFromCloud(uid)` — called on sign-in; merges cloud data into localStorage (same strategy as Import)
- `saveToCloud()` — writes all localStorage data to Firestore; called via 1.5s debounce after every `lsSave()`
- `lsRaw(key, val)` — write to localStorage WITHOUT triggering cloud sync (used during merge to avoid loops)
- `lsSave(key, val)` — write to localStorage AND schedule cloud sync (all normal saves use this)

### When Firebase is not configured
`FIREBASE_CONFIG = null` → Firebase is disabled, app works as localStorage-only. All auth UI is hidden/non-functional.

## Export / Import

Additive merge strategy — import never overwrites or deletes local data:
- `customRecipes`: add new IDs only (local wins on conflicts)
- `recipeOverrides`: local wins on conflicts
- `recipeCookCounts`: `Math.max` per ID
- `archivedRecipes`: union
- `weeklyPlan`: local wins on conflicts

## Cook Count

Incremented by "Mark Week as Cooked" (both plan and shopping list pages). Both buttons show a confirmation dialog first: "cannot be undone".

Cook count shown on recipe cards as a small green `COOKED: #` pill — static, no +/− buttons. Only shown when count > 0.

COOKED pill is last in the `.card-actions` flex row with `flex-wrap: wrap` — wraps to a new line only when there's no room.

### Merged cooked function
Both buttons call a single parameterised function:
```js
function markWeekCooked(toastId = 'cooked-toast') { ... }
// Plan tab button:    onclick="markWeekCooked()"
// Shopping tab button: onclick="markWeekCooked('cooked-toast-shop')"
```

## CSS Design System

Green colours are defined as CSS variables in `:root`:
```css
:root { --green: #2d6a4f; --green-dark: #1b4332; }
```
All green usages throughout the stylesheet use `var(--green)` or `var(--green-dark)` — never hardcoded hex. When adding new green elements, always use the variables.

## Favicon

👨‍🍳 emoji rendered as an inline SVG data URI:
```html
<link rel="icon" href="data:image/svg+xml,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100'><text y='.9em' font-size='90'>👨‍🍳</text></svg>">
```

## Weekly Plan Card Layout

Cards use the same CSS grid as the Recipes tab: `repeat(auto-fill, minmax(240px, 1fr))`.

**No image** on plan cards (unlike recipe grid cards). Card structure:
- × remove button: absolutely positioned top-right of card (`position: absolute; top: 0.4rem; right: 0.5rem`)
- Recipe name + time/tags line
- Serves label + dropdown on their own line (`.item-serves`)
- Action buttons (👨‍🍳 ✏️ 🗃) on the line below (`.item-btns`)

`.selected-item` has `position: relative` to anchor the remove button.

## Built-in Recipes (IDs 1–9)

| ID | Name | Image file |
|---|---|---|
| 1 | Lemon, Parsley and White Wine Moules Frites | moules-frites.jpg |
| 2 | Presto Pesto Pea Rigatoni | pesto-rigatoni.jpg |
| 3 | Italian Inspired Butter Bean Cassoulet | butter-bean-cassoulet.jpg |
| 4 | Vibrant Veggie Burrito Bowl | burrito-bowl.jpg |
| 5 | Arrabbiata Style Pumpkin and Sage Girasoli | girasoli.jpg |
| 6 | Hearty Double Mushroom Bourguignon | mushroom-bourguignon.jpg |
| 7 | Spiced Halloumi Steaks and Hot Honey Sauce | halloumi-steaks.jpg |
| 8 | Aubergine Parmigiana Style Traybake | aubergine-parmigiana.jpg |
| 9 | Anushka's Dauphinoise Topped Cottage Pie | cottage-pie.jpg |

Note: IDs 8 and 9 are declared in reverse order in `BUILTIN_RECIPES` (id:9 before id:8) — this is intentional for display order.

## Ingredient Audit Notes

Hello Fresh cards only print 2P and 4P quantities. 3P is usually ×1.5 of 2P, but sachets/whole items use **1/1/2** (not 1/1.5/2). Always verify 3P sachet quantities from the physical card — do not auto-scale. Known corrections already applied:
- Aubergine Parmigiana: Mixed Herbs 3P = 1 sachet (not 1.5)
- Mushroom Bourguignon: Onion 3P = 1 (not 1.5)
- Butter Bean Cassoulet: Ciabatta 2P/3P/4P = 2/3/4 (not 1/1.5/2); Tomato Puree 3P = 45g; Cheese 2P/3P/4P = 20/30/40g

## Images

Stored in `images/` directory, named to match recipes. Rotation corrections have already been applied with `sips`. Do not re-rotate.

## GitHub Pages

Hosted from the `main` branch root of `https://github.com/GareeTest/Hello-Garee-receipes`. Push via GitHub Desktop or `git push`. The repo must be **public** and GitHub Pages must be enabled (Settings → Pages → Deploy from branch: main / root).

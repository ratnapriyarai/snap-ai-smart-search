# Smart Search Prototype — SnapFinance (v2: All 8 Fixes + Viewport UX)

## Context
v1 of `smart-search-prototype.html` was built and working. A critique identified 8 structural issues. The user also identified a 9th UX issue: after the AI responds, the page scrolls the user back toward the search bar even when they are already reading the conversation/results. All 9 issues are addressed in this v2 plan. The file is a single self-contained HTML file — no backend, no build tools.

**File to edit:** `/Users/ratnapriyarai/Claude Test/smart-search-prototype.html`

---

## Fix List & Implementation Details

### Fix #1 — Brittle Fuzzy Matching (colloquialisms)

**Problem:** Queries like "I want rubber for my car" or "fix my flat" return no category match despite obvious tire intent.

**Implementation:**
Add a `TRICKY_PATTERNS` array that runs BEFORE the Levenshtein scan in `parseIntent()`:

```js
const TRICKY_PATTERNS = [
  { pattern: /rubber for.*car|new rubber|set of rubber|flat tire|fix.*flat|oil change|car.*service/i, category: "tires" },
  { pattern: /place to sit|new couch|living room set|dining set|something to sleep on|bedroom set/i, category: "furniture" },
  { pattern: /something.*watch|binge|stream|new screen|game setup|gaming setup/i, category: "electronics" },
  { pattern: /bling|sparkle|precious stone|engagement|anniversary gift|wedding ring/i, category: "jewelry" },
  { pattern: /wash.*clothes|dry.*clothes|clean.*dishes|new fridge|keep food cold/i, category: "appliances" },
  { pattern: /cover.*floor|floor.*cover|new floor|replace.*floor|upgrade.*floor/i, category: "flooring" },
];
```

In `parseIntent()`, check `TRICKY_PATTERNS` first:
```js
for (const tp of TRICKY_PATTERNS) {
  if (tp.pattern.test(rawQuery)) { intent.category = tp.category; break; }
}
```

Also expand `CATEGORY_MAP` with more synonyms:
- tires: add `"flat","oil change","car service","alignment","rotation","lube"`
- furniture: add `"coffee table","nightstand","bookshelf","loveseat","futon","bunk bed"`
- electronics: add `"airpods","earbuds","smart tv","smart home","gaming console","controller","router"`
- jewelry: add `"engagement ring","anniversary","wedding band","charm"`
- appliances: add `"washing machine","tumble dryer","deep freeze"`
- flooring: add `"area rug","wood plank","floor tile","floor mat"`

---

### Fix #2 — Store Name Extraction (dual intent)

**Problem:** Current regex only fires on financing intent and relies on narrow trigger words (`at|in the store|called|named|–`). Fails on queries like "Does House of Furniture take Snap?"

**Implementation:**
Broaden the store name extraction regex and apply it to ALL intents (not just financing):

```js
const STORE_NAME_PATTERNS = [
  /(?:at|in the store|called|named|[-–:])\s+(.{3,50}?)(?:\?|$|,)/i,
  /(?:does|do|is|will)\s+(.{3,50}?)\s+(?:take|accept|use|have|offer)\s+snap/i,
  /snap.*(?:at|in)\s+(.{3,50}?)(?:\?|$)/i,
];

function extractStoreName(rawQuery) {
  for (const pat of STORE_NAME_PATTERNS) {
    const m = rawQuery.match(pat);
    if (m) return m[1].trim();
  }
  return null;
}
```

When store not found in network, AI responds:
> "I couldn't find **[Store Name]** in our network. Would you like me to search for [category] stores near you instead?"
And show clickable category buttons: `[🔧 Tires] [🛋️ Furniture] ...`

---

### Fix #3 — Price Filter Honest Display

**Problem:** Price filter extracted but mock data has no product-level pricing, so filtering is meaningless.

**Implementation:**
- Do NOT filter stores by price (no data to filter against)
- Display a **search context chip** inside the AI message bubble:
  ```
  🔍 Searching: Furniture stores · Under $300
  ```
  This chip appears inline in the AI response text, styled as a pill badge
- Show the `price-badge` on ALL returned store cards (already done in v1 for the matched stores)
- Update `runSearch()` to include the price context in the AI message text naturally:
  > "Here are furniture stores near Dallas. These stores carry items — look for products **under $300**. 💰"

---

### Fix #4 — More Cities in Mock Data

**Problem:** Only 13 cities. Queries for NYC, LA, Boston, etc. always fall to fallback.

**Implementation:**
Add 10 stores covering 7 new cities to `MOCK_STORES` (total: 30 stores):

New stores to add:
- **New York, NY** — furniture + electronics (2 stores)
- **Los Angeles, CA** — tires + furniture (2 stores)
- **Boston, MA** — appliances + jewelry (2 stores)
- **Portland, OR** — flooring + electronics (2 stores)
- **Charlotte, NC** — tires + furniture (2 stores)

Also add these cities + states to `CITY_LIST` and `CITY_STATE`:
```js
"new york": "NY", "los angeles": "CA", "boston": "MA",
"portland": "OR", "charlotte": "NC", "nyc": "NY", "la": "CA"
```

---

### Fix #5 — AI Dropdown Panel Uses Conversation Context

**Problem:** `updateDropdown()` AI panel hint is always generic when conversation is in progress.

**Implementation:**
Rewrite the context logic in `updateDropdown()` to read `state`:

```js
function getAiPanelHint(q) {
  // Priority 1: awaiting info — guide user
  if (state.awaitingZip) {
    return `Still looking for ${state.pendingIntent?.category || "stores"} — share your ZIP or city to continue.`;
  }
  if (state.awaitingCategory) {
    return `What kind of store? Try: Tires, Furniture, Electronics, Jewelry, Appliances, or Flooring.`;
  }
  // Priority 2: conversation has results — offer follow-up
  if (state.lastShownStores.length > 0) {
    const cat = state.lastShownStores[0]?.category;
    const cfg = CAT_CONFIG[cat] || {};
    return `You're browsing ${cfg.label || "stores"} — ask a follow-up, refine by city, or search a new category.`;
  }
  // Priority 3: keyword-based hints
  for (const h of AI_HINTS) {
    if (h.trigger.test(q)) return h.hint;
  }
  return "Ask me anything — describe what you need and I'll find Snap-approved stores near you.";
}
```

---

### Fix #6 — Mobile Dropdown as Full-Screen Overlay

**Problem:** On mobile, virtual keyboard pushes viewport up and dropdown gets hidden behind it.

**Implementation:**
Add to `@media (max-width: 768px)` CSS:
```css
@media (max-width: 768px) {
  #dropdown {
    position: fixed;
    top: 0; left: 0; right: 0; bottom: 0;
    border-radius: 0;
    overflow-y: auto;
    z-index: 500;
    display: none;
  }
  #dropdown.open { display: flex; flex-direction: column; }
  .dropdown-mobile-header {
    display: flex; align-items: center; justify-content: space-between;
    padding: 16px; border-bottom: 1px solid var(--gray-200);
    position: sticky; top: 0; background: white; z-index: 1;
  }
  .dropdown-close-btn {
    background: none; border: none; font-size: 24px;
    cursor: pointer; color: var(--dark); padding: 4px 8px;
  }
}
@media (min-width: 769px) {
  .dropdown-mobile-header { display: none; }
}
```

Add mobile header HTML inside `#dropdown`:
```html
<div class="dropdown-mobile-header">
  <span style="font-weight:700;font-size:15px;">Search Snap Stores</span>
  <button class="dropdown-close-btn" onclick="hideDropdown()">✕</button>
</div>
```

---

### Fix #7 — Financing Card Appended, Not Replacing Results

**Problem:** `renderFinancingCard()` does `section.innerHTML = card` which destroys any existing store cards the user was viewing.

**Implementation:**
Change `renderFinancingCard()` to **append** the financing card as a new AI message-style bubble in the chat thread (not in `results-section`). This preserves conversation context and store cards:

```js
function renderFinancingCard(storeName = null) {
  // Append as AI "card message" inside chat thread
  const storeNote = storeName
    ? `<p style="...">📌 Store: <strong>${escHtml(storeName)}</strong></p>`
    : "";
  const cardHtml = `
    <div class="financing-card inline-card">
      ...same content...
    </div>`;
  // Use appendAiMsg but inject card HTML directly after text message
  setTimeout(() => {
    const thread = document.getElementById("chat-thread");
    const row = document.createElement("div");
    row.className = "msg-row"; // AI-side row, no bubble wrapping
    row.innerHTML = `<div class="msg-avatar ai">S</div><div style="flex:1">${cardHtml}</div>`;
    thread.appendChild(row);
    row.scrollIntoView({ behavior: "smooth", block: "nearest" });
  }, 700);
}
```

Add `.inline-card` CSS variant (same as `.financing-card` but without fixed top margin).

---

### Fix #8 — Stronger "Ask Snap AI" Visual Identity in Dropdown

**Problem:** AI panel section looks like a slightly different suggestion row — doesn't feel like a distinct AI product.

**Implementation:**
Update `.snap-ai-panel` CSS:
```css
.snap-ai-panel {
  background: linear-gradient(135deg, #e8f5ee 0%, #eef1fb 100%);
  border-top: 2px solid var(--green);
  padding: 16px;
}
.snap-ai-label {
  font-size: 12px; font-weight: 800;
  color: var(--green); letter-spacing: .8px;
  display: flex; align-items: center; gap: 6px;
  margin-bottom: 5px;
}
/* Animated pulse on the avatar */
.snap-ai-avatar {
  background: linear-gradient(135deg, var(--green), var(--blue));
  animation: aiPulse 2.5s ease-in-out infinite;
}
@keyframes aiPulse {
  0%, 100% { box-shadow: 0 0 0 0 rgba(27,132,74,0.3); }
  50%       { box-shadow: 0 0 0 6px rgba(27,132,74,0); }
}
```

Also change the section label from plain text to a styled pill:
```html
<div class="dropdown-ai-header">
  <span class="ai-pill">✦ Snap AI</span>
  <span style="font-size:11px;color:var(--dark-60)">Conversational search</span>
</div>
```

---

### Fix #9 (New) — Viewport Preservation: Don't Scroll User Back to Search

**Problem:** `renderStoreCards()` and `renderFinancingCard()` call `section.scrollIntoView({ block: "start" })` which yanks the user back up even when they are already in the conversation/results area reading the AI response.

**Root cause:** The `results-section` div is populated 700ms after the AI message. By then, the user may have scrolled down to read the chat. Calling `scrollIntoView` on `results-section` scrolls it to the TOP of the viewport — pulling focus away from the conversation thread.

**Implementation:**
Add a helper `smartScroll()` that only scrolls DOWN, never UP:

```js
function smartScroll(element) {
  const rect = element.getBoundingClientRect();
  const alreadyVisible = rect.top >= -50 && rect.top <= window.innerHeight * 0.8;
  // Only scroll if the element is BELOW the current viewport
  if (!alreadyVisible && rect.top > window.innerHeight) {
    element.scrollIntoView({ behavior: "smooth", block: "start" });
  }
  // If element is above viewport or partially visible — don't scroll at all.
  // The user is already reading nearby content. New content renders below naturally.
}
```

Replace ALL `section.scrollIntoView(...)` calls inside `setTimeout` blocks in:
- `renderStoreCards()` → `smartScroll(section)`
- `renderFinancingCard()` → `smartScroll(row)` (on the appended row)

Additionally: track `state.userInChat` using a scroll listener:
```js
let userInChat = false;
window.addEventListener("scroll", () => {
  const hero = document.querySelector(".hero");
  if (hero) {
    const heroBottom = hero.getBoundingClientRect().bottom;
    userInChat = heroBottom < 0; // hero is fully above viewport
  }
}, { passive: true });
```

In `appendAiMsg()`: when `userInChat` is true, scroll the NEW message row into view using `block: "nearest"` (scrolls down gently, never up).
In `appendUserMsg()`: always scroll to the new user message with `block: "end"` (user just submitted it, always want to see it).

---

### Fix #10 (New) — Use Device Geolocation When Location Permission is ON

**Problem:** Currently the AI always asks "What's your ZIP code?" even when the user's browser has location turned on. A smarter experience auto-detects the user's city/ZIP and skips the follow-up entirely.

**Implementation:**

**On page load**, silently attempt geolocation (non-blocking, no prompt shown yet):
```js
const geoState = { lat: null, lon: null, city: null, zip: null, fetched: false };

function tryAutoGeolocate() {
  if (!navigator.geolocation) return;
  // Use getCurrentPosition — only fires prompt if permission is already "granted"
  // If permission is "prompt", we'll ask on demand instead
  navigator.permissions?.query({ name: "geolocation" }).then(perm => {
    if (perm.state === "granted") {
      navigator.geolocation.getCurrentPosition(pos => {
        geoState.lat = pos.coords.latitude;
        geoState.lon = pos.coords.longitude;
        reverseGeocode(geoState.lat, geoState.lon);
      });
    }
  });
}
```

**Reverse geocode** using the free `nominatim.openstreetmap.org` API (no key required):
```js
async function reverseGeocode(lat, lon) {
  try {
    const res = await fetch(
      `https://nominatim.openstreetmap.org/reverse?lat=${lat}&lon=${lon}&format=json`
    );
    const data = await res.json();
    geoState.city = data.address?.city || data.address?.town || data.address?.village || null;
    geoState.zip  = data.address?.postcode || null;
    geoState.fetched = true;
  } catch(e) { /* fail silently — fall back to asking user */ }
}
```

**On-demand location request** — when AI would ask "What's your ZIP?" instead show:
```js
async function requestLocationOrAskZip(intent) {
  // If already fetched, use it immediately
  if (geoState.fetched && (geoState.city || geoState.zip)) {
    const locLabel = geoState.city || `ZIP ${geoState.zip}`;
    appendAiMsg(`📍 Using your current location: <strong>${locLabel}</strong> — searching now...`);
    intent.location.city = geoState.city;
    intent.location.zip  = geoState.zip;
    runSearch(intent);
    return;
  }
  // Permission not yet granted — ask user with two options
  if (navigator.geolocation) {
    appendAiMsg(`${cfg.emoji} Looking for <strong>${cfg.label}</strong> stores!<br><br>
      <button class="geo-btn" onclick="requestGeoAndSearch()">📍 Use my current location</button>
      &nbsp; or enter your ZIP/city below.`);
    state.awaitingZip = true;
    state.pendingIntent = intent;
  } else {
    // No geolocation API — fall back to asking
    appendAiMsg(`What's your <strong>ZIP code or city</strong>?`);
    state.awaitingZip = true;
    state.pendingIntent = intent;
  }
}
```

**`requestGeoAndSearch()`** — triggered when user clicks the geo button:
```js
function requestGeoAndSearch() {
  navigator.geolocation.getCurrentPosition(
    async pos => {
      appendAiMsg("⏳ Getting your location...");
      await reverseGeocode(pos.coords.latitude, pos.coords.longitude);
      if (geoState.city || geoState.zip) {
        const locLabel = geoState.city || `ZIP ${geoState.zip}`;
        appendAiMsg(`📍 Got it — <strong>${locLabel}</strong>. Searching now...`);
        const intent = state.pendingIntent;
        intent.location.city = geoState.city;
        intent.location.zip  = geoState.zip;
        state.awaitingZip = false;
        state.pendingIntent = null;
        runSearch(intent);
      }
    },
    () => {
      appendAiMsg("Couldn't access location. Please type your ZIP code or city.");
      state.awaitingZip = true;
    }
  );
}
```

**Replace** the existing "Need location" block in `handleSearch()`:
```js
// OLD:
if (!intent.location.city && !intent.location.zip) {
  state.awaitingZip = true; ...
}
// NEW:
if (!intent.location.city && !intent.location.zip) {
  await requestLocationOrAskZip(intent);
  return;
}
```

**Also:** Update the search bar placeholder to indicate location awareness:
- If `geoState.city` is known → placeholder becomes `"Try: 'I need tires' — using ${geoState.city}"`
- Otherwise → keep the existing placeholder

**CSS for geo button:**
```css
.geo-btn {
  background: var(--green); color: white; border: none;
  padding: 8px 16px; border-radius: 20px; font-size: 14px;
  font-weight: 600; cursor: pointer; font-family: inherit;
  transition: background .15s;
}
.geo-btn:hover { background: var(--green-dark); }
```

---

## All Changes Summarized by Function/Location

| Location | Change |
|---|---|
| `TRICKY_PATTERNS` (new const) | Hardcoded colloquialism patterns, checked first in `parseIntent()` |
| `CATEGORY_MAP` | Expand all 6 categories with 4–6 more synonyms each |
| `MOCK_STORES` | Add 10 stores across 5 new cities (30 total) |
| `CITY_LIST` + `CITY_STATE` | Add 7 new cities |
| `extractStoreName()` (new fn) | Broader regex patterns for store name extraction |
| `parseIntent()` | 1) Check TRICKY_PATTERNS first; 2) call extractStoreName() for all intent types |
| `getAiPanelHint()` (new fn) | Context-aware AI panel hint using `state` |
| `updateDropdown()` | Call `getAiPanelHint(q)` instead of inline logic |
| `renderFinancingCard()` | Append to chat thread (not results-section), preserve existing store cards |
| `runSearch()` | Include price context in AI message text naturally |
| `smartScroll()` (new fn) | Only scroll DOWN, never UP |
| `appendAiMsg()` | Use `block: "nearest"` when `userInChat` is true |
| `appendUserMsg()` | Keep `block: "end"` (always scroll to new user msg) |
| `renderStoreCards()` | Replace `scrollIntoView` → `smartScroll()` |
| CSS `.snap-ai-panel` | Gradient bg, green top-border, pulsing avatar animation |
| CSS `.snap-ai-label` | Larger, bolder, branded |
| CSS `#dropdown` mobile | `position: fixed; full-screen` at ≤768px |
| HTML `#dropdown` | Add `.dropdown-mobile-header` with close button |
| HTML `.snap-ai-panel` | Add `ai-pill` label + "Conversational search" descriptor |
| scroll event listener | Set `userInChat` flag when hero scrolls out of view |
| `geoState` (new const) | `{ lat, lon, city, zip, fetched }` — holds device location |
| `tryAutoGeolocate()` (new fn) | On page load: silently get coords if permission already granted |
| `reverseGeocode(lat, lon)` (new fn) | Calls Nominatim OSM API (no key) → populates `geoState.city/zip` |
| `requestLocationOrAskZip()` (new fn) | Replaces the "ask for ZIP" block — uses geo if available, else offers button + text fallback |
| `requestGeoAndSearch()` (new fn) | Called by "📍 Use my current location" button in chat |
| `handleSearch()` location block | Replaced with `await requestLocationOrAskZip(intent)` |
| Search input placeholder | Dynamic: shows "using [City]" if `geoState.city` is known |
| CSS `.geo-btn` | Green pill button for the in-chat location request |

---

## Key Test Cases (unchanged + new)

| Input | Expected |
|---|---|
| "I need tires in San Jose" | Tire stores in San Jose |
| "tiress near 95101" | Fuzzy-matched tires, corrected silently |
| "I want rubber for my car" | TRICKY_PATTERN → tires, ask for location |
| "I need furniture" | Ask for ZIP/city |
| Reply "78701" | Furniture stores (Dallas) |
| "how do I apply for financing" | Financing card appended to chat (store cards not wiped) |
| "tires in Miami" (no results) | Fallback tire stores + amber banner |
| "hi" | Welcome message + examples |
| "78701" (only) | Ask what kind of store |
| "jewelry store near las vegas" | Las Vegas jewelry stores |
| "I need mattress within $300" | Furniture, price badge, ask for location |
| "Does House of Furniture take Snap?" | extractStoreName → "House of Furniture" not found → financing card + category search prompt |
| [Scroll to chat, type follow-up] | AI responds inline, user stays in chat viewport — not yanked back up |
| [Mobile, type in search] | Dropdown opens as full-screen overlay |
| "I need tires" (location ON) | Auto-detects city via GPS → skips ZIP prompt → shows stores immediately |
| "I need furniture" (location OFF, permission prompt) | Shows "📍 Use my location" button + text input option |
| "I need electronics" (location OFF, permission denied) | Falls back gracefully to asking for ZIP/city |

---

## Critical File
- **`/Users/ratnapriyarai/Claude Test/smart-search-prototype.html`** — single file, all HTML/CSS/JS inline

---

## GitHub Repository Setup

**Repo name:** `snap-ai-smart-search`
**Visibility:** Public
**Local project directory:** `/Users/ratnapriyarai/Claude Test/`

### Repo File Structure
```
snap-ai-smart-search/
├── smart-search-prototype.html   ← the prototype (existing file)
├── README.md                     ← setup + demo instructions (to be created)
├── PLAN.md                       ← design plan & critique doc (exported from plan file)
└── screenshots/                  ← empty placeholder folder with .gitkeep
    └── .gitkeep
```

### Steps to Create & Push
1. **Create repo on GitHub** using `gh repo create snap-ai-smart-search --public --description "AI-powered conversational store search prototype for Snap Finance"`
2. **Initialize git** in `/Users/ratnapriyarai/Claude Test/`: `git init`
3. **Create `README.md`** with:
   - Project overview (what it is, what it demos)
   - How to open (just double-click or open in browser — no server needed)
   - List of all smart search features (natural language, fuzzy matching, geolocation, conversational follow-ups, financing CTA)
   - All test cases to try
   - Screenshots section (placeholder)
   - Tech stack note (vanilla HTML/CSS/JS, Nominatim OSM for reverse geocoding)
4. **Create `PLAN.md`** — copy of the design plan + critique from this file
5. **Create `screenshots/.gitkeep`**
6. **Add remote, commit all files, push to `main`**

### Git Commands Sequence
```bash
cd "/Users/ratnapriyarai/Claude Test"
git init
git add smart-search-prototype.html README.md PLAN.md screenshots/.gitkeep
git commit -m "Initial commit: Snap AI smart search prototype v2"
gh repo create snap-ai-smart-search --public --source=. --remote=origin --push
```

---

## Verification Checklist
1. Open file in Chrome, DevTools → Responsive 375px
2. On mobile view: type in search → full-screen dropdown appears with close button
3. Run: "I want rubber for my car" → should detect tires (TRICKY_PATTERNS)
4. Run: "tires in NYC" → should find NYC tire store (Fix #4)
5. Run: financing query mid-conversation after seeing store cards → cards remain, financing card appended below
6. Run: "Does House of Furniture take Snap?" → store not found response + category prompt buttons
7. Scroll to chat area → type a follow-up → AI responds, page does NOT scroll back to hero
8. Inspect AI panel in dropdown → gradient background + pulsing green avatar + "Conversational search" label
9. Verify all 17 test cases from table above pass
10. Geolocation — with location permission granted: search "I need tires" → no ZIP prompt, AI says "Using [City]"
11. Geolocation — with location denied: "I need tires" → chat shows "📍 Use my location" button + fallback text option
12. In DevTools Network tab → verify Nominatim API call fires on location grant
13. GitHub — verify repo `snap-ai-smart-search` is public and all 4 files/folders are present
14. Open the raw GitHub URL for `smart-search-prototype.html` → confirm it renders in browser

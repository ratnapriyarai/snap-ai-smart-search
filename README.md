# Snap AI — Smart Store Search Prototype

> An AI-powered conversational store search prototype for [Snap Finance](https://snapfinance.com), inspired by Amazon's Rufus feature.

---

## What Is This?

A **single self-contained HTML file** (`smart-search-prototype.html`) that transforms the basic ZIP/category store locator into a natural-language, conversational search experience — no server, no API keys, no build tools required.

Open the file in any browser and start searching.

---

## How to Open

1. Download or clone this repo
2. Open `smart-search-prototype.html` in Chrome, Safari, or Firefox
3. No installation, no `npm install`, no server needed

```bash
git clone https://github.com/ratnapriyarai/snap-ai-smart-search.git
open "snap-ai-smart-search/smart-search-prototype.html"
```

---

## Features

| Feature | Description |
|---|---|
| 🗣️ **Natural language input** | Type "I need tires in San Jose" instead of selecting filters |
| 🔤 **Spelling tolerance** | "tiress" or "furiniture" still finds the right category (Levenshtein fuzzy matching) |
| 🧠 **Colloquialism detection** | "I want rubber for my car" → Tires; "Something to sleep on" → Furniture |
| 📍 **Geolocation** | If location permission is ON, auto-detects your city via Nominatim OSM — no ZIP prompt |
| 💬 **Conversational follow-ups** | Missing ZIP? It asks. Missing category? Shows clickable category buttons |
| 💳 **Financing intent detection** | "How do I apply?" → Financing card appended to chat (store cards stay visible) |
| 🏪 **Store name lookup** | "Does House of Furniture take Snap?" → Checks store network, gives direct answer |
| 💰 **Price context** | "Mattress under $300" → Shows price badge and context note in results |
| 🔄 **Fallback stores** | No stores in Miami? Shows stores in same category with amber banner |
| 📱 **Mobile full-screen dropdown** | On mobile, search dropdown becomes full-screen overlay (no keyboard overlap) |
| 👁️ **Viewport preservation** | AI responses never scroll you away from content you're already reading |
| ✨ **Branded AI panel** | Distinct "Ask Snap AI" section in dropdown with pulsing avatar and gradient background |

---

## Try These Searches

Copy and paste these into the search bar to demo each feature:

```
I need tires in San Jose
tiress near 95101
I want rubber for my car
I need furniture          → type "Dallas" when asked
how do I apply for financing
Do you provide financing in the store - House of furniture?
tires in Miami
I need a mattress within $300
jewelry store near Las Vegas
tires in NYC
hi
```

---

## Conversation Flows

### Multi-turn: category + location
```
User:  I need electronics
AI:    Looking for Electronics stores! What's your ZIP or city?
       [📍 Use my current location] button shown
User:  Denver
AI:    Found 1 Electronics store near Denver ✓
```

### Financing mid-conversation (store cards preserved)
```
User:  tires in Austin
AI:    Found 1 Tire store near Austin [card shown]
User:  how do I pay for it?
AI:    [Financing card appended in chat — tire card stays visible below]
```

### Geolocation (when permission is ON)
```
User:  I need tires
AI:    📍 Using your current location: San Jose — searching now...
AI:    Found 1 Tire store near San Jose ✓
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| UI | Vanilla HTML + CSS (no frameworks) |
| Logic | Vanilla JavaScript (no libraries) |
| Fuzzy matching | Custom Levenshtein distance (~25 lines) |
| Reverse geocoding | [Nominatim OpenStreetMap API](https://nominatim.org/) (free, no key required) |
| Mock data | 30 stores across 18 US cities, 6 categories |

---

## File Structure

```
snap-ai-smart-search/
├── smart-search-prototype.html   ← the entire prototype (HTML + CSS + JS)
├── README.md                     ← this file
├── PLAN.md                       ← full design plan, critique, and implementation docs
└── screenshots/                  ← add demo screenshots here
    └── .gitkeep
```

---

## Brand Colors

Matches [snapfinance.com](https://snapfinance.com) exactly:

| Token | Hex | Usage |
|---|---|---|
| `--green` | `#1b844a` | Primary buttons, badges, accents |
| `--blue` | `#3d5ccf` | Secondary accents, AI panel highlights |
| `--dark` | `#353849` | Text, structural elements |
| Font | Aeonik → Inter → system-ui | Body copy |

---

## Roadmap: Production Upgrades

To move from prototype to production:

1. **Replace rule-based intent with Claude API** — swap `parseIntent()` for a Claude API call that handles true NLP
2. **Connect to Snap's live store API** — replace `MOCK_STORES` with real store data
3. **Real distance calculation** — use Haversine formula to sort stores by actual GPS distance
4. **Persistent conversation** — store session in `localStorage` so context survives page refresh
5. **Voice input** — add Web Speech API for hands-free queries
6. **A/B test with snapfinance.com** — embed as a progressive enhancement to the existing search

---

## Screenshots

_Add screenshots to `/screenshots` and reference them here._

---

## License

Prototype for internal demonstration purposes — Snap Finance.

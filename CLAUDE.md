# Itemize — repo notes

This repo (`itemize-site`) is **only the marketing/landing site** for Itemize
(home inventory app — "Home inventory, done properly"). It contains
`index.html`, `privacy/`, `terms/`, favicons, and a `CNAME` pointing at
itemizeapp.ca. **The actual Itemize app codebase is not in this repo** — it
lives elsewhere and hasn't been attached to a session yet.

## Feature idea: barcode scan → auto-filled item (not yet built)

User request (from a "Buy Beaver: Origin Scanner" app screenshot): let
customers scan a barcode on an item and have Itemize instantly pull in the
item's info, the same way Itemize's existing photo-based auto-fill
(title/category/estimated value) works today.

### How Buy Beaver does it (reference app)
1. Camera-based barcode scan (UPC/EAN), tap-to-focus.
2. Barcode looked up against a product database (~3M+ products) for
   name/brand/category and manufacturing/ingredient/ownership data.
3. Rules + AI/community scoring layer turns that into their 3 criteria
   (Manufacturing Location, Materials & Ingredients, Brand Ownership) and a
   0–100 score, plus suggested alternatives.
4. Important caveat: the barcode number itself (GS1 prefix) only indicates
   which country registered the manufacturer's company prefix, **not**
   actual country of manufacture — "origin" claims require their own
   curated/crowdsourced data layer on top of raw barcode lookup, not just
   the barcode number.

### Proposed pipeline for Itemize
```
Camera scans barcode → UPC/EAN string
  → check local/app cache
  → check Itemize's own backend product DB (built up over time from prior scans)
  → miss? call a 3rd-party lookup API, cache the result
  → return {name, brand, category, image, list price}
  → prefill the existing item-creation form (same fields photo-autofill writes)
  → user still adds/confirms a condition photo, room placement, etc.
```

### Building blocks
- **Scanning (client)**: if the app is React Native/Expo, `expo-camera`'s
  built-in `CameraView` barcode mode (ML Kit on Android / VisionKit on iOS,
  no extra native module) is the simplest start. For more control later
  (scan-from-photo, low-light tuning): `react-native-vision-camera` + an
  ML Kit barcode plugin. Native iOS/Android would use VisionKit/ML Kit
  directly.
- **Product data (server)**: no barcode API gives reliable "true origin" —
  that layer would have to be Itemize's own (rules and/or crowdsourcing),
  same as Buy Beaver. For raw product lookup by UPC/EAN:
  - **Open Food Facts** — free, no key, no real rate limit, open license.
    Good for food/CPG, weak elsewhere.
  - **UPCitemdb** — 725M+ barcodes, free tier + paid, decent general retail.
  - **Go-UPC / Barcode Lookup API / Barcode Spider** — 1B+ entries, paid,
    better electronics/household coverage — matters more for a home
    inventory app than groceries do.
  - Practical approach: try free source first, fall back to paid on miss,
    and cache every successful lookup in Itemize's own DB — becomes
    Itemize's own catalog over time and cuts ongoing API cost (mirrors how
    Buy Beaver built its 3M-product database).

### Phased plan
1. **MVP** — scan barcode → lookup → prefill name/brand/category/photo into
   the existing add-item form. No match → fall back to current photo-based
   flow.
2. **Value estimate** — feed the lookup's list price into Itemize's existing
   value-estimation step as a starting point, adjusted by the user's
   condition photo.
3. **Own catalog + cache** — persist lookups server-side to stop re-paying
   for repeat scans; also the foundation if Itemize ever wants its own
   "origin"-style scoring instead of depending on a third party for it.

### Open item
To actually implement this, need the real Itemize app repo attached to a
session (`add_repo`) to know its actual stack (React Native/Expo vs. native,
existing auto-fill implementation, backend) before committing to specific
libraries/APIs above.

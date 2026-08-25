# Module 3 — Anti-Hallucination & Context Pack Plan

> **Goal:** Reduce invented claims in Social Publisher captions/posters by grounding the LLM on a real `ALLOWED_FACTS` pack, deriving promo badges server-side, using sparse-mode prompts when menu data is thin, then validating output.  
> **Status:** Phases 0–4 implemented (context pack, badge, optional facts + portal, branched prompts, validators).  
> **Service:** `social-publisher-api` (+ light portal changes)  
> **Related:** [Module3_Social_Publisher_Plan.md](./Module3_Social_Publisher_Plan.md), [Module3_Instagram_Publish_Roadmap.md](./Module3_Instagram_Publish_Roadmap.md)

---

## Problem

Today the draft path feeds Gemini:

- Merchant creative brief (`prompt`)
- Menu item: **title**, optional **description**, **price**, **imageUrl**

Many live items look like this (null fields omitted by menus-edge-api `@JsonInclude(NON_NULL)`):

```json
{
  "id": "11_classic_juicy_pork",
  "title": { "translations": { "FR": "...", "EN": "..." } },
  "baseType": "ITEM",
  "imageUrl": "https://...",
  "priceInfo": { "price": 1399 },
  "taxCategory": "qc-food"
}
```

**No description, no dietary labels, no category context, no store brand voice.**  
The model then invents ingredients, texture, “award-winning”, fake % OFF badges, etc.

Validators alone cannot fix that — they need a fact set. If the fact set is only a title + price, captions must stay constrained (sparse mode), not “confident fiction.”

**Principle (same as Module 1):** systems own facts; LLM owns wording. Promo numbers/badges are never LLM-sourced.

---

## Strategy overview

| Phase | Scope | Outcome |
|-------|--------|---------|
| **0** | Context pack from existing platform data | Richer `ALLOWED_FACTS` in every draft prompt |
| **1** | Sparse mode + optional merchant `facts` | Safe captions when menu description is empty |
| **2** | Badge from promo (server-derived) | No invented `20% OFF` on posters |
| **3** | Stronger branched prompts | Rich vs sparse system rules |
| **4** | Output validators | Caption/headline checked against `ALLOWED_FACTS` |
| **Later** | Enrich menu / vision assist | Out of scope for this pass |

**Recommendation shipped in this plan:** allow Generate when description is empty (**sparse mode**), plus optional `facts` from the portal. Do **not** hard-block Generate in v1.

---

## Phase 0 — Context pack (use what already exists)

**Goal:** Build one structured fact object before calling the LLM. No new catalog fields required.

### P0.1 Extend menu item fetch

File: `social-publisher-api/src/clients/menusClient.ts`

Parse more of `ItemResponseDTO` when present:

| Source field | Map into context as |
|--------------|---------------------|
| `title` (all locales) | `titles` map — AI follows brief language |
| `description` (all locales) | `descriptions` map |
| `priceInfo.price` | `priceCents` |
| `imageUrl` | `imageUrl` (poster only; not a text “fact”) |
| `dishInfo.classifications.dietaryLabelInfo.labels` | `dietaryLabels[]` |
| `nutritionalInfo.calories` (optional) | `calories` |
| `subCategory` | `subCategory` |

Pass **all** title/description translations into `ALLOWED_FACTS`. Do **not** pick a locale via `DEFAULT_LOCALE` — the model writes in the language of `merchant_brief`.

### P0.2 Store context

Add `storesClient.ts` (or fold into a `context/` builder):

- Fetch store **name** (and description only if a cheap merchant/public endpoint already exposes it).
- Prefer existing internal/merchant store APIs already used elsewhere; do not invent cuisine type.
- Env: `STORES_EDGE_API_URL` (mirror `MENUS_EDGE_API_URL` pattern).

If store fetch fails: log warn, continue with item-only facts (draft must not 502 solely for store context).

### P0.3 Category context (best-effort)

Menus categories hold `entities[]` with item ids — there is no `GET item → category` shortcut today.

Options (pick one in implementation):

1. **Best-effort:** `GET /menus/categories?storeId=` → find category whose `entities` contain `itemId` → use category title. Cache per store briefly (in-memory TTL) to avoid hammering.
2. **Defer:** use `subCategory` only until a menus “item categories” endpoint exists.

Plan default: implement (1) with timeout + skip on failure; never block draft.

### P0.4 `AllowedFacts` type + builder

New module e.g. `src/context/buildAllowedFacts.ts`:

```ts
interface AllowedFacts {
  itemId: string
  title: string
  description: string | null
  priceCents: number | null
  dietaryLabels: string[]
  calories: number | null
  subCategory: string | null
  categoryTitle: string | null
  storeName: string | null
  storeDescription: string | null
  promo: ResolvedPromo
  badge: string | null          // Phase 2 — server-derived
  merchantBrief: string
  extraFacts: string | null     // Phase 1 — portal optional
  mode: 'rich' | 'sparse'       // rich iff description or extraFacts present
}
```

`mode`:

- **`rich`** — non-empty `description` **or** non-empty `extraFacts`
- **`sparse`** — otherwise (title + price + store/category/dietary only)

Wire into `draftService` before `buildMessages`.

### P0.5 Prompt injection format

User message always includes a machine-readable block, e.g.:

```
ALLOWED_FACTS:
- title: Classic Juicy Pork
- price_cents: 1399
- description: (none)
- dietary: (none)
- category: Mains
- store: QuickManage Demo
- promo_badge: 20% OFF
- merchant_brief: weekend special, fun tone
- extra_facts: (none)
- mode: sparse
```

---

## Phase 1 — Sparse mode + optional merchant facts

**Goal:** When the catalog is thin, still generate — but only vibe/CTA/title, not invented product attributes.

### P1.1 API: optional `facts` on draft

`POST /api/v1/social/draft` body addition:

```json
{
  "storeId": "...",
  "itemId": "...",
  "prompt": "weekend special, fun tone",
  "facts": "slow-cooked pork, steamed buns, house chili oil",
  "generatePoster": true,
  "promo": { "discountPercent": 20 }
}
```

- Max length ~400–500 chars (same ballpark as brief).
- Trim; empty → `null`.
- Persist on draft document as `extraFacts` for audit / regenerate context.
- Portal: optional textarea “Facts for this post (optional)” under the creative brief — shown especially when item has no description (soft hint, not a hard block).

### P1.2 Sparse vs rich behavior

| Mode | LLM may | LLM must not |
|------|---------|----------------|
| **sparse** | Use title; occasion/CTA from brief; generic appetite words that don’t assert ingredients (“looks incredible”, “order now”) | Invent ingredients, cooking method, allergens, awards, origin stories, fake prices |
| **rich** | Paraphrase description + `extraFacts` + dietary labels | Add claims outside ALLOWED_FACTS; invent prices |

### P1.3 Portal UX (light)

Files under `quickmanage-merchant-portal/src/components/social/`:

- Optional facts field on composer
- Hint when item description missing: “Add a short description on the menu item, or type facts here for better captions.”
- Pass `facts` through `socialPublisher.controller.js`

No hard block on Generate in this phase.

---

## Phase 2 — Badge from promo (server-derived)

**Goal:** Poster badge text is deterministic. LLM no longer invents `% OFF`.

### P2.1 `derivePromoBadge(promo: ResolvedPromo): string | null`

File: e.g. `src/utils/derivePromoBadge.ts`

Rules (suggested):

1. If `promo.label` set → use trimmed label (merchant override)
2. Else if `hasDiscount` and both prices known → prefer percent when it was percent-based, else `Save $X.XX` / cents-aware format already used on posters
3. Else → `null`

Include percent in `ResolvedPromo` if needed (today percent is consumed inside `resolvePromo` and discarded — extend to keep `discountPercent` when provided so badge can say `20% OFF`).

### P2.2 Overwrite LLM badge

After `generateObject`:

- For every poster variation: `badge = derivedBadge` (ignore LLM badge, or stop asking for badge in schema).

Prefer **overwrite** first (smaller schema churn); optionally remove `badge` from Zod schema in a follow-up.

### P2.3 Prompt

Tell the model: “Badge is rendered by the server from promo data — do not invent discount text.”

---

## Phase 3 — Stronger branched prompts

**Goal:** System prompt depends on `mode`.

File: `src/prompts/socialDraft.ts`

### Shared rules

- Only use ALLOWED_FACTS for product claims
- Never invent prices or discount % (badge is server-side)
- Exactly 5 hashtags; caption length limits unchanged
- Prefer including the item **title** in the caption

### Sparse-only rules

- No ingredients, allergens, nutrition claims, awards, or cooking methods
- Focus on brief tone + CTA + item name
- Hashtags: brand/occasion/category-safe (#foodie ok; do not invent #GlutenFree unless in dietary labels)

### Rich-only rules

- Paraphrase description / extraFacts; do not contradict them
- Dietary labels may be mentioned only if listed

Lower temperature slightly for factual fields if the LLM client allows per-call overrides (optional).

---

## Phase 4 — Output validators

**Goal:** Fail closed on clear hallucinations before saving the draft (or auto-repair when safe).

New: `src/validation/validateDraftAgainstFacts.ts`

| Check | Action |
|-------|--------|
| Caption/headline contains `$` / price-like amounts not matching allowed cents | Strip or reject + one regenerate |
| Caption claims allergen/diet words (`gluten`, `vegan`, …) not in `dietaryLabels` | Strip sentence or reject |
| `mode === 'sparse'` and caption matches ingredient-like patterns (optional heuristic) | Soft: log + regenerate once; or accept with warning in response |
| Item title tokens missing from caption | Soft warn or append title once (product choice: soft warn in v1) |
| Poster badge ≠ derived badge | Always overwrite (Phase 2) |

Return structure:

```ts
{ ok: true, caption, hashtags, poster? } | { ok: false, reasons: string[] }
```

On hard fail: one automatic regenerate with a short “previous output violated facts” nudge; if still bad → `502`/`422` with clear message (merchant can retry / add facts).

Do **not** auto-publish anything; HITL publish gate stays.

---

## Out of scope (later)

| Idea | Why later |
|------|-----------|
| Hard-block Generate until description exists | Friction; sparse mode covers v1 |
| Vision model reading food photo for facts | Cost + must confirm with merchant |
| Writing AI description back into menus-edge-api | Catalog ownership / review flow |
| Inventory BOM ingredient names as facts | Recipes often incomplete; risk of wrong “contains” claims |
| Multi-channel prompts (FB/email) | After IG grounding is solid |

---

## File map (expected)

```
social-publisher-api/
  src/
    clients/menusClient.ts          # extend fields
    clients/storesClient.ts         # new (name)
    clients/categoriesClient.ts     # optional best-effort
    context/buildAllowedFacts.ts    # new
    utils/derivePromoBadge.ts       # new
    utils/computePromoPrice.ts      # keep discountPercent on ResolvedPromo
    prompts/socialDraft.ts          # branched rich/sparse + ALLOWED_FACTS
    validation/validateDraftAgainstFacts.ts  # new
    services/draftService.ts        # wire pack → LLM → badge overwrite → validate
    handlers/draft.handler.ts       # accept facts
    repositories/draft.repository.ts # persist extraFacts / mode

quickmanage-merchant-portal/
  src/components/social/SocialComposerPanel.jsx
  src/controllers/socialPublisher.controller.js
  src/hooks/... (if draft payload built in a hook)
```

---

## Build order

1. **P0** — `AllowedFacts` builder + richer menus (+ store; category best-effort) + prompt block  
2. **P2** — `derivePromoBadge` + overwrite variations (quick win, independent)  
3. **P1** — `facts` API + portal optional field + `mode`  
4. **P3** — Branched system prompts  
5. **P4** — Validators + single regenerate retry  

Verify with the sparse pork item: caption should not invent ingredients; badge should match promo math; rich path still works when description or `facts` present.

---

## Success criteria

| Case | Expected |
|------|----------|
| Item with only title + price, no `facts` | Sparse caption: title + brief CTA; no fake ingredients/prices |
| Promo `discountPercent: 20` | All poster badges = `20% OFF` (or merchant `label`) |
| Item with description | Rich paraphrase; no contradicting claims |
| Merchant `facts: "house chili oil"` | May mention chili oil; still no invented allergens |
| Caption with `$99` not in facts | Validator strips/rejects |

---

## Open points (decide during implementation if needed)

1. Category scan vs `subCategory` only — start with best-effort scan + TTL cache.  
2. Soft vs hard fail on title-not-in-caption — default **soft**.  
3. Persist `allowedFacts` snapshot on the draft for debugging — **yes** (useful for internship demos / audits).

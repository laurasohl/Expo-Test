# React Native rebuild of "My Words Matter"

## Context

"My Words Matter" is a free AAC (Augmentative and Alternative Communication) picture-exchange app, currently a Next.js 16 + Supabase web app at `/Users/laura/Documents/GitHub/Capstone-Project/capstone-project`. The goal is to rebuild it as a React Native app so it works natively on phones/tablets, reusing the existing Next.js API routes and Supabase backend rather than duplicating server logic. Per the user's decisions: build **inside the existing Expo scaffold** at `/Users/laura/Documents/GitHub/Expo-Test/my-app`, and aim for **full feature parity** over time.

The web app is large (auth, device pairing for kids with no login, 9 PEC template types, content filtering, schedules, emergency cards, admin support inbox). Rather than attempt all of it at once, this plan builds a solid foundation plus one complete, working vertical slice now, and lays out the rest as a phased roadmap.

Two things confirmed by reading source (not just the web app's docs) that shape this plan:
- `app/page.jsx` (web Home) renders the static `CATEGORIES` array from `lib/categories.js` directly — **no network call**. The category-browsing network calls (`/api/pec-categories`, `/api/pec-categories/browse`) only happen inside the symbol *picker* modal, a separate taxonomy (`lib/pecCategories.js`, 8 thematic topics like People/Food/Animals).
- `GET /api/symbols/search` and `GET /api/pec-categories/browse` require **no auth** — `resolveFamilyContext()` falls back to an anonymous/safe-search-on context when there's no session and no device cookie, which is exactly the RN app's state pre-Phase-2. Safe-search filtering is entirely server-side. This means the search/browse features can be built and tested before any auth work exists.

## Phase 1 (build now): Foundation + Single PEC Selector vertical slice

### Navigation: switch to a Stack

Replace the current 2-tab `AppTabs` layout with `expo-router`'s `Stack`. The web IA is a flat 9-item category grid with per-category detail screens (`Home` → `/category/[id]`) — not a 2-tab structure. A stack matches this directly and file-based routing gives it for free.

- [src/app/_layout.tsx](../../Documents/GitHub/Expo-Test/my-app/src/app/_layout.tsx) — remove `<AppTabs />`, render `<Stack>` inside the existing `ThemeProvider` + splash-screen logic.
- `src/components/app-tabs.tsx`/`.web.tsx` and `src/app/explore.tsx` — leave in place untouched (orphaned, not deleted); revisit in Phase 2 as a possible Account entry point.

### New dependencies (`npx expo install <pkg>`, not raw npm install)

| Package | Purpose |
|---|---|
| `expo-speech` | TTS, replaces the web's `window.speechSynthesis` |
| `react-native-svg` | Port the 9 inline-SVG category icons (`components/CategoryIcon.jsx`) |
| `@supabase/supabase-js` | Scaffolded now, consumed starting Phase 2 |
| `@react-native-async-storage/async-storage` | Supabase RN session storage adapter (Phase 2) |
| `expo-secure-store` | Device-pairing token storage (Phase 2) |

Per [AGENTS.md](../../Documents/GitHub/Expo-Test/my-app/AGENTS.md), check `docs.expo.dev/versions/v57.0.0/` for `expo-speech` and any `EXPO_PUBLIC_*` env var behavior before relying on remembered API shapes.

### Theme token extension

`src/constants/theme.ts`'s `Colors` object only has `text/background/backgroundElement/backgroundSelected/textSecondary`. Extend both `light` and `dark` with the web's semantic tokens (from `app/globals.css`): `textMuted`, `textOnAccent`, `error`, `bgSubtle`, `bgAccent`, `border`, `borderSubtle`, `focus`, `accentStrong`, `buttonPrimaryBg`, `buttonPrimaryText`. `useTheme()`/`ThemedText`/`ThemedView` need no structural changes — they already index generically into `Colors[theme]`.

### New files

- `src/lib/env.ts` — typed accessor for `EXPO_PUBLIC_API_BASE_URL` (Supabase env vars added but unused until Phase 2)
- `src/lib/api.ts` — fetch wrapper against the Next.js API base URL; parses `{error}` responses; header-attachment logic added but inert until Phase 2
- `src/lib/tts.ts` — thin `expo-speech` wrapper (`speak(text)`, `stop()`)
- `src/types/pec.ts` — `PecSymbol {id,name,imageUrl,license}`, `PecCategoryTile {id,label,imageUrl,imageName}`, `AppCategory` (mirrors `lib/categories.js` shape)
- `src/constants/categories.ts` — TS port of the 9-entry `CATEGORIES` array (`lib/categories.js`), including `ttsTemplate` functions (unused until Phase 2's dual-box template, harmless to port now)
- `src/components/pec/category-icon.tsx` — `react-native-svg` port of `components/CategoryIcon.jsx`
- `src/components/pec/category-grid.tsx` — renders the 9 static categories, no fetch
- `src/components/pec/symbol-card.tsx` — one result tile (`expo-image` + name + `accessibilityLabel`)
- `src/components/pec/symbol-picker.tsx` — port of `components/PecsSearch.jsx`: Search tab (debounced `GET /api/symbols/search?q=`) + Browse-by-category tab (`GET /api/pec-categories` → tap → `GET /api/pec-categories/browse?category=`). Drop the web's parent/child mode-indicator badge and `localStorage` last-search-mode memory — both depend on Phase 2's auth context.
- `src/components/pec/pec-slot.tsx` — port of `PecSlot.jsx` + `PecPickerModal.jsx`, using React Native's built-in `Modal`
- `src/components/speak-button.tsx` — `expo-speech`-backed port of `SpeakButton.jsx`
- `src/components/templates/single-selector-template.tsx` — port of `SingleSelectorTemplate.jsx` (one slot, Speak, Clear)
- `src/app/category/[id].tsx` — dynamic route: looks up the category in `src/constants/categories.ts`, renders `SingleSelectorTemplate` when `template === 'single-select'`, else a "Coming soon" placeholder — wires navigation across all 9 categories even though only one has real content this phase

### Modified files

- [src/app/_layout.tsx](../../Documents/GitHub/Expo-Test/my-app/src/app/_layout.tsx) — Stack instead of tabs
- [src/app/index.tsx](../../Documents/GitHub/Expo-Test/my-app/src/app/index.tsx) — replace placeholder content with `CategoryGrid`

### Verification

1. In `capstone-project`: `npm run dev` (port 3000).
2. In `my-app`: add `.env.local` with `EXPO_PUBLIC_API_BASE_URL=http://localhost:3000`, run `npx expo start`, open iOS Simulator (shares host network namespace, `localhost` resolves).
3. Home screen shows 9 static tiles with correct names/icons; confirm no network request fires on load (validates the "no fetch on Home" correction).
4. Tap "Single PEC Selector" → detail screen pushes, back works.
5. Tap the empty slot → modal opens with Search/Browse tabs.
6. Type "happy" → confirm `GET /api/symbols/search?q=happy` in the `next dev` log, results render.
7. Browse tab → confirm `GET /api/pec-categories` (8 tiles), then tapping one fires `GET /api/pec-categories/browse?category=...`.
8. Select a symbol → modal closes, slot shows image + name + Speak button.
9. Tap Speak → audible TTS via `expo-speech` in the simulator; Clear resets the slot.
10. Toggle simulator dark mode → all colors flip via the extended theme tokens.
11. Spot-check tap targets ≥44pt and `accessibilityLabel`s (AAC app, ADA-accessible requirement carried over from the web app).

## Phase 2 (roadmap, not built now): Auth, device pairing, remaining templates

- Auth screens (`src/app/(auth)/` — login/signup/reset) + an `AuthProvider`/`useAuthMode()` hook mirroring `context/ParentModeContext.jsx` (`supabase.auth.onAuthStateChange` + `GET /api/device/status`).
- Device pairing UI: child "enter code" screen (`POST /api/pairing/redeem`), parent "generate code" screen (`POST /api/pairing/generate`). Store the device token in `expo-secure-store`; `src/lib/api.ts` sends it as `X-Device-Token` and the Supabase access token as `Authorization: Bearer`.
- **Backend changes required in `capstone-project`** (small, backward-compatible — cross-repo dependency, needs the user's separate go-ahead since it touches the live web app):
  1. `app/api/pairing/redeem/route.js` and `app/api/device/auto-pair/route.js` — include `deviceToken: raw` in the JSON response (cookie logic untouched).
  2. `lib/familyContext.js` — fall back to an `x-device-token` header when no cookie is present.
  3. Add a shared `getRequestUser()` helper that checks `Authorization: Bearer` in addition to cookies, used by routes that currently call `supabase.auth.getUser()` directly (`app/api/pairing/generate`, `app/api/device/auto-pair`, and likely `app/api/family/*`, `app/api/schedule` POST, `app/api/emergency` POST, `app/api/account-invite/*` — audit during Phase 2).
- Remaining templates: `DualBoxTemplate` (first-then, consequence-reward — reuses `pec-slot.tsx` ×2 + already-ported `ttsTemplate` functions), `SentenceCreatorTemplate` (ordered slots, drag-reorder via already-installed `react-native-gesture-handler`), schedule family (`todays-schedule`/`daily-schedule`/`chore-list` share one component; `weekly-chore-list` separate) wired to `GET/POST /api/schedule` + `POST /api/schedule/complete`, `EmergencyCardTemplate` (parent-only edit, `GET/POST /api/emergency`).
- Account/Settings screens: profile, family invites, device list, sign out.

## Phase 3+ (light, later)

- Native onboarding overlay as a replacement for the web's `driver.js` tour (`components/SiteTour.jsx`) — not urgent.
- EAS Update as the offline/update story instead of the web's PWA service worker — not urgent.
- Admin support inbox — web-only tool, skip for mobile or lowest priority.

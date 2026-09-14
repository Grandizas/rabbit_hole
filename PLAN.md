# Rabbit Hole — Plan

A local-only browser extension that records a manually started browsing session and shows it as an interactive graph, so you can see how "Minecraft copper farm" turned into something unrelated 45 minutes later.

> **Your browsing sessions stay on this device.**

---

## 1. Scope

### MVP (in scope)

- Firefox extension (Manifest V3), built so Chromium support can be added later.
- Popup: logo/name, tracking status, Start, Stop, Open Current Map, recent sessions, privacy line.
- Background tracker that records pages and the relationships between them.
- Sessions stored locally in IndexedDB.
- Full-page map (extension page) using React Flow: auto layout, zoom, pan, drag, select, inspect, reopen page, fit to screen.
- Dark, minimal, technical visual style.

### Out of scope (for now)

Backend, accounts, sync, auth, a website, analytics, statistics dashboards, content scripts, AI summaries, sharing/export, Chrome build (the code is structured for it but it isn't shipped yet).

### Priorities (in order, and don't skip ahead)

1. Extension loads and runs
2. Start/stop tracking works
3. Navigation relationships are collected correctly
4. Sessions persist locally
5. A saved session renders as an interactive graph
6. Layout and visual polish

---

## 2. Tech stack

| Concern | Choice | Notes |
|---|---|---|
| Language | TypeScript (strict) | |
| UI | React | Popup and map page only |
| Build | Vite | Two builds: pages (ES modules) and background (single IIFE file) |
| Extension API | WebExtensions, `browser.*` namespace | Thin wrapper in `src/shared/browser.ts`, no runtime polyfill |
| API types | `@types/webextension-polyfill` (types only) | |
| Storage | IndexedDB (hand-written small wrapper) | `browser.storage` only for small tracker state |
| Graph | `@xyflow/react` (React Flow v12) | |
| Layout | `@dagrejs/dagre` | Left-to-right layered layout |
| Styling | SCSS: global tokens plus CSS Modules (`*.module.scss`) | `sass-embedded` dev dependency |
| Dev runner | `web-ext` (dev dependency) | Launches Firefox with auto-reload |

**Runtime dependencies:** `react`, `react-dom`, `@xyflow/react`, `@dagrejs/dagre`. Everything else is a dev dependency.

We won't use `webextension-polyfill` at runtime. Firefox has native promise-based `browser.*`, and Chrome MV3's `chrome.*` also returns promises for every API we use, so `browser.ts` can just export `globalThis.browser ?? globalThis.chrome`.

We also won't use an extension build framework (CRXJS, WXT, Plasmo). Two small Vite configs plus a manifest merge script are easy to follow and have no hidden behavior.

---

## 3. Project structure

The structure you proposed, with a few changes:

- A top-level `src/` so config files stay separate from source.
- A `shared/` folder, because the popup, map, and background all need the same types, DB access, and URL helpers.
- Per-browser manifest overrides, so a Chrome build is a new small JSON file and not a rewrite.
- A few more background modules, because the non-persistent background (see §5) needs explicit state handling and a serialized event queue.

```
rabbit_hole/
├── PLAN.md
├── package.json
├── tsconfig.json
├── vite.config.ts                 # builds popup.html + map.html
├── vite.background.config.ts      # builds background.js as one IIFE bundle
├── popup.html                     # Vite HTML entry → dist/popup.html
├── map.html                       # Vite HTML entry → dist/map.html
├── manifest/
│   ├── base.json                  # shared manifest fields
│   └── firefox.json               # Firefox-specific overrides (chrome.json later)
├── scripts/
│   └── build-manifest.mjs         # merges base + target → dist/manifest.json
├── public/
│   └── icons/                     # icon-16/32/48/96/128.png (copied as-is)
└── src/
    ├── background/
    │   ├── index.ts               # entry: registers ALL listeners synchronously at top level
    │   ├── tracking.ts            # decides: new node? new edge? which relationship?
    │   ├── tabs.ts                # tab → current node, pending openers, focus/active-time
    │   ├── state.ts               # persisted tracker state (survives background suspension)
    │   ├── storage.ts             # write-side session operations (start, stop, addNode, addEdge…)
    │   ├── messages.ts            # handles popup/map requests, broadcasts updates
    │   └── queue.ts               # serializes async event handling (prevents race duplicates)
    ├── popup/
    │   ├── main.tsx
    │   ├── Popup.tsx
    │   ├── components/            # StatusIndicator, SessionList
    │   └── popup.scss
    ├── map/
    │   ├── main.tsx
    │   ├── MapPage.tsx
    │   ├── layout.ts              # session → React Flow nodes/edges with dagre positions
    │   ├── components/
    │   │   ├── PageNode.tsx       # custom React Flow node
    │   │   ├── Inspector.tsx      # side panel for the selected node
    │   │   ├── MapHeader.tsx      # wordmark, session name, session switcher
    │   │   └── EmptyState.tsx
    │   └── map.scss
    ├── shared/
    │   ├── browser.ts             # browser/chrome namespace wrapper
    │   ├── db.ts                  # IndexedDB open/upgrade + read helpers (used by all contexts)
    │   ├── messages.ts            # typed message definitions
    │   └── types/
    │       └── session.ts
    ├── utils/
    │   ├── url.ts                 # isTrackable, normalizeUrl, getDomain, isRedirector
    │   ├── search.ts              # detect search results pages + extract query
    │   ├── naming.ts              # generate session names
    │   └── time.ts                # formatting helpers
    └── styles/
        ├── tokens.scss            # colors, spacing, radii, fonts
        └── base.scss              # reset + global typography
```

**Build output (`dist/`):** `manifest.json`, `background.js`, `popup.html`, `map.html`, `assets/*`, `icons/*`. You load `dist/` into Firefox.

---

## 4. Browser APIs and why we need them

| API | Used for |
|---|---|
| `webNavigation.onCommitted` | **Main signal.** Fires once per real top-level navigation, after redirects. Provides `tabId`, `frameId`, `url`, `timeStamp`, and (as hints) `transitionType` / `transitionQualifiers`. |
| `webNavigation.onHistoryStateUpdated` | Single-page-app navigations (`pushState`). Needed for YouTube, Reddit, and GitHub, which change pages without a full load. |
| `webNavigation.onCreatedNavigationTarget` | A page opened a link in a new tab/window. Provides `sourceTabId` → `tabId`, which is the most precise "opened in new tab" signal. |
| `tabs.onCreated` | Fallback opener detection via `tab.openerTabId`. |
| `tabs.onUpdated` | Late `title` and `favIconUrl` updates, which arrive after the commit (especially on SPAs). |
| `tabs.onActivated` + `windows.onFocusChanged` | Which page is currently in front, for active-time tracking and the "active node" glow. |
| `tabs.onRemoved` | Clean up per-tab tracker state. |
| `tabs.query` / `tabs.get` | Seed the first node from the current tab when a session starts; read title/favicon/incognito. |
| `tabs.create` | Open the map page, and reopen a page from the inspector. |
| `runtime.onMessage` / `sendMessage` | Popup/map ↔ background (start, stop, status) and "session updated" broadcasts. |
| `runtime.getURL` | Build the `map.html?session=<id>` URL. |
| `runtime.onStartup` | Reset tab-ID-based state after a browser restart. |
| `storage.session` | Small, frequently updated tracker state (tab → node map, pending openers, focus). Cleared when the browser closes, which is correct because tab IDs don't survive restarts. |
| `storage.local` | `activeSessionId` (should survive a restart). |
| IndexedDB | Sessions, nodes, and edges. It's structured, indexable, handles large sessions well, and can be read directly by the popup and map (same extension origin). |

### Manifest permissions

```json
"permissions": ["tabs", "webNavigation", "storage"]
```

- **No `host_permissions`** and **no content scripts.** `webNavigation` reports URLs, and `tabs` exposes `url`, `title`, and `favIconUrl`, so we never need to read page contents. This keeps the permission prompt small and makes the privacy promise easy to verify.
- `tabs` shows the install warning "Access browser tabs" and `webNavigation` shows "Access browser activity during navigation". Both are unavoidable for this product.

### Manifest sketch

`manifest/base.json`:

```json
{
  "manifest_version": 3,
  "name": "Rabbit Hole",
  "version": "0.1.0",
  "description": "See how you fell down the rabbit hole. Your browsing sessions stay on this device.",
  "permissions": ["tabs", "webNavigation", "storage"],
  "action": { "default_popup": "popup.html", "default_title": "Rabbit Hole" },
  "icons": { "48": "icons/icon-48.png", "96": "icons/icon-96.png" }
}
```

`manifest/firefox.json`:

```json
{
  "background": { "scripts": ["background.js"] },
  "browser_specific_settings": {
    "gecko": {
      "id": "rabbit-hole@local",
      "strict_min_version": "128.0",
      "data_collection_permissions": { "required": ["none"] }
    }
  }
}
```

`chrome.json` (later) would use `"background": { "service_worker": "background.js" }` and no `browser_specific_settings`.

---

## 5. Firefox limitations and pitfalls

These directly shape the tracker design.

1. **The MV3 background is a non-persistent event page.** Firefox MV3 doesn't use service workers. It uses `background.scripts`, which Firefox **suspends when idle** (about 30s) and wakes on events.
   - Every listener must be registered **synchronously at the top level** of `background/index.ts`, or it won't wake the background.
   - **In-memory variables are lost** on suspension, so tracker state is kept in `storage.session` / `storage.local` (`state.ts`).
2. **Event order isn't guaranteed.** For a middle-click, `tabs.onCreated`, `onCreatedNavigationTarget`, and the new tab's `onCommitted` can arrive in different orders, and handlers are async. So we:
   - store "pending opener" info keyed by the new tab ID, consumed by the first navigation in that tab.
   - run all handlers through **one serial queue** (`queue.ts`), so two events can't both decide "this node doesn't exist yet" and create duplicates.
3. **`transitionType` / `transitionQualifiers` are hints, not truth.** Firefox fills them in less completely than Chromium, and values like `forward_back` or `client_redirect` may be missing. The tracker uses them when present but never depends on them.
4. **`openerTabId` is often missing.** It isn't set for tabs opened with Ctrl+T, from bookmarks, from external apps, or from session restore. Those navigations become new **roots** in the graph, which is honest and understandable.
5. **Pages open before Start have no history.** We know nothing about how they were reached. On Start we seed the **active tab's current page** as the first node. Other already-open tabs join the graph only when you navigate in them; that first navigation becomes a root, and later navigations link to it.
6. **Client-side redirect pages create noise.** Server redirects collapse into one commit, but JS/meta redirectors (`google.com/url?…`, `t.co`, `out.reddit.com`, `l.facebook.com`, `youtube.com/redirect`) commit briefly. `utils/url.ts` keeps a small redirector list and ignores those URLs.
7. **SPAs.** Without `onHistoryStateUpdated`, YouTube video-to-video browsing would be invisible. Titles also lag behind the URL change, so a new node's title starts empty (the UI shows the domain) and is filled in from `tabs.onUpdated` once the tab's URL matches the node. SPAs also produce "passive" URL changes (YouTube autoplay, Reddit modals). That's accepted in the MVP.
8. **Private windows.** Extensions don't run there unless the user allows it. We also check `tab.incognito` and **never record private tabs**, even if allowed.
9. **Internal pages.** We only track `http:` and `https:`. That excludes `about:` (including `about:reader`, `about:neterror`), `moz-extension:`, `chrome:`, `chrome-extension:`, `view-source:`, `file:`, and `data:`. An allowlist is safer than a denylist.
10. **Favicons.**
    - `favIconUrl` can be a remote URL, a `data:` URI, or an internal `chrome://…` icon that extension pages can't load.
    - Internal icons fall back to a monochrome letter glyph.
    - A remote favicon `<img>` on the map page makes a request to a site you already visited. We use `referrerpolicy="no-referrer"` and document this. A "don't load remote favicons" option is a post-MVP idea.
11. **Temporary add-on data.**
    - A fixed `gecko.id` keeps the extension's internal origin stable, so IndexedDB data survives reloads.
    - Temporary add-ons disappear when Firefox restarts. For persistent test data across restarts, use `web-ext run` with a dedicated profile (`--firefox-profile` + `--keep-profile-changes`).
    - Extension IndexedDB is deleted on uninstall.
12. **Browser restart during an active session.** Tab IDs change after a restart. On `runtime.onStartup` we keep the session active (`storage.local`) but discard the tab → node map (`storage.session` is empty anyway). Navigation after the restart starts new roots.
13. **No `idle` handling in the MVP.** If you leave a tab in front and walk away, its active time keeps counting. We could add `browser.idle` later.

---

## 6. Data model

`src/shared/types/session.ts` (planned shape):

```ts
export type Relationship =
  | 'same_tab_navigation'
  | 'opened_new_tab'
  | 'search_result'
  | 'unknown';

export interface SessionMeta {
  id: string;              // crypto.randomUUID()
  name: string;            // "Minecraft copper farm" or "September 14 — 42 pages"
  startedAt: number;       // epoch ms
  endedAt: number | null;  // null while active
  nodeCount: number;
  edgeCount: number;
  rootQuery?: string;      // first detected search query, used for naming
}

export interface PageNode {
  id: string;
  sessionId: string;
  url: string;             // normalized (no #fragment)
  title: string;           // '' until known; UI falls back to domain
  domain: string;          // "www." stripped
  favicon?: string;
  visitedAt: number;       // first visit
  lastVisitedAt: number;
  visitCount: number;
  activeMs: number;        // time spent as the focused tab
  tabId: number;           // tab of first visit
  searchQuery?: string;    // set if this node is a search results page
}

export interface PageEdge {
  id: string;
  sessionId: string;
  sourceNodeId: string;
  targetNodeId: string;
  relationship: Relationship;
  openedIn: 'same_tab' | 'new_tab';  // kept separately so search_result doesn't lose it
  createdAt: number;
  transitionType?: string;           // raw hint from webNavigation, if provided
}

export interface Session extends SessionMeta {
  nodes: PageNode[];
  edges: PageEdge[];
}
```

### Key decisions

- **One node per normalized URL per session**, not one per visit. If you return to a page, it gets `visitCount++` and doesn't get a duplicate node. This keeps the graph readable and makes loops visible as edges back to an existing node.
- The **previous page** is recorded as the edge's `sourceNodeId`. **Same tab vs new tab** is recorded as `openedIn`.
- **URL normalization:** strip the `#fragment`, lowercase the host, and remove `utm_*`, `fbclid`, and `gclid`. The rest of the query string is kept, because `?v=` on YouTube and `?q=` on Google matter.

### IndexedDB schema (`rabbit-hole`, version 1)

| Store | Key | Indexes |
|---|---|---|
| `sessions` | `id` | `startedAt` |
| `nodes` | `id` | `sessionId`, `[sessionId, url]` (unique) |
| `edges` | `id` | `sessionId`, `[sessionId, sourceNodeId, targetNodeId]` (unique) |

Nodes and edges are in separate stores, so each navigation is a small insert instead of rewriting the whole session. The unique compound indexes are a second line of defense against duplicates. `Session` is assembled on read.

**Single writer:** only the background writes. The popup and map read IndexedDB directly and listen for a `session-updated` runtime message to refresh.

---

## 7. Tracking algorithm

### Tracker state (`state.ts`, in `storage.session`)

```ts
{
  tabs: Record<tabId, { nodeId: string; url: string }>,   // what each tab is showing now
  pendingOpeners: Record<tabId, { sourceNodeId: string }>, // new tab → page that opened it
  focus: { nodeId: string; since: number } | null          // for active-time accounting
}
```

`activeSessionId` lives in `storage.local`.

### Event handling (all run through the serial queue)

**`tabs.onCreated(tab)`**
If `tab.openerTabId` is set and that tab is tracked, set `pendingOpeners[tab.id]` to the opener tab's current node.

**`webNavigation.onCreatedNavigationTarget({ sourceTabId, tabId })`**
Same as above. This signal is more precise, so it overwrites.

**`webNavigation.onCommitted` / `onHistoryStateUpdated` (`frameId === 0` only)**
Call `handleNavigation(tabId, url, details)`:

1. Ignore the event if any of these is true:
   - no active session
   - the URL isn't `http(s)`
   - the tab is incognito
   - the URL is a known redirector
2. `norm = normalizeUrl(url)`
3. `prev = tabs[tabId]`. If `prev?.url === norm`, stop. This covers reloads, repeat events, and fragment-only changes.
4. `existing = node with (sessionId, norm)`
   - If it exists: `visitCount++` and update `lastVisitedAt`.
   - Otherwise: create the node. Title comes from `tab.title` for full loads, or stays empty for SPA changes (filled in later). Domain comes from the URL, and the favicon from `tab.favIconUrl`. If it's a search results page, set `searchQuery`.
5. Pick the source:
   - If `prev` exists, the source is `prev.nodeId`, with `openedIn = 'same_tab'`.
   - Otherwise, if `pendingOpeners[tabId]` exists, the source is that node, with `openedIn = 'new_tab'` (then consume the pending entry).
   - Otherwise there's no source, and the node is a root.
6. Pick the relationship:
   - If the source node has a `searchQuery`, it's `search_result`.
   - Otherwise, if `openedIn = 'new_tab'`, it's `opened_new_tab`.
   - Otherwise, if `openedIn = 'same_tab'`, it's `same_tab_navigation`.
   - Otherwise it's `unknown`.
7. Skip the edge if any of these is true:
   - there's no source
   - source === target
   - the edge source→target already exists
   - it looks like **back/forward**: `transitionQualifiers` includes `forward_back`, **or** the target already existed and the edge target→source exists (you're going back to the parent)
8. `tabs[tabId] = { nodeId, url: norm }`. If this tab is focused, switch active-time focus to the new node.
9. Update the session's `nodeCount`/`edgeCount`, and set `rootQuery` if it isn't set yet. Broadcast `session-updated`.

**`tabs.onUpdated(tabId, { title, favIconUrl })`**
If the tab is tracked and `normalizeUrl(tab.url) === tabs[tabId].url`, patch that node's title and favicon.

**`tabs.onActivated`, `windows.onFocusChanged`**
Add `now - focus.since` to the previously focused node's `activeMs`, then set focus to the newly active tab's node. When the browser window loses focus (`WINDOW_ID_NONE`), focus becomes `null`.

**`tabs.onRemoved`**
Flush active time if this tab had focus, then delete `tabs[tabId]` and `pendingOpeners[tabId]`.

### Session lifecycle

- **Start:**
  1. Create a `SessionMeta` and set `activeSessionId`.
  2. Clear tracker state.
  3. Seed a root node from the active tab if it's trackable, and set focus to it.
- **Stop:**
  1. Flush active time and set `endedAt`.
  2. Generate the final name.
  3. Clear `activeSessionId` and tracker state.
  4. Broadcast.

### Search-result detection (`utils/search.ts`)

| Engine | Pattern | Query param |
|---|---|---|
| Google | `google.<tld>/search` | `q` |
| Bing | `bing.com/search` | `q` |
| DuckDuckGo | `duckduckgo.com/` (and `html.` / `lite.`) | `q` |
| Brave | `search.brave.com/search` | `q` |
| Ecosia | `ecosia.org/search` | `q` |
| YouTube | `youtube.com/results` | `search_query` |

This is easy to extend later.

### Session naming (`utils/naming.ts`)

1. The first detected search query, e.g. "Minecraft copper farm"
2. Otherwise, the root page's title (truncated to about 48 characters)
3. Otherwise, the root domain
4. Otherwise, the date only

The popup and map show the name plus a subtitle like "September 14 — 42 pages". If there's no better name, the subtitle format becomes the name.

---

## 8. UI plan

### Visual tokens (`styles/tokens.scss`)

| Token | Value | Use |
|---|---|---|
| `--bg` | `#0b0b0d` | Canvas / page background |
| `--surface` | `#141417` | Nodes, panels |
| `--surface-raised` | `#1b1b1f` | Hover, inputs |
| `--border` | `rgba(255,255,255,0.08)` | Hairline borders |
| `--border-strong` | `rgba(255,255,255,0.16)` | Selected / hover |
| `--text` | `#ece9e4` | Primary text (off-white) |
| `--text-muted` | `#8b8b93` | Domains, timestamps |
| `--text-faint` | `#55555c` | Hints, grid dots |
| `--edge` | `rgba(255,255,255,0.18)` | Graph connections |
| `--glow` | `0 0 0 1px rgba(236,233,228,.35), 0 0 18px rgba(236,233,228,.12)` | Active node |
| `--recording` | `#d9534f` (muted red) | The *only* accent: a small "recording" dot |

- **Fonts:** the system UI stack for text and the system monospace stack (`ui-monospace, "Cascadia Mono", Consolas, monospace`) for URLs. **No web fonts**, because remote fonts would be a network request.
- **Color:** favicons provide nearly all the color.
- **Avoid:** gradients, big cards, stat widgets, neon.

### Popup (about 320px wide)

```
┌──────────────────────────────┐
│ ◉ Rabbit Hole                │  wordmark
│                              │
│ ● Recording · 12 pages       │  or "○ Not tracking"
│ [ Stop session ]  [ Map ]    │  or [ Start session ]
│                              │
│ RECENT                       │
│ Minecraft copper farm        │
│ Sep 14 · 42 pages        →   │
│ lightning rod physics        │
│ Sep 12 · 17 pages        →   │
│                              │
│ Your browsing sessions stay  │
│ on this device.              │
└──────────────────────────────┘
```

- **Open Current Map** opens the active session, or the most recent one if nothing is active. It opens `map.html?session=<id>` in a new tab.
- **Recent sessions:** the last 5. Clicking one opens its map.

### Map page (full tab)

- **Canvas:** fills the viewport with a very faint dot grid and lots of empty space around the graph.
- **Top-left (small, unobtrusive):** wordmark, session name, date · page count, and a session switcher dropdown. Shows a "Recording" dot if the session is live.
- **Bottom-left:** React Flow controls (zoom in/out, fit view), restyled to match.
- **Node (`PageNode.tsx`, about 220×52px):**
  - Favicon (16px) · title (1 line, ellipsis) · domain (muted, small) · optional time.
  - Selected nodes get `--border-strong`. The page currently open in the focused tab (live sessions only) gets `--glow`.
- **Edges:** 1px smooth-step curves, left to right.
  - `same_tab_navigation`: solid
  - `opened_new_tab`: dashed
  - `search_result`: solid, slightly brighter
  - No labels by default.
- **Inspector (`Inspector.tsx`):** a 320px panel that slides in from the right when you click a node. It shows:
  - favicon + title
  - full URL (mono, selectable)
  - domain
  - first visited time and visit count
  - active time
  - "arrived from" (the parent's title)
  - buttons: **Open page** (`tabs.create`) and **Copy URL**
  - Esc or clicking the canvas closes it.
- **Layout (`layout.ts`):** dagre with `rankdir: 'LR'`, `nodesep` about 24, `ranksep` about 80. Children are ordered by `visitedAt`, so the graph reads left-to-right in time. `fitView` runs on load. Dragged positions are kept only in memory for the MVP.
- **Live sessions:** on `session-updated`, re-read the session and re-run the layout. Nodes the user has already dragged keep their positions.
- **Empty state:** "No pages yet. Start browsing — Rabbit Hole is watching this session." plus the privacy line.

---

## 9. Privacy guarantees

- No backend, no accounts, no network requests made by the extension, no analytics, no telemetry, no remote fonts or CDNs.
- No host permissions and no content scripts. The extension can't read page contents.
- Private-window tabs are never recorded.
- All data lives in the extension's IndexedDB on this device and is removed on uninstall.
- Only caveat: remote favicons are displayed via `<img>` (see §5.10).
- "Your browsing sessions stay on this device." appears in the popup, on the map empty state, and in the manifest description.
- `data_collection_permissions: none` is declared in the Firefox manifest.

---

## 10. Milestones

Each milestone ends with something you can see working in Firefox. **Don't move to the next one until the current one passes.**

### M1: Project setup

- `npm init`. Install the runtime dependencies and the dev dependencies: `vite`, `@vitejs/plugin-react`, `typescript`, `sass-embedded`, `@types/react`, `@types/react-dom`, `@types/webextension-polyfill`, `web-ext`.
- Add `tsconfig.json`, `vite.config.ts` (inputs `popup.html` + `map.html`, `outDir: dist`), `vite.background.config.ts` (lib mode, IIFE, `emptyOutDir: false`), and `scripts/build-manifest.mjs`.
- npm scripts:
  - `build`: pages, then background, then manifest
  - `watch`
  - `dev:firefox`: `web-ext run --source-dir dist`
  - `typecheck`
- Placeholder icons.
- **Test:** `npm run build` produces `dist/manifest.json`, `dist/background.js`, `dist/popup.html`, and `dist/map.html`.

### M2: Manifest + extension loads

- `manifest/base.json` and `manifest/firefox.json`, merged into `dist/manifest.json`.
- `background/index.ts` logs `"Rabbit Hole background started"`.
- A "Hello Rabbit Hole" popup.
- **Test in Firefox:**
  1. Open `about:debugging#/runtime/this-firefox` and click **Load Temporary Add-on…**, then select `dist\manifest.json`.
  2. The toolbar icon shows the popup.
  3. Click **Inspect** on the extension and check the console for the log line.
  4. Alternative: run `npm run dev:firefox`.

### M3: Smallest background tracker (logging only)

- `utils/url.ts` (`isTrackable`, `normalizeUrl`, `getDomain`).
- Register `onCommitted`, `onHistoryStateUpdated`, `onCreatedNavigationTarget`, `tabs.onCreated`, and `tabs.onUpdated`.
- Log structured lines only (`[nav] tab=12 same_tab https://…`). No storage yet.
- **Test:** in the Inspect console, browse Google → a result → a YouTube video → the next video. Each page logs exactly once. `about:` pages log nothing.

### M4: Popup Start/Stop

- `shared/messages.ts` and `background/messages.ts`: `start-session`, `stop-session`, `get-status`.
- `activeSessionId` in `storage.local`. The tracker ignores events when no session is active.
- The popup shows the status and the Start/Stop button.
- **Test:**
  1. Browse without starting a session. Nothing is logged.
  2. Click Start, then browse. Events are logged.
  3. Click Stop. Logging stops.
  4. Close and reopen the popup. The status is still correct.

### M5: Relationship detection, verified

- `state.ts` (storage.session), `queue.ts`, `tabs.ts`, `tracking.ts`, `utils/search.ts`.
- The full algorithm from §7, **still keeping nodes/edges in tracker state and logging them** (no IndexedDB yet).
- A debug message `dump-session` prints the current nodes/edges as a table.
- **Test script (run manually, compare the output):**
  1. Start a session on a new tab. Search Google for "Minecraft copper farm".
  2. Click a YouTube result in the same tab. Expect `search_result`, `same_tab`.
  3. Middle-click a Reddit link from the Google results. Expect `search_result`, `new_tab`.
  4. On Reddit, click a Minecraft Wiki link. Expect `same_tab_navigation`.
  5. Go back, then forward. Expect no new nodes or edges.
  6. Reload. Expect nothing new.
  7. Wait about 60s so the background suspends, then click another link. The edge must still link correctly.
  8. Ctrl+T and type a URL. Expect a new root.
  9. Stop. Then run `dump-session` and check there are no duplicates and nothing from `about:`.

### M6: Persistence

- `shared/db.ts` (open/upgrade, `getSession`, `listSessions`) and `background/storage.ts` (writes).
- The tracker writes to IndexedDB instead of memory. Session naming on stop.
- The popup shows recent sessions.
- **Test:**
  1. Record a session and stop it. It appears in Recent.
  2. Reload the extension in `about:debugging`. It's still there.
  3. Check the data in Inspect → Storage → Indexed DB.

### M7: Basic map page

- `map.html` + `MapPage.tsx`. Reads `?session=`, loads the session, and renders React Flow with the default nodes in a simple grid.
- The popup's **Open Current Map** button and the recent-session links open the map.
- **Test:** open a saved session. All pages and edges appear. Zoom, pan, and drag work.

### M8: Layout + interaction

- `layout.ts` (dagre LR), the custom `PageNode`, `Inspector`, Open page, Copy URL, and fit view.
- **Test:**
  1. The M5 journey renders as a readable left-to-right tree.
  2. Clicking a node shows the correct URL, title, and time.
  3. **Open page** opens a new tab.
  4. **Fit view** frames everything.

### M9: Visual design + live updates

- Tokens, restyled controls, edge styles per relationship, the active-node glow, the empty state, and the map header with the session switcher.
- Live refresh while recording.
- **Test:**
  1. Open the map for an active session and browse in another window. Nodes appear live.
  2. The focused page glows.
  3. The UI matches §8, with no bright colors besides favicons.

### M10: Hardening

- Redirector filter list, SPA title back-fill, active-time accuracy, `runtime.onStartup` handling, favicon fallback glyph.
- Sanity-check a large session (about 300 nodes).
- **Test:**
  1. Clicking a `t.co` / Google redirect link produces no intermediate node.
  2. YouTube nodes show the correct video titles.
  3. Restart Firefox during a session (using the `web-ext` profile). The session is still active.

---

## 11. Definition of done (MVP)

- [ ] Loads in Firefox from `dist/` with no console errors
- [ ] Start/Stop works, and state survives popup close and background suspension
- [ ] The M5 test journey produces the expected nodes, edges, and relationships with no duplicates
- [ ] Sessions persist and are listed in the popup with sensible names
- [ ] The map renders a clean left-to-right graph; zoom, pan, drag, select, inspect, reopen, and fit all work
- [ ] The visual style matches §8
- [ ] No network requests from extension code (verify in the Network tab while using the popup and map)

---

## 12. After the MVP (not now)

These are listed in rough order.

1. Delete a session (and "delete all") — a privacy essential, so it comes first.
2. Rename a session.
3. Export/import a session as JSON (still local).
4. Setting: don't load remote favicons.
5. `browser.idle`-aware active time.
6. Collapse noisy SPA chains (e.g. YouTube autoplay) into one node group.
7. Chrome/Chromium build:
   - add `manifest/chrome.json` with `service_worker`
   - make `storage.session` access level explicit
   - optionally add the `favicon` permission and the `_favicon/` URL for local favicons
   - transition types are more reliable there
8. Search/filter nodes on the map; highlight the path from root to the selected node.
9. Persist dragged node positions.

---

## 13. Open questions

- **Tabs already open when a session starts.** Seed only the active tab (current plan), or all tabs in the window as roots? Current choice: active tab only, to avoid clutter.
- **Revisits across tabs.** The same URL in two tabs is one node (current plan). Is that what you want, or do you want per-tab duplicates?
- **Auto-stop.** Should a session stop automatically after N hours of inactivity? Not in the MVP.

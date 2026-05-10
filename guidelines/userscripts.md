# Userscript Development Guidelines

## Role

You are an expert in building userscripts that run in engines like Tampermonkey and Violentmonkey. To reduce boilerplate and get a modern DX, you prefer using **vite-plugin-monkey**, which lets you write userscripts with full Vite tooling — HMR, TypeScript, ESM imports, and a proper build pipeline.

---

## vite-plugin-monkey Intro

Bootstrap with:

```shell
bun create monkey
```

Pick `empty` and `ts` as the template. The entry point is a single `main.ts` that runs in the page context. Unlike a browser extension, there is no popup, sidebar, background script, or manifest — just one script injected into matching pages.

### Default Template Structure (empty - ts)

```
.
├── dist
│   └── vite-empty-ts-starter.user.js
├── package.json
├── src
│   ├── main.ts
│   ├── vite-env.d.ts
├── tsconfig.json
└── vite.config.ts
```

## Non-shell guidance. 

Most of the times, the user will execute this command to set up a new userscript project, so you don't need to rewrite the boilerplate again. You just need to add your own code for the src/ 

Here is the full template source code for reference so you understand the full picture. 

## Full Template Source Code

```txt
monkey-template-empty
├── dist
│   └── vite-empty-ts-starter.user.js
├── package.json
├── src
│   ├── main.ts
│   └── vite-env.d.ts
├── tsconfig.json
└── vite.config.ts

```

`dist/vite-empty-ts-starter.user.js`:

```js
// ==UserScript==
// @name       vite-empty-ts-starter
// @namespace  npm/vite-plugin-monkey
// @version    0.0.0
// @icon       https://vitejs.dev/logo.svg
// @match      https://www.google.com/
// ==/UserScript==

(function () {
	'use strict';

	console.log("hello world");

})();
```

`package.json`:

```json
{
  "name": "monkey-template-empty",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview"
  },
  "devDependencies": {
    "typescript": "^6.0.3",
    "vite": "^8.0.11",
    "vite-plugin-monkey": "^8.0.0"
  }
}
```

`src/main.ts`:

```ts
// @ts-ignore isolatedModules
console.log('hello world');

```

`src/vite-env.d.ts`:

```ts
/// <reference types="vite/client" />
/// <reference types="vite-plugin-monkey/client" />
//// <reference types="vite-plugin-monkey/global" />
/// <reference types="vite-plugin-monkey/style" />

```

`tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "useDefineForClassFields": true,
    "module": "ESNext",
    "lib": ["ESNext", "DOM"],
    "moduleResolution": "Node",
    "strict": true,
    "sourceMap": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "noEmit": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "skipLibCheck": true
  },
  "include": ["src"]
}

```

`vite.config.ts`:

```ts
import { defineConfig } from 'vite';
import monkey from 'vite-plugin-monkey';

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [
    monkey({
      entry: 'src/main.ts',
      userscript: {
        icon: 'https://vitejs.dev/logo.svg',
        namespace: 'npm/vite-plugin-monkey',
        match: ['https://www.google.com/'],
      },
    }),
  ],
});

```

---

## Userscript vs Extension — Know the Format

Before writing a line of code, ask: **is a userscript actually the right format?**

A userscript is intentionally limited. It runs as a guest in someone else's page, with no manifest, no browser API access, no persistent UI surfaces, and no cross-tab coordination. That limitation is also its strength — frictionless distribution, no store review, one URL to share.

A **browser extension** (e.g. built with WXT) unlocks a much broader surface: popups, sidebars, background workers, `chrome.storage`, network request interception, cross-tab messaging, and the full Web Extension API. The key insight is that you probably **don't need most of that** for the majority of page-augmentation tasks — and reaching for an extension when a userscript suffices is genuine overengineering.

| Situation | Format |
|---|---|
| Inject a button, reformat content, automate interactions | Userscript |
| Small quality-of-life tweaks to a specific site | Userscript |
| Need a popup, sidebar, or persistent background logic | Extension |
| Need network request interception (`webRequest`) | Extension |
| Need cross-tab coordination or `chrome.storage` | Extension |
| UI has grown so complex it needs a component framework | Probably Extension |

That last row is the important one. If your userscript has grown complex enough that you're tempted to pull in Svelte, that's usually a signal the problem has outgrown the format — not a reason to add a framework. At that point you're fighting the userscript format to get extension-like behavior. Take the hint and build an extension instead.

**Userscripts should stay lean: TypeScript, manual DOM, and GM_* APIs. That's the stack.**

---

The SKILL.md is read-only, so here's the revised **Module Structure** section ready to paste in. The changes: tightened the folder-rule prose, added an explicit `lib/` stance, strengthened the flat-first advice with a before/after, and fixed the `button.ts` example confusion.

---

## Module Structure

No God Files. Even though a userscript is a single bundle, the source should be modular. `main.ts` should read like a true composition root — the one place where the dependency graph is assembled and visible, not an implementation dump.

### The five layers

A userscript has five legitimate concerns. Each gets a folder when it has two or more files. Each folder has a strict contract — a file belongs in exactly one of them.

**`ui/`** — DOM creation and state rendering only. Files here create elements, mutate their appearance, and listen for user input. They know nothing about what happens when the user acts — they only know how to render a given state and surface a callback. This includes injection logic that places elements into the host page's DOM — even if it's navigating YouTube's existing structure, it's still a UI concern.

**`actions/`** — sequenced user-facing operations. One file per thing *a user can explicitly trigger* — a click, a keyboard shortcut, a context menu item. Each file sequences calls *across concerns*: fetch transcript, then write to clipboard, then persist state. The sequence is the user's intent expressed in code — each step is a different system, connected because that's what the goal requires. No DOM, no GM_* calls, no rendering. If you removed the UI entirely, these files would still make sense.

**`services/`** — reliable data access for a single concern. Files here sequence cache checks, retries, and backoff *around a single adapter* to make data retrieval dependable. The orchestration is a *mechanism*, not a user goal — there is no user intent to express, only a data reliability problem to solve. Unlike `actions/`, services are triggered by the *system* — a page load, a polling interval, an observer callback — not a user gesture. Unlike `adapters/`, they contain real logic that survives even if you swap the underlying transport. A file belongs here when the question is "how do I reliably get this data?" not "what does the user want to do?"

**`adapters/`** — wrappers around external APIs and browser capabilities. GM_* calls, network requests, storage reads and writes. Each file translates between your codebase's language and some external system's language. No business logic — only translation and error classification that exists *because the external system requires it*. If you could swap the underlying API (e.g. `GM_setValue` → `localStorage`) by only touching this file, it's an adapter. If logic would survive that swap, it belongs elsewhere.

**`utils/`** — generic, domain-free helpers. Pure functions: DOM shortcuts, string formatters, icon factories. No GM_* calls, no business logic, no side effects. If a file would make sense in a completely different userscript, it belongs here.

### The dependency rule

Dependencies only flow downward:

```
ui/ → actions/ → services/ → adapters/
               ↘           ↗
                 adapters/
                          ↘
                           utils/  ← available to all layers
```

`ui/` may call `actions/`. `actions/` may call `services/` or `adapters/` directly. `services/` may call `adapters/`. Nothing calls upward. `utils/` is available to all layers.

Violating this — an adapter importing from services, a UI file importing directly from adapters — is the signal that something is in the wrong layer. The direction of the import is the smell, not the content.

### `main.ts` is the composition root

`main.ts` has one job: wire the layers together and start the script. It is the only file allowed to reach across all layers. It must make the dependency graph visible — if the ordering of initialisation matters, that ordering must be explicit here, not implicit in import order.

```ts
// main.ts — you can read the whole architecture from this file
import { initStorage } from './adapters/storage';
import { injectCreationDate } from './ui/injector';
import { getCreationDate } from './services/githubRepo';

async function main(): Promise<void> {
  await initStorage();                               // adapters boot first
  setInterval(() => {                                // system loop, not user action
    const date = await getCreationDate(user, repo);  // service call
    injectCreationDate(date);                        // ui call
  }, POLL_INTERVAL_MS);
}

void main();
```

If `main.ts` grows beyond wiring and starting, something is wrong.

### A realistic layout

```
src/
├── main.ts
├── ui/
│   ├── button.ts
│   ├── button.css
│   └── injector.ts      ← places UI into the host page's DOM
├── actions/
│   └── copyTranscript.ts
├── services/
│   └── githubRepo.ts    ← cache check → backoff check → fetch
├── adapters/
│   ├── storage.ts
│   ├── cache.ts
│   ├── transcript.ts
│   └── clipboard.ts
└── utils/
    ├── dom.ts
    └── icons.ts
```

Not every script will use all five layers. A passive augmentation script (observe, fetch, inject) may have `services/`, `adapters/`, `ui/`, and `utils/` but no `actions/` at all — because nothing is user-triggered. A purely interactive script might skip `services/` entirely. Folders only exist when they have two or more files. A single-file layer stays flat in `src/`.

### On naming

Folder names carry architectural claims. Hold them to it.

- `ui/` → "these render things" ✓
- `actions/` → "these are things a user explicitly triggers" ✓
- `services/` → "these orchestrate data access with cross-cutting concerns" ✓
- `adapters/` → "these translate to external systems" ✓
- `utils/` → "these are generic helpers" ✓
- `core/` → "these are... important?" ✗ means nothing
- `lib/` → implies publishable/cross-project reuse that doesn't exist here ✗
- `modules/` → describes every JS file in existence ✗

The distinction between `actions/` and `services/` is not ambiguous if you apply the trigger test: **did a user gesture cause this to run?** If yes, it's an action. If no, it's a service. That question has a clear answer for every file.

If you find yourself debating which folder a file belongs in, re-read the trigger test. If the test doesn't resolve it, the file is probably doing two things.


## UI: Just Use the DOM

There are no native UI surfaces in a userscript. Everything you render is injected into the host page. Manual DOM is the right default, but you must protect your elements from colliding with the host page.

### The Module-Scoped Dynamic ID
For elements you inject and need to track (e.g., to ensure you don't inject them twice), **generate a stable, random ID at module scope**. 

```ts
// ui/injector.ts
// Generated once when the module loads, stable for the entire session.
const NS = `uc_${Math.random().toString(36).slice(2, 10)}`;

export function isAlreadyInjected() {
    // ⚠️ Crucial: Use getElementById, not querySelector. 
    // getElementById is an O(1) hash map lookup and skips the browser's CSS parser.
    return document.getElementById(NS) !== null; 
}

export function injectElement() {
    const btn = document.createElement('button');
    btn.id = NS; // Bind the generated ID
    btn.className = 'myscript-btn';
    btn.textContent = 'Do Thing';
    document.querySelector('#target')?.appendChild(btn);
}
```

**Why this is mandatory:**
1. **Zero Collisions:** A random string like `uc_x7k2m9p3` guarantees you will never accidentally conflict with a class or ID the host site deploys in the future.
2. **Maximum Polling Performance:** As discussed below, modern userscripts often poll the DOM rapidly. Checking for existence via `getElementById(NS)` costs practically zero CPU.

---

## GM_* API Usage

Always import GM APIs via ESM from `vite-plugin-monkey/dist/client`. Never rely on global variables:

```ts
import { GM_setValue, GM_getValue, GM_addStyle } from '$';
```

Wrap GM_* calls in a `services/` layer rather than scattering them through logic. This keeps the GM dependency contained in one place and your other modules clean.

---

## CSS Injection

Inject styles with `GM_addStyle`. Use specific, namespaced class names (e.g. `myscript-panel`, `myscript-btn`) to avoid conflicts with the host page. You rarely need Shadow DOM unless you're embedding something that must be fully visually isolated.

---

## DOM Interaction & Timing (The SPA Polling Architecture)

Modern web apps are Single Page Applications (SPAs) built with React, Vue, or Turbo. They do not trigger full page reloads on navigation; they just rip out and replace chunks of the DOM. 

### The `MutationObserver` Trap
Tutorials often preach using `MutationObserver` to watch the DOM. **Avoid this for SPAs.** If you attach an observer to a specific container, it gets destroyed when the framework replaces that container. If you attach it to `document.body` with `{ subtree: true }`, your script will trigger thousands of times per second during a page transition, tanking performance and requiring complex debouncing logic.

### The `setInterval` Solution
The most robust, resilient, and performant approach for injecting UI into an SPA is **Continuous Polling via `setInterval`** (e.g., every 300ms), combined with strict execution guards. Polling survives framework re-renders, catches SPA navigations natively, and requires zero cleanup. 

To prevent your interval from becoming a "CPU/Network Hammer," you **must** structure your main loop with four strict guards:

1. **The Fast-Path Guard:** Use your dynamic ID (`document.getElementById(NS)`) to check if your element exists. If it does, exit immediately.
2. **The Path Guard:** Use string matching on `window.location.pathname` to ensure you are on the correct page *before* doing any heavy DOM traversals (`querySelectorAll`).
3. **The Network Guard:** Cache external data/API responses (using memory or `GM_setValue`) so you only fetch data once per context, not every 300ms.
4. **The Concurrency Guard:** Use an `isRunning` boolean lock to prevent overlapping async executions if the interval fires while an API call is still pending.

**The Golden Architecture:**

```ts
// main.ts
let isRunning = false;

async function main() {
    // 1. O(1) Fast Paths
    if (window.location.hostname !== 'targetsite.com') return;
    if (isAlreadyInjected()) return; // document.getElementById(NS)

    // 2. CPU Guard (Path Checking)
    // E.g., Only run on /users/profile pages
    const pathParts = window.location.pathname.split('/');
    if (pathParts[1] !== 'users' || pathParts.length > 3) return; 

    // 3. Concurrency Guard
    if (isRunning) return;
    isRunning = true;

    try {
        const userId = pathParts[2];
        
        // 4. Network Guard (This adapter should check local cache before fetching)
        const userData = await getUserDataAdapter(userId);
        
        // 5. Finally, run the heavy DOM traversal and inject
        // If the React UI is still rendering and the target container is missing,
        // this fails safely. The 300ms loop will just try again.
        injectUI(userData);

    } finally {
        isRunning = false;
    }
}

// Start the continuous, self-healing loop
setInterval(main, 300);
```

---

## TypeScript

Full TypeScript, always. vite-plugin-monkey ships with GM API type hints out of the box when you import from `$`. No `any`, strict null handling, all exported functions annotated with return types.

---

## Dependencies

Keep them minimal. You're injecting into someone else's page — every extra KB is a liability. If a native Web API or a short utility function covers it, use that. Reserve external dependencies for things where correctness really matters.

---

## Build & Distribution

- Development: `bun dev` — auto-opens the `.user.js` install URL on changes, with HMR.
- Production: `bun build` — outputs a single `.user.js` with the userscript comment header injected automatically.
- Keep `minify: false` if publishing to Greasy Fork (their rules disallow minified code).
- Use `externalGlobals` for any large dependency to pull it via CDN `@require` instead of bundling it.

---

## Keep It Simple
 
Before adding any non-feature code, ask one question: **does this solve a real problem that actually occurs, or am I just imagining it?**
 
If you can't point to a concrete failure or waste, don't add the code.
 
---
 
**Scope guards** — bail out early when there's nothing to do. A page check at the top of `main.ts` is always worth it. A debounce on a MutationObserver is worth it (it fires dozens of times per SPA transition). Both prevent real, unnecessary work.
 
**Correctness guards** — only for failures that genuinely happen. Re-entrant MutationObserver callbacks are real (your own DOM writes trigger it). Race conditions between two async calls are real. Theoretical concurrent mutation by a random other script is not real — skip it.
 
**Performance optimisations** — skip them unless you can name the specific page, the specific bottleneck, and why the simple version is actually too slow. `querySelectorAll` on 5 nodes is not a bottleneck.
 
**Teardown** — only for resources the browser holds independently: `observer.disconnect()`, `clearInterval`, `socket.close()`. Nulling out JS variables on `pagehide` is pointless — the browser destroys the whole context anyway.
 
---
 
**On tooling choices:** prefer the simpler API when it's sufficient.
 
`setInterval` polling every 300ms is often better than a MutationObserver — it's two lines, trivially debuggable, and perfectly reliable for "wait until element appears" cases. Reach for MutationObserver only when you need to intercept changes before the user sees them, or when you're reacting to high-frequency mutations where polling would visibly lag.
 
The instinct to use the more sophisticated API is usually overengineering. Match the tool to the actual precision required.
 
---
 
**The one-line version:** write the simplest code that handles what actually happens, not the most defensive code that handles what could theoretically happen.

## Skills

Don't forget to check out Skills that you have. For userscripts, it's usually TypeScript and Userscripts guidelines.

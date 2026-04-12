# Userscript Development Guidelines

## Role

You are an expert in building userscripts that run in engines like Tampermonkey and Violentmonkey. To reduce boilerplate and get a modern DX, you prefer using **vite-plugin-monkey**, which lets you write userscripts with full Vite tooling — HMR, TypeScript, ESM imports, and a proper build pipeline.

---

## vite-plugin-monkey Intro

Bootstrap with:

```shell
bun create monkey
```

Pick `vanilla-ts` as the default. The entry point is a single `main.ts` that runs in the page context. Unlike a browser extension, there is no popup, sidebar, background script, or manifest — just one script injected into matching pages.

### Default Template Structure (vanilla-ts)

```
.
├── dist
│   └── vite-vanilla-ts-starter.user.js
├── package.json
├── src
│   ├── counter.ts
│   ├── main.ts
│   ├── style.css
│   ├── typescript.svg
│   ├── vite-env.d.ts
│   └── vite.svg
├── tsconfig.json
├── tsconfig.node.json
└── vite.config.ts
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

### The four folders

A userscript has four legitimate concerns. Each gets a folder when it has two or more files. Each folder has a strict contract — a file belongs in exactly one of them.

**`ui/`** — DOM creation and state rendering only. Files here create elements, mutate their appearance, and listen for user input. They know nothing about what happens when the user acts — they only know how to render a given state and surface a callback.

**`actions/`** — sequenced user-facing operations. One file per thing a user can trigger. Each file sequences calls to `adapters/` in the right order, handles errors, and returns a result. No DOM, no GM_* calls, no rendering. If you removed the UI entirely, these files would still make sense.

**`adapters/`** — wrappers around external APIs and browser capabilities. GM_* calls, network requests, storage reads and writes. Each file translates between your codebase's language and some external system's language. No business logic — only translation and error classification that exists because the external system requires it.

**`utils/`** — generic, domain-free helpers. Pure functions: DOM shortcuts, string formatters, icon factories. No GM_* calls, no business logic, no side effects. If a file would make sense in a completely different userscript, it belongs here.

### The dependency rule

Dependencies only flow downward:

```
ui/ → actions/ → adapters/
             ↘ utils/
```

`ui/` may call `actions/`. `actions/` may call `adapters/`. Nothing calls upward. `utils/` is available to all layers. Violating this — an adapter importing from actions, a UI file importing directly from adapters — is the signal that something is in the wrong folder.

### `main.ts` is the composition root

`main.ts` has one job: wire the layers together and start the script. It is the only place allowed to reach across all folders. It must make the dependency graph visible — if the ordering of initialisation matters, that ordering must be explicit here, not implicit in import order.

```ts
// main.ts — you can read the whole architecture from this file
import { initStorage } from './adapters/storage';
import { injectAll } from './ui/dom';

async function main(): Promise<void> {
  await initStorage();                          // adapters boot first
  setInterval(injectAll, POLL_INTERVAL_MS);     // ui starts after
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
│   └── button.css
├── actions/
│   └── copyTranscript.ts
├── adapters/
│   ├── storage.ts
│   ├── transcript.ts
│   └── clipboard.ts
└── utils/
    ├── dom.ts
    └── icons.ts
```

### On naming

Folder names carry architectural claims. Hold them to it.

- `ui/` → "these render things" ✓
- `actions/` → "these are things a user can do" ✓
- `adapters/` → "these translate to external systems" ✓
- `utils/` → "these are generic helpers" ✓
- `core/` → "these are... important?" ✗ means nothing
- `services/` → technically accurate for adapters, but blurs the line with actions — both could claim the label. The ambiguity is exactly the problem it fails to solve.

If you find yourself debating which folder a file belongs in, the folder names are not doing their job.

## UI: Just Use the DOM

There are no native UI surfaces in a userscript. Everything you render is injected into the host page. Manual DOM is the right default:

```ts
const btn = document.createElement('button');
btn.className = 'myscript-btn';
btn.textContent = 'Do Thing';
btn.onclick = handleClick;
document.querySelector('#target')?.appendChild(btn);
```

This is not a compromise — it's appropriate for the format. Keep it simple.

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

## DOM Interaction & Timing

You handle timing yourself — there's no framework magic.

- **`setInterval` polling** is underrated. For most "wait for element to appear" cases, polling every 200–500ms is simple, readable, and reliable enough.
- **MutationObserver** when you need precise interception — before a user sees something, or reacting to high-frequency DOM changes.
- **`DOMContentLoaded` / `window.onload`** for scripts that only need to run once after page load.

Match the tool to the precision required. Don't default to MutationObserver just because it sounds more thorough.

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

## Skills

Don't forget to check out Skills that you have. For userscripts, it's usually TypeScript and Userscripts guidelines.

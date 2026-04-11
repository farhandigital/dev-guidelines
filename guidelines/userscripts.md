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

## Module Structure

No God Files. Even though a userscript is a single bundle, the source should be modular. `main.ts` should read like an orchestrator, not an implementation dump.

Beyond that, **let the contents drive the structure — not a template.** There is no canonical folder layout to copy. The right structure for your project is the one that reflects what your project actually contains.

### The rule for creating a folder

A folder earns its existence when:
1. You have **two or more files** that belong together, and
2. The folder name **communicates something** the filenames alone don't

A single-file folder is almost always wrong — it's just indirection. A folder named `core/` is a particularly weak signal: it means "important" which means everything and nothing. Compare:

- `services/` → "these talk to external things" ✓ communicates something
- `utils/` → "these are generic helpers" ✓ communicates something
- `core/` → "this is... important?" ✗ adds no information

### Practical guidance

**Start flat.** If you have one standalone module — a state machine, a parser, a detector — put it as a file directly in `src/`. Don't create a folder for it.

```
src/
├── main.ts          ← orchestrator
├── button.ts        ← state machine, stands alone — no folder needed
├── services/
│   ├── storage.ts
│   ├── transcript.ts
│   └── clipboard.ts
└── utils/
    ├── dom.ts
    ├── icons.ts
    └── styles.ts
```

**Hybrid is fine.** Flat and foldered can coexist in the same `src/`. There's no rule that structure must be uniform. `services/` earns its folder (three files, all GM_* wrappers). A lone `button.ts` doesn't need one just because `services/` has one.

**Promote when ready.** If `button.ts` later spawns `button-observer.ts` and `button-registry.ts`, *then* make `button/`. Not before.

### Common categories that do earn folders

These groupings tend to make sense once you have multiple files in them:

- `services/` — GM_* wrappers, external API calls, storage abstractions (all side-effectful, all "talk to the outside world")
- `utils/` — generic helpers: DOM shortcuts, string formatters, pure functions with no business logic

---

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

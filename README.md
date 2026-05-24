# agex-studio-apps

Cross-origin sandbox host for [agex.studio](https://agex.studio)'s agent-generated apps. Lives at `apps.agex.studio`.

## What this is

A single static page (`index.html`) that the studio embeds as an iframe. The studio posts the agent's bundled app HTML to this page over `postMessage`; this page receives the HTML and replaces its document with it. The app then runs at `apps.agex.studio`'s origin — separately from the studio at `agex.studio`.

## Why a separate repo

GitHub Pages allows one custom domain per repository, so the cross-origin separation requires a second repo. That's the whole reason this exists; there's no other content here.

## Why cross-origin

The studio previously sandboxed apps with `sandbox="allow-scripts"` on a same-origin blob URL. That gave the iframe an **opaque origin**, which has several real limitations:

- Persistent permissions (camera, microphone, geolocation) can't be granted to opaque origins, so `getUserMedia` and friends fail with `SecurityError: Invalid security origin`.
- Cross-origin stylesheet introspection (`CSSStyleSheet.cssRules`) is blocked even with `crossorigin="anonymous"` on `<link>` tags.
- A long tail of services (map tile providers, font CDNs, some API gateways) filter requests by `Referer` or `Origin` and reject the blank/null values opaque origins send.

Moving the iframe to a real cross-origin domain (`apps.agex.studio`) resolves all of the above. The origin separation also provides the same isolation guarantee the sandbox attribute used to: agent code at `apps.agex.studio` cannot reach `agex.studio`'s storage, settings, or IndexedDB via same-origin policy.

## Protocol

1. The studio creates an iframe with `src="https://apps.agex.studio/"`.
2. This page loads, runs its inline script, and posts `{ type: 'agex-host-ready' }` to its parent.
3. The studio receives the ready signal, then posts `{ type: 'agex-host-init', html: '<app html>' }` to the iframe.
4. This page validates the message's origin against an allowlist (`agex.studio` for prod, `localhost:5173` for dev), then replaces its document with the received HTML via `document.open()` / `document.write()` / `document.close()`.
5. The app's injected scripts (query bridge, storage shim, control bridge — added by the studio's `buildAppHtml`) use `window.__AGEX_PARENT_ORIGIN` (set by this page) to target `postMessage`s back to `agex.studio` specifically rather than `*`.

## Local development

If you're working on the studio against a local apps origin:

```bash
# from this repo
python3 -m http.server 5174
```

Then set `VITE_APPS_ORIGIN=http://localhost:5174` in the studio's `.env.local` and run `npm run dev` as usual. The bootloader's parent-origin allowlist already includes `http://localhost:5173` (vite's default).

## Deployment

GitHub Pages deploys from `main`. No build step — `index.html` is the artifact.

## License

MIT — see [LICENSE](LICENSE).

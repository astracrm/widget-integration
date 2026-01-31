# AstraCRM Widget - Integration Guide

[Русская версия](./README_RU.md)

How to embed the AstraCRM order widget on your site. Copy-paste a snippet, tweak a few attributes, and you're done. No need to dig into source code.

What you get:
- Custom element `astra-order-widget` that just works
- Three modes: floating button, embedded block, or headless (no UI)
- Single UMD bundle from CDN, or a headless-only version if you don't need styles
- CDN: `https://cdn.astracrm.pro/widget/v1/astra-widget.umd.js`
- CDN serves the bundle **gzipped** (`Content-Encoding: gzip`) - browsers decode automatically.
- Needs: modern browser, HTTPS, and a public widget key

---

## Quick Start

Two ways to set it up:

### Option A: Pass the key as an attribute

```html
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.umd.js"></script>

<astra-order-widget
  api-key="YOUR_PUBLIC_KEY"
  mode="floating"
  position="bottom-right"
  button-text="Request service">
</astra-order-widget>
```

### Option B: Use a global variable (you can copy this from CRM UI → Settings → Widgets)

```html
<script>
  window.ASTRA_WIDGET_PUBLIC_KEY = 'YOUR_PUBLIC_KEY';
</script>
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.umd.js"></script>

<astra-order-widget mode="floating" position="bottom-right"></astra-order-widget>
```

That's it. The widget registers itself and starts working.

---

## Modes

### Floating button
A button that floats on your page and opens the form when clicked. Good for "contact us" scenarios.

Positions: `bottom-right` (default), `bottom-left`, `top-right`, `top-left`, `bottom-center`, `right-center`, `left-center`.

You can customize the button text and icon:

```html
<astra-order-widget
  api-key="..."
  mode="floating"
  position="bottom-center"
  button-text="Need help?"
  button-icon="message">
</astra-order-widget>
```

Available icons: `message` (default), `users`, `star`, `clock`, `phone`, `chat` (alias for `message-square`), `message-square`, `map-pin`, `briefcase`, `user`.

### Embedded block
Renders right where you put it, like a normal form. Takes full width of its container. You can add your own title and subtitle.

```html
<section class="contact">
  <h2>Service request</h2>
  <astra-order-widget
    api-key="..."
    mode="embedded"
    theme="light"
    widget-title="Request a service"
    widget-subtitle="Describe your issue - we'll call you back">
  </astra-order-widget>
</section>
```

### Headless mode
Widget handles validation and API calls, but you build your own UI. Perfect when you need full control over the form design.

Set `mode="headless"`, then use `__ASTRA_WIDGET_API__` on the element once it's ready. There's also a headless-only bundle (`astra-widget.headless.umd.js`) if you don't want any styles loaded.

Full headless integration guide: [HEADLESS_INTEGRATION_EN.md](./HEADLESS_INTEGRATION_EN.md)

Quick example:
```html
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.headless.umd.js"></script>
<astra-order-widget mode="headless" api-key="..."></astra-order-widget>
<script>
  const el = document.querySelector('astra-order-widget');
  el.addEventListener('widget:ready', async () => {
    const api = el.__ASTRA_WIDGET_API__;
    api.setFormData({ clientPhone: '79990000000', description: 'Need technician' });
    if (api.isValid()) {
      const res = await api.submit();
      console.log('Order created:', res.orderId);
    }
  }, { once: true });
</script>
```

---

## Attributes

| Attribute | Type/values | Default | Purpose |
|---|---|---|---|
| `api-key` | string | - | Public widget key. Or set `window.ASTRA_WIDGET_PUBLIC_KEY` before loading the script. Missing key throws “Missing widget public key”. |
| `api-url` | URL (must be full, incl. `/api/v1`) | `https://api.astracrm.pro/api/v1` | Override API base. No auto-append of `/api/v1` when you pass an attribute. |
| `mode` | `floating` · `embedded` · `headless` | `floating` | Display mode. |
| `headless` | boolean (`""`, `true`, `1`, `yes`) | false | Alternative switch for headless mode (same as `mode="headless"`). |
| `theme` | `light` · `dark` · `auto` | `light` | Color theme. `auto` follows system (prefers-color-scheme). |
| `locale` | `en` · `ru` | `en` | UI language. |
| `required-fields` | CSV (`clientPhone,description,...`) | `clientPhone,description` | Extra required fields enforced before submit. Validation still enforces `serviceId` UUID. |
| `position` | see list | `bottom-right` | Floating button position. |
| `button-text` | string | auto by `locale` | Floating button text. |
| `button-icon` | `message` · `users` · `star` · `clock` · `phone` · `chat`/`message-square` · `map-pin` · `briefcase` · `user` | `message` | Floating button icon. |
| `widget-title` | string | auto | Form title (embedded/floating welcome screen). |
| `widget-subtitle` | string | auto | Form subtitle. |
| `order-state` | `draft` · `distributing` | `draft` | Order status passed to backend. |
| `debug` | `true` · `false` | `false` | Verbose event logs + DOM events. |

Priority: explicit attributes > `window.*` globals. Backend widget settings (title/subtitle/button text/icon, mode/position, order-state, required-fields, locale) are used only as defaults; if you set an attribute or override via headless `updateConfig`, your value wins. Branding/colors and service catalog still come from backend.

Init-only (read once): `mode`, `headless`, `debug`. Reactive (watched): `api-key`, `api-url`, `locale`, `widget-title`, `widget-subtitle`, `theme`, `required-fields`, `position`, `button-text`, `button-icon`, `order-state`.

---

## Configuration

Most things work out of the box. Grab the code snippet from AstraCRM UI (Widgets → Embed → Code) and paste it. The widget connects to `https://api.astracrm.pro/api/v1` automatically.

If you set `api-url` yourself, pass the full base including `/api/v1`-it is not auto-appended for the attribute override. Always define `api-key` (or `window.ASTRA_WIDGET_PUBLIC_KEY`) before loading the script.

---

## Order State

- **`draft`** (default): Order gets saved but doesn't go to workers yet. Useful if you want to review it first.
- **`distributing`**: Order goes straight to available workers. They get notified and can pick it up immediately.

---

## Security & networking

The public key isn't secret - it's meant to be in your HTML. All requests use HTTPS, and CORS is handled automatically on our side.

### IDN domains and subdomains

- IDN domains are supported: you can use `https://терпение_и_труд.рф` and we store it internally as punycode.
- If you have many subdomains, add a single wildcard origin: `https://*.yourdomain.tld` (works for Cyrillic domains too) so CORS covers all subdomains.

---

## Localization

Set `locale="en"` or `locale="ru"` to switch the UI language. You can always override titles with `widget-title` and `widget-subtitle` attributes.

---

## Theming

Use `theme="light"`, `theme="dark"`, or `theme="auto"` (auto follows system). Want more control? Override these CSS variables:

```css
astra-order-widget {
  --ow-primary: #3b82f6;
  --ow-secondary: #64748b;
  --ow-background: #ffffff;
  --ow-text: #1f2937;
  --ow-border: #e5e7eb;
  --ow-border-radius: 12px;
  --ow-shadow-floating: 0 25px 50px -12px rgba(0,0,0,.25);
}
```

---

## Form fields & validation

What is validated client-side (hard failure on submit):
- `clientPhone`: required, regex `^7\\d{10}$` (exactly 11 digits starting with `7`).
- `description`: required, 1–1000 chars.
- `serviceId`: required, UUID-like.
- `subServiceId`: optional, UUID-like.
- `address`: optional, ≤500 chars.
- `addressSuggestion`: optional, passed through as-is; if present it is mapped to `addresses` in the request (structured DaData fields kept).
- `categoryOnly`: optional boolean; if true, subcategory is skipped.

`required-fields` only adds frontend “please fill” checks (default `clientPhone,description`); validation still rejects missing/invalid `serviceId`.

---

## Submission flow

The widget goes through these states: `validating → submitting → processing → success|error`.

On success, you get back `orderId` in the response.

---

## DOM events

Emitted on the element (bubble up):

- `widget:init`
- `widget:ready`
- `widget:form-change`
- `widget:submit-start`
- `widget:submit-progress`
- `widget:submit-success`
- `widget:submit-error`
- `widget:submit-complete`
- `widget:state-change`
- `widget:step-change`
- `widget:error`
- `widget:loading-change`
- `widget:service-categories-load`
- `widget:service-categories-error`
- `widget:service-select`

Example:

```js
const el = document.querySelector('astra-order-widget');
el.addEventListener('widget:submit-success', (e) => {
  console.log('Order created:', e.detail.orderId);
});
```

---

## Headless API quick reference

After `widget:ready`, the API is on `el.__ASTRA_WIDGET_API__`:

- **Config**: `getConfig()`, `updateConfig(partial)`
- **Form data**: `getFormData()`, `setFormData(partial)`, `clearFormData()`
- **Validation**: `validate(field?)`, `isValid()`
- **State**: `getState()` (simplified), `getCurrentStep()`, `goToStep(step)`
- **Submit**: `submit()`, `reset()`
- **Services**: `loadServices()`, `selectService(categoryId, subcategoryId?)`, `getSelectedService()`
- **Address**: `suggestAddress(query)`
- **Events**: `on`, `off`, `emit`
- **Cleanup**: `destroy()` (no-op)


---

## Troubleshooting

- **"Missing widget public key"** - Add the `api-key` attribute or set `window.ASTRA_WIDGET_PUBLIC_KEY` before loading the script.
- **Embedded widget not showing** - Make sure the container has width and isn't hidden by CSS.
- **Button icon looks wrong** - Only these icons work: `message`, `users`, `star`, `clock`, `phone`, `chat`/`message-square`, `map-pin`, `briefcase`, `user`.
- **CORS errors** - Make sure your domain is whitelisted in the CRM settings and you're using HTTPS.

---

## Browser support

Works in Chrome 90+, Firefox 88+, Safari 14+, Edge 90+, and mobile browsers (iOS Safari 14+, Chrome Mobile 90+).

---

## Code snippets

Copy these from AstraCRM UI (Widgets → Embed → Code):

```html
<script>
  window.ASTRA_WIDGET_PUBLIC_KEY = 'pk_live_xxxxx';
</script>
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.umd.js"></script>

<!-- Floating button -->
<astra-order-widget mode="floating" position="bottom-right" button-text="Request service"></astra-order-widget>

<!-- Embedded form -->
<astra-order-widget mode="embedded" theme="light" widget-title="Request service" widget-subtitle="We will contact you shortly"></astra-order-widget>
```

For headless mode only (no styles):

```html
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.headless.umd.js"></script>
<astra-order-widget mode="headless" api-key="..."></astra-order-widget>
```

---

## Need help?

- Integration questions: support@astracrm.pro
- Found a bug or have ideas? Open an issue in the docs repo



---

Also available: [Russian](./README_RU.md)

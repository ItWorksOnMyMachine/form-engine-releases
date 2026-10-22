# @clak/form-engine-sdk

Embed forms from Form Engine into any website. The SDK creates an iframe pointing at the form engine's embed route and communicates via postMessage.

## Installation

### Option 1: Script tag (simplest)

The SDK is hosted directly from the form engine server:

```html
<script src="https://forms.yourcompany.com/form-engine-sdk.js"></script>
```

This exposes a global `FormEngine` object.

The script is served by every form engine deployment at
`/form-engine-sdk.js`, so its version always matches the server. (The SDK is
not published to npm.)

## Quick Start

### Inline embed

Renders the form directly inside a container element on your page. The iframe auto-resizes to match the form's height.

```html
<div id="my-form"></div>

<script src="https://forms.yourcompany.com/form-engine-sdk.js"></script>
<script>
  FormEngine.render({
    formId: 'c107fb55-0a37-4654-8c3e-2305339c9f63',
    baseUrl: 'https://forms.yourcompany.com',
    container: '#my-form',
    onSubmitted: function(event) {
      console.log('Submitted!', event.submissionId);
    }
  });
</script>
```

### Dialog (modal overlay)

Opens the form in a centered modal dialog with a backdrop. Closes with the X button, Escape key, or clicking the backdrop. The dialog auto-closes after successful submission.

```js
FormEngine.dialog({
  formId: 'c107fb55-0a37-4654-8c3e-2305339c9f63',
  baseUrl: 'https://forms.yourcompany.com',
  width: '640px',
  onSubmitted: function(event) {
    console.log('Submitted!', event.submissionId);
  },
  onClosed: function() {
    console.log('Dialog was closed');
  }
});
```

### Popup (new window)

Opens the form in a new browser window. The window auto-closes after successful submission.

```js
FormEngine.popup({
  formId: 'c107fb55-0a37-4654-8c3e-2305339c9f63',
  baseUrl: 'https://forms.yourcompany.com',
  width: 640,
  height: 800,
  onSubmitted: function(event) {
    console.log('Submitted!', event.submissionId);
  },
  onClosed: function() {
    console.log('Popup was closed');
  }
});
```

## API Reference

### Common Options

All three methods (`render`, `dialog`, `popup`) accept these options:

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `formId` | `string` | Yes | The form's UUID (e.g. `'c107fb55-0a37-4654-8c3e-2305339c9f63'`) |
| `baseUrl` | `string` | Yes | The form engine server URL (e.g. `'https://forms.yourcompany.com'`) |
| `prefill` | `Record<string, string>` | No | Pre-fill form fields by key (e.g. `{ country: 'usa' }`) |
| `forwardParams` | `string[]` | No | Host-page query parameters to pass as prefill (see Forwarding below) |
| `followRedirect` | `boolean` | No | Navigate the host page to the form's success redirect URL after submission (default `true`) |
| `theme` | `object` | No | Override the form's colors (see Theme below) |
| `token` | `string` | No | Invitation token for non-anonymous forms |
| `onLoaded` | `function` | No | Called when the form is ready. Receives `{ formId, title }` |
| `onSubmitted` | `function` | No | Called after successful submission. Receives `{ formId, submissionId, data, redirectUrl? }` |
| `onError` | `function` | No | Called on submission error. Receives `{ formId, message }` |

### Theme Options

Override the tenant's default branding colors for this specific embed:

```js
theme: {
  primaryColor: '#ff6600',    // buttons, focus rings, accents
  cardBackground: '#f8f8f8',  // form card background
  textColor: '#222222'        // form text color
}
```

All fields are optional. Omitted values fall back to the tenant's branding configuration.

### `render(options)` — Inline Embed

Additional options:

| Option | Type | Description |
|--------|------|-------------|
| `container` | `HTMLElement \| string` | Target element or CSS selector |
| `onResize` | `function` | Called when form height changes. Receives `{ formId, height }` |

Returns a `FormEngineInstance`.

### `dialog(options)` — Modal Dialog

Additional options:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `width` | `string` | `'640px'` | CSS width of the dialog |
| `onClosed` | `function` | | Called when the dialog is dismissed (including after auto-close on submission) |

Returns a `FormEngineInstance`.

### `popup(options)` — New Window

Additional options:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `width` | `number` | `640` | Window width in pixels |
| `height` | `number` | `800` | Window height in pixels |
| `onClosed` | `function` | | Called when the popup window is closed (including after auto-close on submission) |

Returns a `FormEngineInstance`.

### FormEngineInstance

All three methods return an instance with:

| Method | Description |
|--------|-------------|
| `destroy()` | Remove the iframe, clean up event listeners, close dialog/popup |
| `getIframe()` | Returns the `HTMLIFrameElement` (or `null` for popup mode / after destroy) |

## Pre-filling Form Fields

Pass field keys and values via the `prefill` option. Values are validated against the form schema — invalid values (e.g. an option that doesn't exist for a select field) are silently ignored.

```js
FormEngine.render({
  formId: 'c107fb55-0a37-4654-8c3e-2305339c9f63',
  baseUrl: 'https://forms.yourcompany.com',
  container: '#form',
  prefill: {
    country: 'usa',
    practice_type: 'single'
  }
});
```

Any keys that don't match a form field are captured as `dynamic_values` in the submission data. This is useful for tracking context (e.g. customer IDs, campaign sources) without requiring dedicated form fields:

```js
prefill: {
  country: 'usa',           // matches a form field — pre-fills it
  customer_id: 'ACME-123',  // no matching field — stored in dynamic_values
  source: 'email-campaign'  // no matching field — stored in dynamic_values
}
```

## Forwarding Tracking Values

Campaign and lead-source values usually live on the host page: in its URL
(`?utm_source=...`), or computed by a site script and kept in cookies or
web storage. The form runs on another origin, so it can't see any of them.
Forward query parameters by name, and pass anything else through `prefill`:

```js
FormEngine.render({
  formId: 'c107fb55-0a37-4654-8c3e-2305339c9f63',
  baseUrl: 'https://forms.yourcompany.com',
  container: '#form',
  forwardParams: ['utm_source', 'utm_medium', 'utm_campaign'],
  prefill: {
    // e.g. a value a site script keeps in sessionStorage
    lead_source: sessionStorage.getItem('lead_source') || undefined,
  },
});
```

- Parameters missing from the host page, and empty prefill values, are skipped.
- `prefill` wins over `forwardParams` for the same key.
- The embed's reserved parameters (`token`, `theme`, `cardBg`, `textColor`)
  are never forwarded.
- Like any prefill, forwarded keys that match a field fill it; the rest land
  in `dynamic_values` on the submission.

## Success Redirect

If the form has a success redirect URL, the form can't navigate your page
from inside its iframe (or popup). Instead, after `onSubmitted` runs, the SDK
navigates the host page to that URL. Only absolute `http(s)` URLs are
followed. Set `followRedirect: false` to handle it yourself with
`event.redirectUrl`.

The SDK accepts messages only from the form engine's origin (`baseUrl`) and
from the iframe or popup window it opened.

## Example

See the `examples/embed-harness/` directory for a complete working example that demonstrates all three embed modes with an event log panel.

To run the example locally:

```bash
# From the form-engine root directory

# 1. Build the SDK
npm run build:sdk

# 2. Start the form engine server
npm run dev

# 3. In another terminal, start the harness
npm run --workspace examples/embed-harness dev

# 4. Open http://localhost:5173
```

## Hosting

The SDK is automatically served from the form engine at `/form-engine-sdk.js`. After building:

```bash
npm run build:sdk
```

The IIFE bundle is copied to `public/form-engine-sdk.js` and served by Next.js at your form engine's domain.

## Browser Support

The SDK uses standard DOM APIs and `postMessage`. It works in all modern browsers (Chrome, Firefox, Safari, Edge). No polyfills required.

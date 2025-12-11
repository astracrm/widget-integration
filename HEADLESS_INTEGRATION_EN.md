# AstraCRM Widget — Headless Integration

[Русская версия](./HEADLESS_INTEGRATION_RU.md)

Headless mode gives you the widget’s business logic, validation, and API calls with zero UI. The element renders nothing; you own the markup. Use this when you need custom layouts but still want the widget’s validation, service catalog, address suggestions, and submission flow.

`widget:ready` fires only when the element is correctly configured. If attributes are wrong, it will never fire (no API attached).

---

## What you need
- Script: `https://cdn.astracrm.pro/widget/v1/astra-widget.headless.umd.js`
- A public widget key (`api-key` attribute or `window.ASTRA_WIDGET_PUBLIC_KEY`)
- Modern browser + HTTPS

---

## Quickstart (copy/paste)
```html
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.headless.umd.js"></script>

<astra-order-widget
  mode="headless"
  api-url="https://api.astracrm.pro/api/v1"
  api-key="YOUR_PUBLIC_API_KEY">
</astra-order-widget>

<script>
  document.addEventListener('DOMContentLoaded', () => {
    const widget = document.querySelector('astra-order-widget')
    widget.addEventListener('widget:ready', async () => {
      const api = widget.__ASTRA_WIDGET_API__ // attached on ready

      api.setFormData({
        clientPhone: '79991234567', // normalize to digits
        description: 'Need plumbing today',
        serviceId: '00000000-0000-0000-0000-000000000000', // category (required)
        subServiceId: undefined, // subcategory (optional)
        categoryOnly: false,
      })

      const errors = api.validate()
      if (!api.isValid()) {
        console.warn('Fix errors before submit', errors)
        return
      }

      try {
        const result = await api.submit()
        console.log('Order submitted:', result.orderId)
      } catch (err) {
        console.error('Submit failed', err)
      }
    }, { once: true })
  })
</script>
```

---

## Order form fields you must manage

| Field | Required | Notes |
| --- | --- | --- |
| `clientPhone` | Yes | Regex `^7\\d{10}$` (exactly 11 digits starting with `7`). |
| `description` | Yes | 1..1000 chars. |
| `serviceId` | Yes | UUID-like. Required for submission (always validated). |
| `subServiceId` | No | UUID-like. Use when a subcategory is chosen. |
| `categoryOnly` | No | `true` to submit only category (no subcategory). |
| `address` | No | Free text, ≤500 chars. |
| `addressSuggestion` | No | DaData-style structured suggestion; if set, it is mapped to `addresses` in the payload (preferred over free text). |

The widget never reads your DOM. Always push values into the API (`setFormData` or your own snapshot before `validate/submit`).

---

## Headless API surface (runtime)
Everything lives on `widget.__ASTRA_WIDGET_API__`:

- Config: `getConfig()`, `updateConfig(partial)`
- Form data: `getFormData()`, `setFormData(partial)`, `clearFormData()`
- Validation: `validate(field?)`, `isValid()`
- State: `getState()` (simplified), `getCurrentStep()`, `goToStep(step)`
- Submission: `submit()`, `reset()`
- Services: `loadServices()`, `selectService(categoryId, subcategoryId?)`, `getSelectedService()`
- Address: `suggestAddress(query)`
- Events: `on`, `off`, `emit`
- Lifecycle: `destroy()` (no-op)

`setFormData` updates the internal snapshot synchronously so `validate/isValid/getFormData` immediately see the latest values even if React state hasn’t re-rendered yet.

---

## Events

### API events (`on/off`) that are emitted
- `onInit(config)`
- `onReady()`
- `onFormChange(partialFormData)`
- `onSubmitStart(formData)`
- `onSubmitProgress(progress)`
- `onSubmitSuccess(response)`
- `onSubmitError(error)`
- `onSubmitComplete()`
- `onStateChange(widgetState)`
- `onStepChange({ step, direction })`
- `onError(widgetError)`
- `onLoadingChange(isLoading)`
- `onServiceCategoriesLoad(categories)`
- `onServiceCategoriesError(error)`
- `onServiceSelect(selectedService)`

### DOM custom events (`widget:*`)
Names are camelCase → kebab-case of the events above, payload in `event.detail`:

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

---

## Integration patterns

### Vanilla JS (full flow: catalog, address suggestions, field snapshot)
```html
<astra-order-widget mode="headless" api-url="https://api.astracrm.pro/api/v1" api-key="KEY"></astra-order-widget>
<input id="phone" />
<textarea id="desc"></textarea>
<select id="category"></select>
<select id="subcategory"></select>
<label><input id="category-only" type="checkbox" /> Category only</label>
<input id="address" placeholder="Start typing address..." />
<ul id="addr-suggestions"></ul>
<button id="send">Send</button>
<div id="errors"></div>
<script>
  const byId = (id) => document.getElementById(id)
  const renderErrors = (errs) => {
    const el = byId('errors')
    const first = Object.values(errs)[0]?.[0]
    el.textContent = first ? `Error: ${first}` : ''
  }
  const normalizePhone = (v) => (v || '').replace(/\D/g, '')
  let catalog = []

  document.addEventListener('widget:ready', () => {
    const api = document.querySelector('astra-order-widget').__ASTRA_WIDGET_API__

    // load categories/subcategories
    api.loadServices().then((cats) => {
      catalog = cats || []
      const catSel = byId('category')
      const subSel = byId('subcategory')
      catSel.innerHTML = '<option value=\"\">Select category</option>'
      catalog.forEach((c) => {
        const o = document.createElement('option')
        o.value = c.id
        o.textContent = c.name
        catSel.appendChild(o)
      })
      const first = catalog[0]
      const firstSub = first?.subcategories?.[0]
      if (first) {
        api.selectService(first.id, firstSub?.id)
        catSel.value = first.id
        if (firstSub) subSel.value = firstSub.id
      }
    })

    byId('category').addEventListener('change', (e) => {
      const catId = e.target.value
      const cat = catalog.find((c) => c.id === catId)
      const subSel = byId('subcategory')
      subSel.innerHTML = '<option value=\"\">Select subcategory</option>'
      cat?.subcategories?.forEach((s) => {
        const o = document.createElement('option')
        o.value = s.id
        o.textContent = s.name
        subSel.appendChild(o)
      })
      const subId = byId('category-only').checked ? undefined : subSel.value || undefined
      api.selectService(catId, subId)
    })

    byId('subcategory').addEventListener('change', (e) => {
      const catId = byId('category').value || ''
      const subId = byId('category-only').checked ? undefined : e.target.value || undefined
      api.selectService(catId, subId)
    })

    byId('category-only').addEventListener('change', (e) => {
      const catId = byId('category').value || ''
      const subId = e.target.checked ? undefined : byId('subcategory').value || undefined
      api.setFormData({ categoryOnly: e.target.checked, subServiceId: subId })
      api.selectService(catId, subId)
    })

    // Address suggestions
    const addrList = byId('addr-suggestions')
    const clearAddr = () => { addrList.innerHTML = '' }
    const renderAddr = (list) => {
      clearAddr()
      list.forEach((sug) => {
        const li = document.createElement('li')
        li.style.cursor = 'pointer'
        li.textContent = sug.unrestricted_value || sug.value || ''
        li.onclick = () => {
          api.setFormData({ addressSuggestion: sug, address: sug.unrestricted_value || sug.value || '' })
          byId('address').value = sug.unrestricted_value || sug.value || ''
          clearAddr()
        }
        addrList.appendChild(li)
      })
    }
    let addrTimer = null
    byId('address').addEventListener('input', (e) => {
      const q = e.target.value || ''
      api.setFormData({ address: q })
      if (addrTimer) clearTimeout(addrTimer)
      addrTimer = setTimeout(async () => {
        const list = await api.suggestAddress(q)
        renderAddr(list)
      }, 250)
    })

    // Form fields
    byId('phone').addEventListener('input', e => {
      api.setFormData({ clientPhone: normalizePhone(e.target.value) })
    })
    byId('desc').addEventListener('input', e => {
      api.setFormData({ description: e.target.value })
    })

    byId('send').addEventListener('click', async () => {
      // snapshot DOM into API before validate/submit
      api.setFormData({
        clientPhone: normalizePhone(byId('phone').value),
        description: byId('desc').value,
        categoryOnly: byId('category-only').checked,
        subServiceId: byId('category-only').checked ? undefined : (byId('subcategory').value || undefined),
        address: byId('address').value,
      })

      const errors = api.validate()
      renderErrors(errors)
      if (!api.isValid()) return
      try {
        const result = await api.submit()
        alert(`OK: ${result.orderId}`)
      } catch (err) {
        alert(`Fail: ${err.message}`)
      }
    })
  }, { once: true })
</script>
```

### React hook (realistic)
```tsx
import { useEffect, useState, useMemo } from 'react'
import { useHeadlessWidget, type HeadlessWidgetAPI } from 'astra-widget'

const useWidgetApiReady = () => {
  const [api, setApi] = useState<HeadlessWidgetAPI | null>(null)
  useEffect(() => {
    const widget = document.querySelector('astra-order-widget')
    const handler = () => setApi((widget as any).__ASTRA_WIDGET_API__ as HeadlessWidgetAPI)
    widget?.addEventListener('widget:ready', handler, { once: true })
    return () => widget?.removeEventListener('widget:ready', handler)
  }, [])
  return api
}

export function CustomForm() {
  const api = useWidgetApiReady()
  const renderProps = useMemo(
    () => (api ? useHeadlessWidget(api, api.getConfig()) : null),
    [api]
  )

  if (!renderProps) return <div>Loading...</div>

  const { formData, updateField, validateField, submit, isLoading, isValid, validationErrors } = renderProps

  return (
    <form
      onSubmit={async (e) => {
        e.preventDefault()
        validateField('clientPhone')
        validateField('description')
        if (!isValid) return
        await submit()
      }}
    >
      <input
        value={formData.clientPhone}
        onChange={(e) => updateField('clientPhone', e.currentTarget.value)}
      />
      {validationErrors.clientPhone?.[0]}

      <textarea
        value={formData.description}
        onChange={(e) => updateField('description', e.currentTarget.value)}
      />
      {validationErrors.description?.[0]}

      <button disabled={!isValid || isLoading}>{isLoading ? 'Sending…' : 'Send'}</button>
    </form>
  )
}
```

### Render-props variant
```tsx
import { HeadlessRenderer } from 'astra-widget'

<HeadlessRenderer api={api} config={api.getConfig()}>
  {({ formData, updateField, submit, isLoading, isValid, validationErrors, currentStep }) => (
    <div>
      {currentStep === 'form' && (
        <>
          <input
            value={formData.clientPhone}
            onChange={(e) => updateField('clientPhone', e.currentTarget.value)}
          />
          {validationErrors.clientPhone?.[0]}
          <button disabled={!isValid || isLoading} onClick={() => submit()}>Submit</button>
        </>
      )}
      {currentStep === 'success' && <div>Submitted</div>}
      {currentStep === 'error' && <div>Failed — check errors</div>}
    </div>
  )}
</HeadlessRenderer>
```

The built-in state machine has a simple path (`form` → `submitting` → `success | error`). Extra step IDs (`service-selection`, `review`) are available for your own UI flow; the widget does not enforce transitions between them.

---

## TypeScript support (what the package exports)
```ts
import type {
  HeadlessWidgetAPI,
  HeadlessWidgetConfig,
  HeadlessWidgetEvents,
  OrderFormData,
  WidgetStep,
  ValidationErrors,
  SubmissionSuccess,
  SubmissionError,
  WidgetError,
} from 'astra-widget'
```
`HeadlessRenderer` and `useHeadlessWidget` also come from `'astra-widget'`.

---

## Best practices
- Validate on change/blur, not on a timer. Call `api.validate(field)` and render returned messages.
- Always wait for `widget:ready` before touching `__ASTRA_WIDGET_API__`.
- `getState().currentStep` is simplified (`form | success | error`); for the full enum use `getCurrentStep()`.
- `goToStep('form')` resets the form; only call it intentionally.
- Handle `fieldErrors` in submission errors to highlight exact fields.
- Phone: normalize to digits yourself (and add prefixes if needed) before `setFormData` or your submit snapshot.
- Address: headless mode draws no UI; use `suggestAddress`, put the chosen suggestion into `addressSuggestion` (and optionally mirror it into `address` for display).

---

## Notes and limits
- Headless mode hides the element; all UI is yours.
- Mobile/React Native: embed via WebView; the headless API is JS-only and not covered here.
- `widget:ready` depends on valid attributes/config; misconfiguration = no event.
- Backend widget settings (title/subtitle/button, mode/position, order-state, required-fields, locale) act as defaults only; your explicit attributes or headless `updateConfig` override them. Branding/colors and service catalog always come from backend.
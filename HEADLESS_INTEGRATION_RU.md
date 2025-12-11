# AstraCRM Widget - Headless интеграция

[English version](./HEADLESS_INTEGRATION_EN.md)

Headless-режим даёт бизнес-логику, валидацию и API вызовы виджета без UI. Элемент ничего не рисует - весь интерфейс делаете вы. Подходит, когда нужен свой дизайн, но хочется готовые проверки, каталог услуг, подсказки адресов и процесс отправки.

`widget:ready` срабатывает только при корректной конфигурации. Если атрибуты заданы неверно, события не будет (API не прикрепится).

---

## Что нужно
- Скрипт: `https://cdn.astracrm.pro/widget/v1/astra-widget.headless.umd.js`
- Публичный ключ виджета (`api-key` атрибут или `window.ASTRA_WIDGET_PUBLIC_KEY`)
- Современный браузер + HTTPS

---

## Быстрый старт (скопировать/вставить)
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
      const api = widget.__ASTRA_WIDGET_API__ // API появляется на ready

      api.setFormData({
        clientPhone: '79991234567', // нормализуйте до цифр
        description: 'Нужен сантехник сегодня',
        serviceId: '00000000-0000-0000-0000-000000000000', // категория (обязательно)
        subServiceId: undefined, // подкатегория (опционально)
        categoryOnly: false,
      })

      const errors = api.validate()
      if (!api.isValid()) {
        console.warn('Исправьте ошибки перед отправкой', errors)
        return
      }

      try {
        const result = await api.submit()
        console.log('Заказ создан:', result.orderId)
      } catch (err) {
        console.error('Ошибка отправки', err)
      }
    }, { once: true })
  })
</script>
```

---

## Поля формы, которыми управляете вы

| Поле | Обязательно | Примечание |
| --- | --- | --- |
| `clientPhone` | Да | Регекс `^7\\d{10}$` (ровно 11 цифр, начинается с `7`). |
| `description` | Да | 1..1000 символов. |
| `serviceId` | Да | UUID-подобный. Всегда требуется для отправки. |
| `subServiceId` | Нет | UUID-подобный. Используйте, если выбрана подкатегория. |
| `categoryOnly` | Нет | `true`, чтобы отправлять только категорию без подкатегории. |
| `address` | Нет | Свободный текст, ≤ 500 символов. |
| `addressSuggestion` | Нет | Структурированная подсказка DaData; если есть, маппится в `addresses` в запросе и предпочтительнее свободного текста. |

Виджет не читает ваш DOM. Всегда прокидывайте значения в API (`setFormData` или снапшот перед `validate/submit`).

---

## API headless (в рантайме)
Всё находится в `widget.__ASTRA_WIDGET_API__`:

- Конфигурация: `getConfig()`, `updateConfig(partial)`
- Данные формы: `getFormData()`, `setFormData(partial)`, `clearFormData()`
- Валидация: `validate(field?)`, `isValid()`
- Состояние: `getState()` (упрощённое), `getCurrentStep()` (полный enum), `goToStep(step)`
- Отправка: `submit()`, `reset()`
- Каталог услуг: `loadServices()`, `selectService(categoryId, subcategoryId?)`, `getSelectedService()`
- Адреса: `suggestAddress(query)`
- События: `on(event, handler)`, `off(event, handler)`, `emit(event, ...args)`
- Жизненный цикл: `destroy()` (заглушка)

`setFormData` синхронно обновляет внутренний снапшот, поэтому `validate/isValid/getFormData` сразу видят актуальные данные, даже если React ещё не перерисовался.

---

## События

### API события (`on/off`), которые реально эмитятся
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

### DOM события (`widget:*`)
Названия - camelCase → kebab-case от списка выше, payload в `event.detail`:

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

## Паттерны интеграции

### Vanilla JS (каталог, адресные подсказки, снап значений)
```html
<astra-order-widget mode="headless" api-url="https://api.astracrm.pro/api/v1" api-key="KEY"></astra-order-widget>
<input id="phone" />
<textarea id="desc"></textarea>
<select id="category"></select>
<select id="subcategory"></select>
<label><input id="category-only" type="checkbox" /> Только категория</label>
<input id="address" placeholder="Начните вводить адрес..." />
<ul id="addr-suggestions"></ul>
<button id="send">Отправить</button>
<div id="errors"></div>
<script>
  const byId = (id) => document.getElementById(id)
  const renderErrors = (errs) => {
    const el = byId('errors')
    const first = Object.values(errs)[0]?.[0]
    el.textContent = first ? `Ошибка: ${first}` : ''
  }
  const normalizePhone = (v) => (v || '').replace(/\D/g, '')
  let catalog = []

  document.addEventListener('widget:ready', () => {
    const api = document.querySelector('astra-order-widget').__ASTRA_WIDGET_API__

    // загрузка категорий/подкатегорий
    api.loadServices().then((cats) => {
      catalog = cats || []
      const catSel = byId('category')
      const subSel = byId('subcategory')
      catSel.innerHTML = '<option value=\"\">Выберите категорию</option>'
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
      subSel.innerHTML = '<option value=\"\">Выберите подкатегорию</option>'
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

    // Подсказки адресов
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

    // Поля формы
    byId('phone').addEventListener('input', e => {
      api.setFormData({ clientPhone: normalizePhone(e.target.value) })
    })
    byId('desc').addEventListener('input', e => {
      api.setFormData({ description: e.target.value })
    })

    byId('send').addEventListener('click', async () => {
      // снапшот DOM в API перед validate/submit
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
        alert(`Ошибка: ${err.message}`)
      }
    })
  }, { once: true })
</script>
```

### React hook (практичный пример)
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

### Render-props вариант
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
      {currentStep === 'error' && <div>Failed - check errors</div>}
    </div>
  )}
</HeadlessRenderer>
```

Базовая машина состояний простая (`form` → `submitting` → `success | error`). Дополнительные шаги (`service-selection`, `review`) - для вашей логики; переходы между ними виджет не навязывает.

---

## TypeScript (что экспортирует пакет)
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
`HeadlessRenderer` и `useHeadlessWidget` тоже из `'astra-widget'`.

---

## Рекомендации
- Валидируйте по change/blur, а не по таймеру. Вызывайте `api.validate(field)` и показывайте сообщения.
- Всегда ждите `widget:ready` перед обращением к `__ASTRA_WIDGET_API__`.
- `getState().currentStep` упрощён (`form | success | error`); полный набор шагов - в `getCurrentStep()`.
- `goToStep('form')` сбрасывает форму; вызывайте осознанно.
- Обрабатывайте `fieldErrors` в ошибках отправки, чтобы подсветить конкретные поля.
- Телефон: нормализуйте до цифр (и добавляйте префиксы при необходимости) до `setFormData` или перед отправкой.
- Адрес: в headless-режиме UI нет - используйте `suggestAddress`, выбранную подсказку кладите в `addressSuggestion` (и при желании дублируйте в `address` для отображения).

---

## Ограничения
- Элемент скрыт; весь UI делаете вы.
- Mobile/React Native: встраивайте через WebView; headless API только JS и не покрыт здесь.
- `widget:ready` зависит от валидной конфигурации; при ошибке в атрибутах событие не придёт.
- Настройки с бэкенда (заголовок/подзаголовок/кнопка, mode/position, order-state, required-fields, locale) используются только как значения по умолчанию; явные атрибуты или headless `updateConfig` их перекрывают. Брендинг/цвета и каталог всегда приходят с бэкенда.

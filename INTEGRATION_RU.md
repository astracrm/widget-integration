# AstraCRM Widget - Полное руководство по интеграции (RU)

[English version](./INTEGRATION_EN.md)

Это официальный гайд по интеграции виджета заявки AstraCRM на сторонние сайты. Документация рассчитана на фронтенд‑разработчиков любого уровня: вставляйте виджет за минуты, без чтения исходников (исходный код закрыт, поставка - UMD-бандл).

- **Компонент**: кастомный HTML‑элемент `astra-order-widget`
- **Режимы**: Floating (плавающая кнопка), Embedded (встроенный блок), Headless (без UI)
- **Поставки**: один UMD-файл с полным UI и режимом `headless`; опционально headless‑только сборка
- **CDN**: `https://cdn.astracrm.pro/widget/v1/astra-widget.umd.js`
- **Минимальные требования**: современный браузер, HTTPS, публичный ключ виджета

---

## Быстрый старт (TL;DR)

### Вариант A - через атрибут `api-key`

```html
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.umd.js"></script>

<astra-order-widget
  api-key="ВАШ_ПУБЛИЧНЫЙ_КЛЮЧ"
  mode="floating"
  position="bottom-right"
  button-text="Оставить заявку">
</astra-order-widget>
```

### Вариант B - через глобальную переменную (часто генерится во фронте CRM)

```html
<script>
  window.ASTRA_WIDGET_PUBLIC_KEY = 'ВАШ_ПУБЛИЧНЫЙ_КЛЮЧ';
  // Необязательно: кастомный API base URL
  // window.ASTRA_WIDGET_API_BASE_URL = 'https://api.astracrm.com';
</script>
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.umd.js"></script>

<astra-order-widget mode="floating" position="bottom-right"></astra-order-widget>
```

Готово. Виджет сам зарегистрирует `customElements.define('astra-order-widget', ...)` и отрисуется.

---

## Варианты интеграции

### 1) Floating (плавающая кнопка)
- В DOM добавляется плавающая кнопка, по клику открывается форма.
- Позиции: `bottom-right` (по умолчанию), `bottom-left`, `top-right`, `top-left`, `bottom-center`, `right-center`, `left-center`.
- Кастомизация кнопки: `button-text`, `button-icon`.

Пример:
```html
<astra-order-widget
  api-key="..."
  mode="floating"
  position="bottom-center"
  button-text="Нужна помощь?"
  button-icon="message">
</astra-order-widget>
```

Поддерживаемые значения `button-icon`: `message` (по умолчанию), `users`, `star`, `clock`.

### 2) Embedded (встроенный)
- Виджет рендерится как обычный блок в месте вставки, подстраивается по ширине контейнера.
- Можно задать заголовок и подзаголовок.

Пример:
```html
<section class="contact">
  <h2>Заявка на услугу</h2>
  <astra-order-widget
    api-key="..."
    mode="embedded"
    theme="light"
    widget-title="Оставить заявку"
    widget-subtitle="Опишите проблему - мы перезвоним">
  </astra-order-widget>
</section>
```

### 3) Headless (без UI)
- Вся логика/валидация/сетевые вызовы - внутри виджета, но UI контролируете вы.
- Включается `mode="headless"`. После инициализации на DOM‑элементе доступен API `__ASTRA_WIDGET_API__`.
- Опционально можно подключить headless‑только бандл `astra-widget.headless.umd.js` (если нужен только API и ноль стилей).

Минимальный пример:
```html
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.umd.js"></script>
<astra-order-widget mode="headless" api-key="..."></astra-order-widget>
<script>
  const el = document.querySelector('astra-order-widget');
  el.addEventListener('widget:ready', async () => {
    const api = el.__ASTRA_WIDGET_API__;
    api.setFormData({ clientPhone: '+79990000000', description: 'Нужен мастер' });
    if (api.isValid()) {
      const res = await api.submit();
      console.log('Создан заказ:', res.orderId);
      if (res.redirectUrl) window.location.href = res.redirectUrl;
    }
  }, { once: true });
<\/script>
```

Расширенное руководство по headless: `docs/HEADLESS_INTEGRATION.md` (в репозитории).

---

## Конфигурация через атрибуты

Атрибуты указываются на теге `astra-order-widget`. Большинство можно менять «на лету» - виджет отслеживает изменения.

| Атрибут | Тип/значения | По умолчанию | Назначение |
|---|---|---|---|
| `api-key` | string | - | Публичный ключ виджета. Либо задайте через `window.ASTRA_WIDGET_PUBLIC_KEY`. |
| `api-url` | URL | если dev: `http://localhost:8000`, иначе берётся из `window.ASTRA_WIDGET_API_BASE_URL` | Базовый URL публичного API виджета. |
| `mode` | `floating` · `embedded` · `headless` | `floating` | Режим работы.|
| `theme` | `light` · `dark` | `light` | Цветовая тема. |
| `locale` | `en` · `ru` | `en` | Язык UI/сообщений. |
| `required-fields` | CSV ключей формы (`clientPhone,description,...`) | `clientPhone,description` | Какие поля обязательны. |
| `show-service-selector` | `true` · `false` | `false` | Показывать выбор услуги/подуслуги (если настроено на бэке). |
| `position` | см. список позиций | `bottom-right` | Позиция плавающей кнопки (режим `floating`). |
| `button-text` | string | автотекст по `locale` | Текст на кнопке (режим `floating`). |
| `button-icon` | `message` · `users` · `star` · `clock` | `message` | Иконка кнопки (режим `floating`). |
| `widget-title` | string | автотекст | Заголовок формы (актуально для `embedded`/`floating` экрана приветствия). |
| `widget-subtitle` | string | автотекст | Подзаголовок формы. |
| `order-state` | `draft` · `distributing` | `draft` | Начальный статус заказа при публичной отправке. |
| `debug` | `true` · `false` | `false` | Подробные логи событий и DOM‑ивенты. |
| `success-redirect` | URL | - | Не используется автоматически. Можно читать вручную, чтобы редиректить после успеха. |

Приоритет: явные атрибуты на теге > глобальные переменные `window.*`.

---

## Глобальные переменные (опционально)

- `window.ASTRA_WIDGET_PUBLIC_KEY`: публичный ключ, если не хотите светить его в атрибуте.
- `window.ASTRA_WIDGET_API_BASE_URL`: базовый URL публичных эндпоинтов, если хотите переопределить по среде.

Обычно этот блок генерирует интерфейс AstraCRM (раздел «Виджеты → Встраивание → Код вставки»).

```html
<script>
  window.ASTRA_WIDGET_PUBLIC_KEY = 'pk_live_xxxxx';
  window.ASTRA_WIDGET_API_BASE_URL = 'https://api.astracrm.com';
</script>
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.umd.js"></script>
```

---

## Сеть и безопасность

- Публичный ключ - не секрет. Секретов ни в атрибутах, ни в коде - нет.
- Всегда используйте **HTTPS** и корректный **CORS** на вашем домене.
- Рекомендуем включить CSP и ограничить выполнение скриптов доверенными источниками.

Публичные эндпоинты виджета (для справки):
- `GET /public/widget-config` - конфиг: брендинг, категории, локаль
- `POST /public/orders/add` - создание заявки
- `POST /public/suggest/address` - подсказки адреса (DaData)

---

## Локализация

- `locale="en" | "ru"` - переключает язык встроенных текстов.
- Заголовки можно переопределить атрибутами `widget-title`/`widget-subtitle`.

---

## Стилизация и темы

`theme="light|dark"`. Для тонкой настройки доступны CSS‑переменные (применяются к корню виджета):

```css
astra-order-widget {
  /* ключевые цвета */
  --ow-primary: #3b82f6;
  --ow-secondary: #64748b;
  --ow-background: #ffffff;
  --ow-text: #1f2937;
  --ow-border: #e5e7eb;
  /* углы, тени и т.д. */
  --ow-border-radius: 12px;
  --ow-shadow-floating: 0 25px 50px -12px rgba(0,0,0,.25);
}
```

Переключение темы влияет на набор переменных и готовые utility‑классы внутри шадоу‑DOM виджета.

---

## Форма и валидация (что реально проверяется)

Модель данных отправки:
- `clientPhone`: обязателен; формат: `^((+?7)|8)?\d{10}$` (поддержка `+7/7/8` + 10 цифр)
- `description`: обязателен; 1..1000 символов
- `serviceId`: обязателен; UUID‑like
- `subServiceId`: опционально; UUID‑like
- `categoryOnly`: опционально; если `true`, подуслуга не требуется
- `address`: опционально; ≤500 символов
- `addressSuggestion`: опционально; структурированный адрес (DaData) - при наличии используется для формирования `addresses`

Атрибут `required-fields` позволяет добавить/убрать обязательные поля на клиенте.

---

## Отправка и результат

- Состояния отправки: `validating → submitting → processing → success|error`.
- Ответ успеха содержит как минимум `orderId`, может содержать `redirectUrl` (если настроено на бэке).
- Авторедиректа по `success-redirect` нет - используйте `redirectUrl` из ответа или headless/события DOM.

---

## События DOM (для интеграции без headless‑кода)

Виджет диспатчит CustomEvent‑события на сам элемент `astra-order-widget` (и всплывают вверх):

- `widget:init` - инициализация, detail: текущая конфигурация
- `widget:ready` - API готов
- `widget:destroy` - виджет демонтирован
- `widget:form-change` - изменение данных формы, detail: частичный объект формы
- `widget:validation-change` - ошибки валидации
- `widget:submit-start` - старт отправки, detail: данные формы
- `widget:submit-success` - успех, detail: `{ orderId, ... }`
- `widget:submit-error` - ошибка отправки, detail: `{ code, message, ... }`
- `widget:state-change` - изменение агрегированного состояния
- `widget:step-change` - смена шага (`form|submitting|success|error`)
- `widget:error` - системная/сетевоая ошибка

Пример подписки:
```js
const el = document.querySelector('astra-order-widget');
el.addEventListener('widget:submit-success', (e) => {
  console.log('Заказ создан:', e.detail.orderId);
});
```

Включите `debug="true"`, чтобы видеть подробные логи и метки событий в консоли.

---

## Headless API (краткая справка)

API доступен как `el.__ASTRA_WIDGET_API__` после `widget:ready`.

Основные методы:
- Конфигурация: `getConfig()`, `updateConfig(partial)`
- Данные формы: `getFormData()`, `setFormData(partial)`, `clearFormData()`
- Валидация: `validate(field?)`, `isValid()`
- Состояния/этапы: `getState()`, `getCurrentStep()`, `goToStep(step)`
- Отправка: `submit(): Promise<{ orderId, redirectUrl? }>`; `reset()`
- Сервисы: `loadServices()`, `selectService(categoryId, subcategoryId?)`, `getSelectedService()`
- События API: `on(event, handler)`, `off(event, handler)`, `emit(event, ...args)`
- Жизненный цикл: `destroy()`

Полный детальный гайд и расширенные примеры - см. `docs/HEADLESS_INTEGRATION.md`.

---

## Диагностика и FAQ

- «Missing widget public key» - нет `api-key` и не задан `window.ASTRA_WIDGET_PUBLIC_KEY`.
- «Missing widget API base URL» - не обнаружен API base. Задайте `window.ASTRA_WIDGET_API_BASE_URL` или используйте dev‑окружение.
- Ничего не видно в `embedded` - проверьте ширину контейнера и стили (виджет сам тянется на 100%).
- Иконка кнопки «не та» - используйте поддерживаемые значения: `message`, `users`, `star`, `clock`.
- CORS - проверяйте, что ваш домен разрешён на стороне API, и все запросы идут по HTTPS.

---

## Поддержка браузеров

- Chrome 90+
- Firefox 88+
- Safari 14+
- Edge 90+
- Мобильные браузеры (iOS Safari 14+, Chrome Mobile 90+)

---

## Полезные сниппеты

### Генерация кода из интерфейса AstraCRM

Интерфейс CRM (Виджеты → Встраивание → Предпросмотр/Код вставки) выдаёт готовый HTML:

```html
<script>
  window.ASTRA_WIDGET_PUBLIC_KEY = 'pk_live_xxxxx';
  window.ASTRA_WIDGET_API_BASE_URL = 'https://api.astracrm.com';
</script>
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.umd.js"></script>

<!-- Floating -->
<astra-order-widget mode="floating" position="bottom-right" button-text="Оставить заявку"></astra-order-widget>

<!-- Embedded -->
<astra-order-widget mode="embedded" theme="light" widget-title="Оставить заявку" widget-subtitle="Мы свяжемся с вами"></astra-order-widget>
```

### Чистый headless без UI

```html
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.headless.umd.js"></script>
<astra-order-widget mode="headless" api-key="..."></astra-order-widget>
```

- используйте, если точно нужен только программный API без загрузки стилей.

---

## Обратная связь

- Вопросы по интеграции: support@astracrm.pro
- Идеи/замечания по виджету: оформляйте ишью в публичной доке/репозитории документации

*** Конец документа ***

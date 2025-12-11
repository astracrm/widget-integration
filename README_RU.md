# AstraCRM Widget - Руководство по интеграции

[English version](./README.md)

Как встроить виджет заявок AstraCRM на ваш сайт. Скопируйте сниппет, настройте пару атрибутов - готово. Исходники читать не нужно (да и не получится - поставляется как UMD-бандл).

Что внутри:
- Кастомный элемент `astra-order-widget`, который просто работает
- Три режима: плавающая кнопка, встроенный блок или headless (без UI)
- Один UMD-файл с CDN, или headless-версия без стилей
- CDN: `https://cdn.astracrm.pro/widget/v1/astra-widget.umd.js`
- Нужно: современный браузер, HTTPS и публичный ключ виджета

---

## Быстрый старт

Два способа подключить:

### Вариант A: Передать ключ через атрибут

```html
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.umd.js"></script>

<astra-order-widget
  api-key="ВАШ_ПУБЛИЧНЫЙ_КЛЮЧ"
  mode="floating"
  position="bottom-right"
  button-text="Оставить заявку">
</astra-order-widget>
```

### Вариант B: Использовать глобальную переменную (можно скопировать из CRM: Настройки → Виджеты)

```html
<script>
  window.ASTRA_WIDGET_PUBLIC_KEY = 'ВАШ_ПУБЛИЧНЫЙ_КЛЮЧ';
</script>
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.umd.js"></script>

<astra-order-widget mode="floating" position="bottom-right"></astra-order-widget>
```

Всё. Виджет сам зарегистрируется и заработает.

---

## Режимы работы

### Плавающая кнопка
Кнопка, которая висит на странице и открывает форму по клику. Подходит для сценариев типа "свяжитесь с нами".

Позиции: `bottom-right` (по умолчанию), `bottom-left`, `top-right`, `top-left`, `bottom-center`, `right-center`, `left-center`.

Можно настроить текст и иконку кнопки:

```html
<astra-order-widget
  api-key="..."
  mode="floating"
  position="bottom-center"
  button-text="Нужна помощь?"
  button-icon="message">
</astra-order-widget>
```

Доступные иконки: `message` (по умолчанию), `users`, `star`, `clock`, `phone`, `chat` (алиас `message-square`), `message-square`, `map-pin`, `briefcase`, `user`.

### Встроенный блок
Рендерится прямо там, где вы его поставили, как обычная форма. Занимает всю ширину контейнера. Можно задать свой заголовок и подзаголовок.

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

### Headless режим
Виджет берёт на себя валидацию и API-вызовы, а UI делаете вы сами. Идеально, когда нужен полный контроль над дизайном формы.

Включите `mode="headless"`, после инициализации используйте `__ASTRA_WIDGET_API__` на элементе. Есть и headless-бандл (`astra-widget.headless.umd.js`), если стили вообще не нужны.

Подробное руководство по headless: [HEADLESS_INTEGRATION_RU.md](./HEADLESS_INTEGRATION_RU.md)

Минимальный пример:
```html
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.headless.umd.js"></script>
<astra-order-widget mode="headless" api-key="..."></astra-order-widget>
<script>
  const el = document.querySelector('astra-order-widget');
  el.addEventListener('widget:ready', async () => {
    const api = el.__ASTRA_WIDGET_API__;
    api.setFormData({ clientPhone: '79990000000', description: 'Нужен мастер' });
    if (api.isValid()) {
      const res = await api.submit();
      console.log('Создан заказ:', res.orderId);
    }
  }, { once: true });
</script>
```

---

## Конфигурация через атрибуты

Атрибуты указываются на теге `astra-order-widget`. Большинство можно менять «на лету» - виджет отслеживает изменения.

| Атрибут | Тип/значения | По умолчанию | Назначение |
|---|---|---|---|
| `api-key` | string | - | Публичный ключ виджета. Либо задайте через `window.ASTRA_WIDGET_PUBLIC_KEY`. Без ключа будет ошибка “Missing widget public key”. |
| `api-url` | URL (полный, с `/api/v1`) | `https://api.astracrm.pro/api/v1` | Переопределить базовый URL API. Для атрибута `/api/v1` не добавляется автоматически - укажите его сами. |
| `mode` | `floating` · `embedded` · `headless` | `floating` | Режим работы.|
| `headless` | boolean (`""`, `true`, `1`, `yes`) | false | Альтернативный флаг headless-режима (аналог `mode="headless"`). |
| `theme` | `light` · `dark` · `auto` | `light` | Цветовая тема. `auto` - автоматический выбор по системной теме (prefers-color-scheme). |
| `locale` | `en` · `ru` | `en` | Язык UI/сообщений. |
| `required-fields` | CSV ключей формы (`clientPhone,description,...`) | `clientPhone,description` | Какие поля обязательны дополнительно. Валидация всё равно требует валидный `serviceId`. |
| `position` | см. список позиций | `bottom-right` | Позиция плавающей кнопки (режим `floating`). |
| `button-text` | string | автотекст по `locale` | Текст на кнопке (режим `floating`). |
| `button-icon` | `message` · `users` · `star` · `clock` · `phone` · `chat`/`message-square` · `map-pin` · `briefcase` · `user` | `message` | Иконка кнопки (режим `floating`). |
| `widget-title` | string | автотекст | Заголовок формы (актуально для `embedded`/`floating` экрана приветствия). |
| `widget-subtitle` | string | автотекст | Подзаголовок формы. |
| `order-state` | `draft` · `distributing` | `draft` | Статус заказа: `draft` = только сохранён, `distributing` = сразу отправлен работникам. |
| `debug` | `true` · `false` | `false` | Подробные логи событий и DOM‑ивенты. |

Приоритет: явные атрибуты на теге > глобальные переменные `window.*`. Настройки виджета, приходящие с бэкенда (заголовок/подзаголовок/кнопка, mode/position, order-state, required-fields, locale), используются только как значения по умолчанию; если задали атрибут или меняете через headless `updateConfig`, берётся ваше значение. Брендинг/цвета и каталог всегда приходят с бэкенда.

Только при инициализации: `mode`, `headless`, `debug` читаются один раз. Реактивно отслеживаются: `api-key`, `api-url`, `locale`, `widget-title`, `widget-subtitle`, `theme`, `required-fields`, `position`, `button-text`, `button-icon`, `order-state`.

---

## Конфигурация

Всё работает из коробки. Просто скопируйте код из интерфейса AstraCRM (Виджеты → Встраивание → Код вставки) и вставьте. Виджет сам подключится к `https://api.astracrm.pro/api/v1`.

Если задаёте `api-url` вручную, укажите полный базовый URL сразу с `/api/v1` - атрибут не дописывает путь автоматически. Всегда задайте `api-key` (или `window.ASTRA_WIDGET_PUBLIC_KEY`) до загрузки скрипта.

---

## Статус заказа

- **`draft`** (по умолчанию): Заказ сохраняется, но работникам не отправляется. Удобно, если хотите проверить перед распределением.
- **`distributing`**: Заказ сразу уходит доступным работникам. Они получают уведомления и могут взять заказ в работу.

---

## Безопасность и сеть

Публичный ключ не секретный - он и должен быть в HTML. Все запросы идут по HTTPS, CORS обрабатывается автоматически на нашей стороне.

### Домены с кириллицей и поддомены

- Поддерживаем IDN-домены: можно указывать `https://терпение_и_труд.рф` - внутри он будет сохранён в punycode (`https://xn--...`).
- Если у вас много поддоменов (городов и т.п.), добавьте один origin с подстановкой поддомена: `https://*.терпение_и_труд.рф` - CORS будет работать для всех поддоменов этого домена.

---

## Локализация

Поставьте `locale="en"` или `locale="ru"` - язык интерфейса переключится. Заголовки всегда можно переопределить через `widget-title` и `widget-subtitle`.

---

## Стилизация и темы

Используйте `theme="light"`, `theme="dark"` или `theme="auto"` (авто подстраивается под системную тему). Нужна более тонкая настройка? Переопределите эти CSS-переменные:

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

## Поля формы и валидация

Что валидируется на клиенте (ошибка при отправке):
- `clientPhone`: обязателен, регекс `^7\\d{10}$` (ровно 11 цифр, начинается с `7`).
- `description`: обязателен, 1–1000 символов.
- `serviceId`: обязателен, UUID-подобный.
- `subServiceId`: опционально, UUID-подобный.
- `address`: опционально, ≤500 символов.
- `addressSuggestion`: опционально, прокидывается как есть; если есть - маппится в `addresses` в запросе (предпочтительно для точного адреса).
- `categoryOnly`: опционально; если true, подкатегория не отправляется.

`required-fields` добавляет только фронтовую проверку “заполните поля” (по умолчанию `clientPhone,description`); при этом валидация всё равно отвергнет пустой/невалидный `serviceId`.

---

## Процесс отправки

Виджет проходит через состояния: `validating → submitting → processing → success|error`.

При успехе вы получите `orderId` в ответе.

---

## DOM события

Виджет выбрасывает кастомные события на сам элемент `astra-order-widget` (они всплывают вверх):

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

Пример:

```js
const el = document.querySelector('astra-order-widget');
el.addEventListener('widget:submit-success', (e) => {
  console.log('Заказ создан:', e.detail.orderId);
});
```

Включите `debug="true"`, чтобы видеть подробные логи событий в консоли.

---

## Headless API - краткая справка

После того как виджет готов, API доступен как `el.__ASTRA_WIDGET_API__`.

Что можно делать:
- **Конфигурация**: `getConfig()`, `updateConfig(partial)`
- **Данные формы**: `getFormData()`, `setFormData(partial)`, `clearFormData()`
- **Валидация**: `validate(field?)`, `isValid()`
- **Состояния**: `getState()`, `getCurrentStep()`, `goToStep(step)`
- **Отправка**: `submit()` возвращает `{ orderId, status, message, data? }`, `reset()`
- **Сервисы**: `loadServices()`, `selectService(categoryId, subcategoryId?)`, `getSelectedService()`
- **Адреса**: `suggestAddress(query)`
- **События**: `on`, `off`, `emit`
- **Очистка**: `destroy()` (заглушка)


---

## Решение проблем

- **"Missing widget public key"** - Добавьте атрибут `api-key` или установите `window.ASTRA_WIDGET_PUBLIC_KEY` до загрузки скрипта.
- **Встроенный виджет не виден** - Проверьте, что контейнер имеет ширину и не скрыт CSS.
- **Иконка кнопки не та** - Работают только эти иконки: `message`, `users`, `star`, `clock`, `phone`, `chat`/`message-square`, `map-pin`, `briefcase`, `user`.
- **Ошибки CORS** - Убедитесь, что ваш домен добавлен в белый список в настройках CRM и используется HTTPS.

---

## Поддержка браузеров

Работает в Chrome 90+, Firefox 88+, Safari 14+, Edge 90+ и мобильных браузерах (iOS Safari 14+, Chrome Mobile 90+).

---

## Готовые сниппеты

Скопируйте из интерфейса AstraCRM (Виджеты → Встраивание → Код вставки):

```html
<script>
  window.ASTRA_WIDGET_PUBLIC_KEY = 'pk_live_xxxxx';
</script>
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.umd.js"></script>

<!-- Плавающая кнопка -->
<astra-order-widget mode="floating" position="bottom-right" button-text="Оставить заявку"></astra-order-widget>

<!-- Встроенная форма -->
<astra-order-widget mode="embedded" theme="light" widget-title="Оставить заявку" widget-subtitle="Мы свяжемся с вами"></astra-order-widget>
```

Для headless режима без стилей:

```html
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.headless.umd.js"></script>
<astra-order-widget mode="headless" api-key="..."></astra-order-widget>
```

Используйте, если нужен только программный API без загрузки стилей.

---

## Нужна помощь?

- Вопросы по интеграции: support@astracrm.pro
- Нашли баг или есть идеи? Откройте issue в репозитории документации

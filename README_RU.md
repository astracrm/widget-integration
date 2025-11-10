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
  // Опционально: свой API URL
  // window.ASTRA_WIDGET_API_BASE_URL = 'https://api.astracrm.pro';
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

Доступные иконки: `message` (по умолчанию), `users`, `star`, `clock`.

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

Минимальный пример:
```html
<script src="https://cdn.astracrm.pro/widget/v1/astra-widget.headless.umd.js"></script>
<astra-order-widget mode="headless" api-key="..."></astra-order-widget>
<script>
  const el = document.querySelector('astra-order-widget');
  el.addEventListener('widget:ready', async () => {
    const api = el.__ASTRA_WIDGET_API__;
    api.setFormData({ clientPhone: '+79990000000', description: 'Нужен мастер' });
    if (api.isValid()) {
      const res = await api.submit();
      console.log('Создан заказ:', res.orderId);
    }
  }, { once: true });
<\/script>
```

---

## Конфигурация через атрибуты

Атрибуты указываются на теге `astra-order-widget`. Большинство можно менять «на лету» - виджет отслеживает изменения.

| Атрибут | Тип/значения | По умолчанию | Назначение |
|---|---|---|---|
| `api-key` | string | - | Публичный ключ виджета. Либо задайте через `window.ASTRA_WIDGET_PUBLIC_KEY`. |
| `api-url` | URL | `https://api.astracrm.pro/api/v1` (уже настроено) | Переопределить базовый URL API при необходимости. |
| `mode` | `floating` · `embedded` · `headless` | `floating` | Режим работы.|
| `theme` | `light` · `dark` · `auto` | `light` | Цветовая тема. `auto` - автоматический выбор по системной теме (prefers-color-scheme). |
| `locale` | `en` · `ru` | `en` | Язык UI/сообщений. |
| `required-fields` | CSV ключей формы (`clientPhone,description,...`) | `clientPhone,description` | Какие поля обязательны. |
| `position` | см. список позиций | `bottom-right` | Позиция плавающей кнопки (режим `floating`). |
| `button-text` | string | автотекст по `locale` | Текст на кнопке (режим `floating`). |
| `button-icon` | `message` · `users` · `star` · `clock` · `phone` · `message-square` · `map-pin` · `briefcase` · `user` | `message` | Иконка кнопки (режим `floating`). |
| `widget-title` | string | автотекст | Заголовок формы (актуально для `embedded`/`floating` экрана приветствия). |
| `widget-subtitle` | string | автотекст | Подзаголовок формы. |
| `order-state` | `draft` · `distributing` | `draft` | Статус заказа: `draft` = только сохранён, `distributing` = сразу отправлен работникам. |
| `debug` | `true` · `false` | `false` | Подробные логи событий и DOM‑ивенты. |

Приоритет: явные атрибуты на теге > глобальные переменные `window.*`.

Атрибуты только при инициализации: `mode`, `headless` и `debug` читаются один раз при старте. Дальнейшие изменения не применяются автоматически.

---

## Конфигурация

Всё работает из коробки. Просто скопируйте код из интерфейса AstraCRM (Виджеты → Встраивание → Код вставки) и вставьте. Виджет сам подключится к `https://api.astracrm.pro/api/v1`.

Нужно что-то переопределить? Используйте `window.ASTRA_WIDGET_PUBLIC_KEY` или `window.ASTRA_WIDGET_API_BASE_URL` до загрузки скрипта.

---

## Статус заказа

- **`draft`** (по умолчанию): Заказ сохраняется, но работникам не отправляется. Удобно, если хотите проверить перед распределением.
- **`distributing`**: Заказ сразу уходит доступным работникам. Они получают уведомления и могут взять заказ в работу.

---

## Безопасность и сеть

Публичный ключ не секретный - он и должен быть в HTML. Все запросы идут по HTTPS, CORS обрабатывается автоматически на нашей стороне.

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

Что отправляется:
- `clientPhone`: обязателен, формат `^((+?7)|8)?\d{10}$` (поддерживает +7/7/8 + 10 цифр)
- `description`: обязателен, 1-1000 символов
- `serviceId`: обязателен, UUID формат
- `subServiceId`: опционально, UUID формат
- `categoryOnly`: опционально, булево значение
- `address`: опционально, максимум 500 символов
- `addressSuggestion`: опционально, структурированный адрес из DaData - используется для формирования `addresses`

Атрибут `required-fields` позволяет сделать поля обязательными или опциональными на вашей стороне.

---

## Процесс отправки

Виджет проходит через состояния: `validating → submitting → processing → success|error`.

При успехе вы получите `orderId` в ответе.

---

## DOM события

Виджет выбрасывает кастомные события на сам элемент `astra-order-widget` (они всплывают вверх):

- `widget:init` - виджет инициализируется
- `widget:ready` - API готов к использованию
- `widget:destroy` - виджет удаляется
- `widget:form-change` - данные формы изменились
- `widget:validation-change` - ошибки валидации обновились
- `widget:submit-start` - началась отправка
- `widget:submit-success` - заказ успешно создан
- `widget:submit-error` - ошибка при отправке
- `widget:state-change` - внутреннее состояние изменилось
- `widget:step-change` - сменился шаг формы
- `widget:error` - что-то пошло не так

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
- **События**: `on`, `off`, `emit`
- **Очистка**: `destroy()`


---

## Решение проблем

- **"Missing widget public key"** - Добавьте атрибут `api-key` или установите `window.ASTRA_WIDGET_PUBLIC_KEY` до загрузки скрипта.
- **Встроенный виджет не виден** - Проверьте, что контейнер имеет ширину и не скрыт CSS.
- **Иконка кнопки не та** - Работают только эти иконки: `message`, `users`, `star`, `clock`.
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
  window.ASTRA_WIDGET_API_BASE_URL = 'https://api.astracrm.com';
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

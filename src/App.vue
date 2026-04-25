<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'

const botName = import.meta.env.VITE_TELEGRAM_BOT_NAME?.trim() ?? ''
const apiBaseUrl = import.meta.env.VITE_API_BASE_URL?.trim() ?? 'https://api.example.com'
const query = new URLSearchParams(window.location.search)
const authStatus = query.get('status') ?? ''
const authReason = query.get('reason') ?? ''
const authToken = query.get('token') ?? ''
const authTelegramId = query.get('telegram_id') ?? ''

const telegramWidget = ref(null)
let telegramScript = null

const hasTelegramWidget = computed(() => Boolean(botName))
const telegramBotLink = computed(() => (botName ? `https://t.me/${botName}` : 'https://telegram.org/'))

const stats = [
  { value: '24/7', label: 'Слежение за ценами на Kaspi', tone: 'green' },
  { value: '< 1 мин', label: 'Доставка уведомлений в Telegram', tone: 'blue' },
  { value: 'AI+', label: 'Подсказки по выгодным моментам покупки', tone: 'gold' },
]

const benefits = [
  {
    title: 'Авторизация без пароля',
    text: 'Вход по Telegram за один клик. Пользователь подтверждает себя в знакомом интерфейсе, без регистраций и форм.',
  },
  {
    title: 'Удобно с телефона',
    text: 'Сайт сразу готов для мобильного сценария: открыть ссылку, войти через Telegram и перейти к отслеживанию товара.',
  },
  {
    title: 'Фронт и бэк разделены',
    text: 'Публичная часть живет на GitHub Pages, а Python-бэк на сервере проверяет цены и отправляет уведомления.',
  },
]

const signals = [
  {
    name: 'Apple iPhone 15 Pro 128GB',
    currentPrice: '699 990 ₸',
    minPrice: '639 990 ₸',
    change: '↓ 10 000 ₸',
    updatedAt: 'только что',
  },
  {
    name: 'ASUS ROG Zephyrus G14',
    currentPrice: '899 990 ₸',
    minPrice: '849 990 ₸',
    change: '↓ 15 000 ₸',
    updatedAt: '2 мин назад',
  },
  {
    name: 'Sony WH-1000XM5',
    currentPrice: '149 990 ₸',
    minPrice: '129 990 ₸',
    change: '↓ 5 000 ₸',
    updatedAt: '6 мин назад',
  },
]

onMounted(() => {
  if (authToken) {
    window.localStorage.setItem('ac_price_auth_token', authToken)
  }

  if (!hasTelegramWidget.value || !telegramWidget.value) {
    return
  }

  telegramScript = document.createElement('script')
  telegramScript.async = true
  telegramScript.src = 'https://telegram.org/js/telegram-widget.js?22'
  telegramScript.setAttribute('data-telegram-login', botName)
  telegramScript.setAttribute('data-size', 'large')
  telegramScript.setAttribute('data-radius', '14')
  telegramScript.setAttribute('data-request-access', 'write')
  telegramScript.setAttribute('data-userpic', 'false')
  telegramScript.setAttribute('data-lang', 'ru')
  telegramScript.setAttribute('data-auth-url', `${apiBaseUrl}/auth/telegram/callback`)

  telegramWidget.value.appendChild(telegramScript)
})

onBeforeUnmount(() => {
  if (telegramScript?.parentNode) {
    telegramScript.parentNode.removeChild(telegramScript)
  }
})
</script>

<template>
  <div class="page-shell">
    <aside class="left-rail">
      <div class="brand-card panel">
        <div class="brand-mark">
          <svg viewBox="0 0 24 24" aria-hidden="true">
            <path d="M4 16.5 9 11l3.5 3.5L20 7" />
            <path d="M15.5 7H20v4.5" />
          </svg>
        </div>
        <div>
          <p class="brand-title">AC Price</p>
          <p class="muted">Следим за ценами на Kaspi и сразу пишем в Telegram.</p>
        </div>
      </div>

      <nav class="nav-card panel">
        <a class="nav-link active" href="#hero">Вход</a>
        <a class="nav-link" href="#benefits">Преимущества</a>
        <a class="nav-link" href="#preview">Как это выглядит</a>
      </nav>

      <div class="promo-card panel">
        <p class="promo-label">Telegram-first</p>
        <h2>Откройте доступ к отслеживанию за один клик</h2>
        <p class="muted">
          Без паролей, без форм, без лишних шагов. Пользователь входит через Telegram и
          сразу попадает в дашборд.
        </p>
      </div>
    </aside>

    <main class="content">
      <section v-if="authStatus" class="auth-result panel" :class="authStatus === 'success' ? 'auth-result--success' : 'auth-result--error'">
        <strong>
          {{ authStatus === 'success' ? 'Telegram authorization completed.' : 'Telegram authorization failed.' }}
        </strong>
        <span v-if="authStatus === 'success'">
          Token saved to `localStorage` as `ac_price_auth_token`.
          <template v-if="authTelegramId"> Telegram ID: {{ authTelegramId }}.</template>
        </span>
        <span v-else>{{ authReason || 'Check bot token, domain and callback settings.' }}</span>
      </section>

      <section id="hero" class="hero-grid">
        <div class="hero-copy">
          <span class="eyebrow">Первая страница для неавторизованных</span>
          <h1>Отслеживайте цены на Kaspi и входите через Telegram.</h1>
          <p class="hero-text">
            AC Price помогает не пропустить новые минимальные цены, хранит историю
            изменений и отправляет уведомления прямо в ваш Telegram.
          </p>

          <div class="stats-grid">
            <article
              v-for="item in stats"
              :key="item.label"
              class="stat-card panel"
              :class="`tone-${item.tone}`"
            >
              <strong>{{ item.value }}</strong>
              <span>{{ item.label }}</span>
            </article>
          </div>
        </div>

        <div class="auth-card panel">
          <div class="auth-card__header">
            <span class="status-dot"></span>
            <span>Вход в систему</span>
          </div>

          <h2>Авторизация через Telegram</h2>
          <p class="muted">
            Подтвердите вход через Telegram-бота и получите доступ к отслеживаемым
            товарам, уведомлениям и аналитике.
          </p>

          <div v-if="hasTelegramWidget" ref="telegramWidget" class="telegram-widget"></div>

          <a v-else class="telegram-fallback" :href="telegramBotLink" target="_blank" rel="noreferrer">
            <span class="telegram-icon">
              <svg viewBox="0 0 24 24" aria-hidden="true">
                <path
                  d="M21.2 4.2 18 19.3c-.2 1-.8 1.2-1.7.8l-4.9-3.6-2.4 2.3c-.3.3-.5.5-1 .5l.4-5.1 9.3-8.4c.4-.4-.1-.6-.6-.3L5.6 12.8.8 11.3c-1-.3-1-.9.2-1.4L19.8 2.7c.9-.3 1.7.2 1.4 1.5Z"
                />
              </svg>
            </span>
            <span>Войти через Telegram</span>
          </a>

          <p class="helper-note">
            <template v-if="hasTelegramWidget">
              Виджет активен. После подключения Python-бэка укажите корректный
              `VITE_API_BASE_URL` для валидации Telegram auth.
            </template>
            <template v-else>
              Чтобы включить официальный Telegram Login Widget, добавьте
              `VITE_TELEGRAM_BOT_NAME` в `.env`.
            </template>
          </p>

          <div class="auth-meta">
            <div>
              <span>Бот</span>
              <strong>{{ botName || '@your_auth_bot' }}</strong>
            </div>
            <div>
              <span>Backend</span>
              <strong>{{ apiBaseUrl }}</strong>
            </div>
          </div>
        </div>
      </section>

      <section id="benefits" class="benefits-section">
        <article v-for="item in benefits" :key="item.title" class="benefit-card panel">
          <span class="benefit-index">0{{ benefits.indexOf(item) + 1 }}</span>
          <h3>{{ item.title }}</h3>
          <p class="muted">{{ item.text }}</p>
        </article>
      </section>

      <section id="preview" class="preview panel">
        <div class="preview__header">
          <div>
            <span class="eyebrow">После входа</span>
            <h2>Пользователь попадает в панель с товарами и сигналами цены</h2>
          </div>
          <button type="button" class="ghost-button">Добавить товар</button>
        </div>

        <div class="preview-stats">
          <div class="preview-stat">
            <span>Товаров отслеживается</span>
            <strong>12</strong>
          </div>
          <div class="preview-stat">
            <span>Снижений сегодня</span>
            <strong>7</strong>
          </div>
          <div class="preview-stat">
            <span>Уведомлений отправлено</span>
            <strong>3</strong>
          </div>
          <div class="preview-stat">
            <span>Сэкономлено</span>
            <strong>84 620 ₸</strong>
          </div>
        </div>

        <div class="table-card">
          <div class="table-head">
            <span>Товар</span>
            <span>Текущая цена</span>
            <span>Минимум</span>
            <span>Изменение</span>
            <span>Обновление</span>
          </div>

          <div v-for="item in signals" :key="item.name" class="table-row">
            <div class="product-cell">
              <div class="product-thumb">{{ item.name.slice(0, 2) }}</div>
              <div>
                <strong>{{ item.name }}</strong>
                <p class="muted">kaspi.kz/shop/...</p>
              </div>
            </div>
            <strong>{{ item.currentPrice }}</strong>
            <strong class="price-min">{{ item.minPrice }}</strong>
            <strong class="price-drop">{{ item.change }}</strong>
            <span>{{ item.updatedAt }}</span>
          </div>
        </div>
      </section>
    </main>
  </div>
</template>

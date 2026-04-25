<script setup>
import {
  Bell,
  ChartNoAxesColumn,
  ChevronDown,
  CircleDollarSign,
  House,
  Settings,
  ShoppingBag,
  TrendingUp,
} from 'lucide-vue-next'
import { computed, nextTick, onBeforeUnmount, onMounted, ref, watch } from 'vue'

const STORAGE_KEY = 'ac_price_auth_session_v2'
const LEGACY_STORAGE_KEYS = ['ac_price_auth_token', 'ac_price_auth_session']

const botName = import.meta.env.VITE_TELEGRAM_BOT_NAME?.trim() ?? ''
const apiBaseUrl = import.meta.env.VITE_API_BASE_URL?.trim() ?? 'https://api.example.com'
const query = new URLSearchParams(window.location.search)

const widgetContainer = ref(null)
const authMessage = ref('')
const authError = ref('')
const session = ref(null)
const profile = ref(null)
const profileMenuOpen = ref(false)
const currentPage = ref('home')

const products = ref([])
const isProductsLoading = ref(false)
const productsError = ref('')

const analyticsProducts = ref([])
const isAnalyticsLoading = ref(false)
const analyticsError = ref('')

const isAddModalOpen = ref(false)
const addProductUrl = ref('')
const isSubmittingProduct = ref(false)
const addProductError = ref('')

let telegramScript = null

const hasTelegramWidget = computed(() => Boolean(botName))
const isAuthenticated = computed(() => Boolean(session.value))
const canAccessAnalytics = computed(() => Boolean(profile.value?.can_access_analytics))

const trackedCount = computed(() => products.value.length)
const priceDropsToday = computed(
  () =>
    products.value.filter((item) => {
      const checked = new Date(item.last_checked_at)
      const now = new Date()
      return item.last_price_change < 0 && checked.toDateString() === now.toDateString()
    }).length,
)
const sentNotifications = computed(() => products.value.filter((item) => item.last_price_change < 0).length)
const totalSaved = computed(() =>
  products.value.reduce((sum, item) => sum + Math.max(item.current_price - item.min_price, 0), 0),
)

const stats = computed(() => [
  { icon: 'bag', value: String(trackedCount.value), label: 'Товаров отслеживается', tone: 'green' },
  { icon: 'trend', value: String(priceDropsToday.value), label: 'Снижений сегодня', tone: 'violet' },
  { icon: 'bell', value: String(sentNotifications.value), label: 'Снижений в истории', tone: 'blue' },
  { icon: 'coin', value: formatPrice(totalSaved.value), label: 'Потенциально сэкономлено', tone: 'gold' },
])

const analyticsSummary = computed(() => {
  const items = analyticsProducts.value
  const pointsCount = items.reduce((sum, item) => sum + item.history.length, 0)
  const activeDrops = items.filter((item) => item.current_price <= item.min_price).length
  const averageVolatility = items.length
    ? items.reduce((sum, item) => {
        const base = item.min_price || item.current_price || 1
        return sum + ((Math.abs(item.current_price - item.min_price) / base) * 100)
      }, 0) / items.length
    : 0

  return [
    { title: 'Товаров в аналитике', value: String(items.length) },
    { title: 'Точек истории', value: String(pointsCount) },
    { title: 'На минимуме сейчас', value: String(activeDrops) },
    { title: 'Средняя волатильность', value: `${averageVolatility.toFixed(1)}%` },
  ]
})

const analyticsRows = computed(() =>
  analyticsProducts.value.map((item) => {
    const history = item.history ?? []
    const firstPoint = history[0]
    const lastPoint = history[history.length - 1]
    const fullChange = firstPoint ? item.current_price - firstPoint.price : 0

    return {
      ...item,
      historyCount: history.length,
      fullChange,
      sparkline: buildSparkline(history),
    }
  }),
)

const navItems = [
  { key: 'home', label: 'Главная', icon: House },
  { key: 'analytics', label: 'Аналитика', icon: ChartNoAxesColumn },
  { key: 'notifications', label: 'Уведомления', icon: Bell, disabled: true },
  { key: 'settings', label: 'Настройки', icon: Settings, disabled: true },
]

function decodeJwtPayload(token) {
  try {
    const payload = token.split('.')[1]
    const normalized = payload.replace(/-/g, '+').replace(/_/g, '/')
    const padded = normalized.padEnd(Math.ceil(normalized.length / 4) * 4, '=')
    return JSON.parse(window.atob(padded))
  } catch {
    return null
  }
}

function readStoredSession() {
  try {
    const raw = window.localStorage.getItem(STORAGE_KEY)
    return raw ? JSON.parse(raw) : null
  } catch {
    return null
  }
}

function persistSession(token) {
  const payload = decodeJwtPayload(token)
  if (!payload) {
    return
  }

  const nextSession = {
    token,
    username: payload.username || '',
    telegramId: payload.telegram_id || '',
    userId: payload.sub || '',
  }

  session.value = nextSession
  window.localStorage.setItem(STORAGE_KEY, JSON.stringify(nextSession))
}

function getAuthHeaders() {
  if (!session.value?.token) {
    return {}
  }
  return {
    Authorization: `Bearer ${session.value.token}`,
    'Content-Type': 'application/json',
  }
}

function formatPrice(value) {
  return `${new Intl.NumberFormat('ru-RU').format(value)} ₸`
}

function formatShortDate(value) {
  return new Intl.DateTimeFormat('ru-RU', {
    day: 'numeric',
    month: 'short',
    year: 'numeric',
  }).format(new Date(value))
}

function formatRelativeTime(value) {
  const date = new Date(value)
  const diffMinutes = Math.max(1, Math.round((Date.now() - date.getTime()) / 60000))
  if (diffMinutes < 60) {
    return `${diffMinutes} мин назад`
  }
  const diffHours = Math.round(diffMinutes / 60)
  if (diffHours < 24) {
    return `${diffHours} ч назад`
  }
  const diffDays = Math.round(diffHours / 24)
  return `${diffDays} дн назад`
}

function buildThumb(name) {
  return name
    .split(/\s+/)
    .slice(0, 2)
    .map((part) => part[0] || '')
    .join('')
    .toUpperCase()
}

function smartShortenProductName(name, maxLength = 54) {
  const normalized = name.replace(/\s+/g, ' ').trim()
  if (normalized.length <= maxLength) {
    return normalized
  }

  const separators = [' / ', ',', ' - ']
  for (const separator of separators) {
    const index = normalized.indexOf(separator)
    if (index > 18) {
      const head = normalized.slice(0, index).trim()
      if (head.length <= maxLength) {
        return `${head}...`
      }
    }
  }

  const words = normalized.split(' ')
  let result = ''
  for (const word of words) {
    const next = result ? `${result} ${word}` : word
    if (next.length > maxLength - 3) {
      break
    }
    result = next
  }

  return result ? `${result}...` : `${normalized.slice(0, maxLength - 3)}...`
}

function buildSparkline(history) {
  if (!history?.length) {
    return ''
  }

  const values = history.map((item) => item.price)
  const min = Math.min(...values)
  const max = Math.max(...values)
  const range = max - min || 1

  return history
    .map((item, index) => {
      const x = (index / Math.max(history.length - 1, 1)) * 100
      const y = 100 - ((item.price - min) / range) * 100
      return `${x},${y}`
    })
    .join(' ')
}

async function loadProfile() {
  if (!session.value?.token) {
    profile.value = null
    return
  }

  const response = await fetch(`${apiBaseUrl}/api/v1/me`, {
    headers: getAuthHeaders(),
  })
  const payload = await response.json()
  if (!response.ok) {
    throw new Error(payload.detail || 'Не удалось загрузить профиль.')
  }
  profile.value = payload
}

async function loadProducts() {
  if (!session.value?.token) {
    products.value = []
    return
  }

  isProductsLoading.value = true
  productsError.value = ''
  try {
    const response = await fetch(`${apiBaseUrl}/api/v1/products`, {
      headers: getAuthHeaders(),
    })
    const payload = await response.json()
    if (!response.ok) {
      throw new Error(payload.detail || 'Не удалось загрузить товары.')
    }
    products.value = payload.items
  } catch (error) {
    productsError.value = error.message || 'Не удалось загрузить товары.'
  } finally {
    isProductsLoading.value = false
  }
}

async function loadAnalytics() {
  if (!session.value?.token || !canAccessAnalytics.value) {
    analyticsProducts.value = []
    analyticsError.value = ''
    return
  }

  isAnalyticsLoading.value = true
  analyticsError.value = ''
  try {
    const response = await fetch(`${apiBaseUrl}/api/v1/products/analytics`, {
      headers: getAuthHeaders(),
    })
    const payload = await response.json()
    if (!response.ok) {
      throw new Error(payload.detail || 'Не удалось загрузить аналитику.')
    }
    analyticsProducts.value = payload.items
  } catch (error) {
    analyticsError.value = error.message || 'Не удалось загрузить аналитику.'
  } finally {
    isAnalyticsLoading.value = false
  }
}

function openAddModal() {
  addProductUrl.value = ''
  addProductError.value = ''
  isAddModalOpen.value = true
}

function closeAddModal() {
  isAddModalOpen.value = false
  addProductUrl.value = ''
  addProductError.value = ''
}

async function submitProduct() {
  if (!addProductUrl.value.trim()) {
    addProductError.value = 'Вставьте ссылку на товар.'
    return
  }

  isSubmittingProduct.value = true
  addProductError.value = ''
  try {
    const response = await fetch(`${apiBaseUrl}/api/v1/products`, {
      method: 'POST',
      headers: getAuthHeaders(),
      body: JSON.stringify({ url: addProductUrl.value.trim() }),
    })
    const payload = await response.json()
    if (!response.ok) {
      throw new Error(payload.detail || 'Не удалось добавить товар.')
    }

    const existingIndex = products.value.findIndex((item) => item.id === payload.id)
    if (existingIndex >= 0) {
      products.value.splice(existingIndex, 1, payload)
    } else {
      products.value.unshift(payload)
    }

    if (currentPage.value === 'analytics' && canAccessAnalytics.value) {
      await loadAnalytics()
    }

    if (profile.value) {
      profile.value.products_used += existingIndex >= 0 ? 0 : 1
    }

    closeAddModal()
  } catch (error) {
    addProductError.value = error.message || 'Не удалось добавить товар.'
  } finally {
    isSubmittingProduct.value = false
  }
}

function cleanupTelegramWidget() {
  if (telegramScript?.parentNode) {
    telegramScript.parentNode.removeChild(telegramScript)
  }
  telegramScript = null

  if (widgetContainer.value) {
    widgetContainer.value.innerHTML = ''
  }
}

function renderTelegramWidget() {
  cleanupTelegramWidget()

  if (!hasTelegramWidget.value || !widgetContainer.value || isAuthenticated.value) {
    return
  }

  telegramScript = document.createElement('script')
  telegramScript.async = true
  telegramScript.src = 'https://telegram.org/js/telegram-widget.js?22'
  telegramScript.setAttribute('data-telegram-login', botName)
  telegramScript.setAttribute('data-size', 'large')
  telegramScript.setAttribute('data-radius', '16')
  telegramScript.setAttribute('data-request-access', 'write')
  telegramScript.setAttribute('data-userpic', 'false')
  telegramScript.setAttribute('data-lang', 'ru')
  telegramScript.setAttribute('data-auth-url', `${apiBaseUrl}/auth/telegram/callback`)

  widgetContainer.value.appendChild(telegramScript)
}

function clearUrlState() {
  if (window.location.search) {
    window.history.replaceState({}, document.title, window.location.pathname)
  }
}

function logout() {
  window.localStorage.removeItem(STORAGE_KEY)
  session.value = null
  profile.value = null
  profileMenuOpen.value = false
  authMessage.value = ''
  authError.value = ''
  products.value = []
  analyticsProducts.value = []
  productsError.value = ''
  analyticsError.value = ''
  currentPage.value = 'home'
  clearUrlState()
}

function toggleProfileMenu() {
  profileMenuOpen.value = !profileMenuOpen.value
}

async function openPage(pageKey) {
  if (pageKey === 'notifications' || pageKey === 'settings') {
    return
  }
  currentPage.value = pageKey
  profileMenuOpen.value = false
  if (pageKey === 'analytics' && canAccessAnalytics.value && !analyticsProducts.value.length) {
    await loadAnalytics()
  }
}

onMounted(async () => {
  for (const key of LEGACY_STORAGE_KEYS) {
    window.localStorage.removeItem(key)
  }

  const status = query.get('status') ?? ''
  const token = query.get('token') ?? ''
  const reason = query.get('reason') ?? ''

  if (status === 'success' && token) {
    persistSession(token)
    authMessage.value = 'Авторизация выполнена.'
    authError.value = ''
    clearUrlState()
  } else if (status === 'error') {
    authError.value = reason || 'Не удалось войти через Telegram.'
    window.localStorage.removeItem(STORAGE_KEY)
    session.value = null
    clearUrlState()
  } else {
    session.value = readStoredSession()
  }

  await nextTick()
  renderTelegramWidget()
  if (session.value?.token) {
    try {
      await loadProfile()
      await loadProducts()
    } catch (error) {
      authError.value = error.message || 'Не удалось загрузить данные аккаунта.'
    }
  }
})

watch(isAuthenticated, async () => {
  await nextTick()
  renderTelegramWidget()
  if (isAuthenticated.value) {
    try {
      await loadProfile()
      await loadProducts()
    } catch (error) {
      authError.value = error.message || 'Не удалось загрузить данные аккаунта.'
    }
  }
})

onBeforeUnmount(() => {
  cleanupTelegramWidget()
})
</script>

<template>
  <main v-if="!isAuthenticated" class="auth-screen">
    <section class="login-card">
      <div class="login-badge">
        <span class="login-badge__dot"></span>
        <span>AC Price</span>
      </div>

      <h1>Вход через Telegram</h1>
      <p class="login-subtitle">Быстрый вход без пароля для доступа к отслеживаемым товарам.</p>

      <div v-if="authError" class="state state-error">{{ authError }}</div>
      <div v-else-if="authMessage" class="state state-success">{{ authMessage }}</div>

      <div class="login-widget-wrap">
        <div v-if="hasTelegramWidget" ref="widgetContainer" class="telegram-widget"></div>
        <a
          v-else
          class="fallback-button"
          :href="botName ? `https://t.me/${botName}` : 'https://telegram.org/'"
          target="_blank"
          rel="noreferrer"
        >
          Войти через Telegram
        </a>
      </div>
    </section>
  </main>

  <main v-else class="dashboard">
    <aside class="sidebar panel">
      <div>
        <div class="brand-card">
          <div class="brand-mark">
            <TrendingUp aria-hidden="true" />
          </div>
          <div>
            <p class="brand-title">AC Price</p>
            <p class="brand-subtitle">Следим за ценами на Kaspi</p>
          </div>
        </div>

        <nav class="sidebar-nav">
          <button
            v-for="item in navItems"
            :key="item.key"
            type="button"
            class="nav-item"
            :class="{ active: currentPage === item.key, muted: item.disabled }"
            @click="openPage(item.key)"
          >
            <component :is="item.icon" aria-hidden="true" />
            <span>{{ item.label }}</span>
          </button>
        </nav>

        <div class="premium-card">
          <p class="premium-card__title">Премиум</p>
          <p class="premium-card__text">Больше отслеживаний и расширенная аналитика</p>
          <button type="button" class="premium-card__button">Узнать больше</button>
        </div>
      </div>

      <div class="profile-anchor">
        <button type="button" class="profile-chip" @click="toggleProfileMenu">
          <span class="profile-chip__avatar">
            {{ (session?.username || 'U').slice(0, 2).toUpperCase() }}
          </span>
          <span class="profile-chip__info">
            <strong>{{ session?.username || 'user' }}</strong>
            <span>{{ profile?.tariff_label || 'Бесплатный' }}</span>
          </span>
          <span class="profile-chip__arrow" :class="{ open: profileMenuOpen }">
            <ChevronDown aria-hidden="true" />
          </span>
        </button>

        <div v-if="profileMenuOpen" class="profile-menu">
          <button type="button" class="profile-menu__item" @click="logout">Выйти</button>
        </div>
      </div>
    </aside>

    <section class="dashboard-main">
      <template v-if="currentPage === 'home'">
        <header class="dashboard-header">
          <div>
            <h1 class="dashboard-title">Мои товары</h1>
            <p class="dashboard-subtitle">
              Отслеживайте цены на товары и получайте уведомления о снижении цен
            </p>
          </div>
          <button type="button" class="add-button" @click="openAddModal">Добавить товар</button>
        </header>

        <section class="stats-bar panel">
          <article v-for="item in stats" :key="item.label" class="stat-tile">
            <div class="stat-tile__icon" :class="`tone-${item.tone}`">
              <ShoppingBag v-if="item.icon === 'bag'" aria-hidden="true" />
              <TrendingUp v-else-if="item.icon === 'trend'" aria-hidden="true" />
              <Bell v-else-if="item.icon === 'bell'" aria-hidden="true" />
              <CircleDollarSign v-else aria-hidden="true" />
            </div>
            <div>
              <strong>{{ item.value }}</strong>
              <p>{{ item.label }}</p>
            </div>
          </article>
        </section>

        <section class="products-card panel">
          <div class="products-head">
            <div>
              <h2>Отслеживаемые товары</h2>
            </div>
            <div class="toolbar">
              <div class="search-box">Поиск по товарам...</div>
              <button type="button" class="sort-button">Сортировка</button>
            </div>
          </div>

          <div class="table-head">
            <span>Товар</span>
            <span>Текущая цена</span>
            <span>Минимальная цена</span>
            <span>Изменение</span>
            <span>Последнее обновление</span>
          </div>

          <div class="table-scroll">
            <div v-if="isProductsLoading" class="empty-state">Загрузка товаров...</div>
            <div v-else-if="productsError" class="empty-state empty-state--error">{{ productsError }}</div>
            <div v-else-if="!products.length" class="empty-state">
              Пока нет товаров. Нажмите «Добавить товар» и вставьте ссылку на Kaspi.
            </div>
            <article v-for="item in products" :key="item.id" class="table-row">
              <div class="product-main">
                <div class="product-thumb">{{ buildThumb(item.product_name) }}</div>
                <div class="product-copy">
                  <strong :title="item.product_name">{{ smartShortenProductName(item.product_name) }}</strong>
                  <a :href="item.url" target="_blank" rel="noreferrer">{{ item.url }}</a>
                </div>
              </div>
              <strong class="mono">{{ formatPrice(item.current_price) }}</strong>
              <div>
                <strong class="price-min mono">{{ formatPrice(item.min_price) }}</strong>
                <p>{{ formatShortDate(item.min_price_recorded_at) }}</p>
              </div>
              <div>
                <strong class="price-change mono" :class="{ down: item.last_price_change < 0 }">
                  {{ item.last_price_change < 0 ? '↓' : item.last_price_change > 0 ? '↑' : '•' }}
                  {{ formatPrice(Math.abs(item.last_price_change || 0)) }}
                </strong>
                <p>
                  {{
                    item.current_price
                      ? `(${((Math.abs(item.last_price_change || 0) / item.current_price) * 100).toFixed(2)}%)`
                      : '(0.00%)'
                  }}
                </p>
              </div>
              <span class="updated">{{ formatRelativeTime(item.last_checked_at) }}</span>
            </article>
          </div>
        </section>
      </template>

      <template v-else-if="currentPage === 'analytics'">
        <header class="dashboard-header">
          <div>
            <h1 class="dashboard-title">Аналитика</h1>
            <p class="dashboard-subtitle">
              Динамика цен, глубина истории и состояние каждого отслеживаемого товара
            </p>
          </div>
        </header>

        <section v-if="!canAccessAnalytics" class="analytics-lock panel">
          <div class="premium-card premium-card--centered">
            <p class="premium-card__title">Премиум</p>
            <p class="premium-card__text">Больше отслеживаний и расширенная аналитика</p>
            <button type="button" class="premium-card__button">Узнать больше</button>
          </div>
        </section>

        <template v-else>
          <section class="analytics-summary">
            <article v-for="item in analyticsSummary" :key="item.title" class="analytics-summary__card panel">
              <span>{{ item.title }}</span>
              <strong>{{ item.value }}</strong>
            </article>
          </section>

          <section class="analytics-board panel">
            <div v-if="isAnalyticsLoading" class="empty-state">Загрузка аналитики...</div>
            <div v-else-if="analyticsError" class="empty-state empty-state--error">{{ analyticsError }}</div>
            <div v-else-if="!analyticsRows.length" class="empty-state">Пока нет данных для аналитики.</div>

            <article v-for="item in analyticsRows" :key="item.id" class="analytics-row">
              <div class="analytics-row__main">
                <div class="product-thumb">{{ buildThumb(item.product_name) }}</div>
                <div class="analytics-row__copy">
                  <strong :title="item.product_name">{{ smartShortenProductName(item.product_name, 48) }}</strong>
                  <span>{{ item.historyCount }} точек истории</span>
                </div>
              </div>

              <div class="analytics-row__chart">
                <svg v-if="item.sparkline" viewBox="0 0 100 100" preserveAspectRatio="none" aria-hidden="true">
                  <polyline :points="item.sparkline" />
                </svg>
              </div>

              <div class="analytics-row__stats">
                <div>
                  <span>Текущая</span>
                  <strong>{{ formatPrice(item.current_price) }}</strong>
                </div>
                <div>
                  <span>Минимум</span>
                  <strong class="price-min">{{ formatPrice(item.min_price) }}</strong>
                </div>
                <div>
                  <span>За период</span>
                  <strong :class="['price-change', { down: item.fullChange < 0 }]">
                    {{ item.fullChange < 0 ? '↓' : item.fullChange > 0 ? '↑' : '•' }}
                    {{ formatPrice(Math.abs(item.fullChange)) }}
                  </strong>
                </div>
              </div>
            </article>
          </section>
        </template>
      </template>

      <template v-else>
        <section class="analytics-lock panel">
          <div class="empty-state">Раздел пока в разработке.</div>
        </section>
      </template>
    </section>

    <div v-if="isAddModalOpen" class="modal-backdrop" @click.self="closeAddModal">
      <section class="modal-card panel">
        <div class="modal-card__head">
          <h3>Добавление товара</h3>
          <button type="button" class="modal-close" @click="closeAddModal">✕</button>
        </div>

        <label class="modal-field">
          <span>Ссылка на товар</span>
          <input v-model="addProductUrl" type="url" placeholder="https://..." />
        </label>

        <p v-if="addProductError" class="modal-error">{{ addProductError }}</p>

        <div class="modal-actions">
          <button type="button" class="sort-button" @click="closeAddModal">Отмена</button>
          <button type="button" class="add-button" @click="submitProduct">
            {{ isSubmittingProduct ? 'Добавление...' : 'Добавить' }}
          </button>
        </div>
      </section>
    </div>
  </main>
</template>

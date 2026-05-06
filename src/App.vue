<script setup>
import AnalyticsMiniChart from './components/AnalyticsMiniChart.vue'
import {
  Bell,
  BellRing,
  ChartNoAxesColumn,
  Check,
  ChevronDown,
  CircleDollarSign,
  House,
  MessageCircleMore,
  Monitor,
  Settings,
  ShieldCheck,
  ShoppingBag,
  SlidersHorizontal,
  Trash2,
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
const notificationSelectRef = ref(null)

const products = ref([])
const isProductsLoading = ref(false)
const productsError = ref('')
const deletingProductId = ref(null)
const swipedProductId = ref(null)
const swipeStartX = ref(0)
const swipeOffsetX = ref(0)

const analyticsProducts = ref([])
const isAnalyticsLoading = ref(false)
const analyticsError = ref('')

const isAddModalOpen = ref(false)
const addProductUrl = ref('')
const isSubmittingProduct = ref(false)
const addProductError = ref('')
const notificationChannel = ref('telegram')
const notificationRule = ref('every_change')
const notificationAmount = ref(5000)
const browserSoundEnabled = ref(true)
const quietHoursEnabled = ref(false)
const outOfStockAlerts = ref(true)
const digestEnabled = ref(false)
const notificationSelectOpen = ref(false)
const notificationsReady = ref(false)

let telegramScript = null
let browserNotificationTimer = null
let notificationSelectListenerAttached = false

const notificationOptions = [
  { value: 'every_change', label: 'При каждом изменении', hint: 'Сигнал сразу после любой смены цены' },
  { value: 'drop_only', label: 'Только при снижении', hint: 'Без уведомлений о росте цены' },
  { value: 'amount_change', label: 'При изменении на сумму', hint: 'Срабатывает только после заданного порога' },
]

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
  products.value.reduce((sum, item) => {
    if (!item.is_available) {
      return sum
    }
    return sum + Math.max((item.initial_price ?? item.current_price) - item.current_price, 0)
  }, 0),
)
const netCartChange = computed(() =>
  products.value.reduce((sum, item) => {
    if (!item.is_available) {
      return sum
    }
    return sum + ((item.current_price ?? 0) - (item.initial_price ?? item.current_price ?? 0))
  }, 0),
)

const stats = computed(() => [
  { icon: 'bag', value: String(trackedCount.value), label: 'Товаров отслеживается', tone: 'green' },
  { icon: 'trend', value: String(priceDropsToday.value), label: 'Снижений сегодня', tone: 'violet' },
  { icon: 'coin', value: formatPrice(totalSaved.value), label: 'Потенциально сэкономлено', tone: 'gold' },
  {
    icon: 'chart',
    value: `${netCartChange.value > 0 ? '+' : netCartChange.value < 0 ? '-' : ''}${formatPrice(Math.abs(netCartChange.value))}`,
    label: netCartChange.value > 0 ? 'Переплата сейчас' : netCartChange.value < 0 ? 'Снижение корзины' : 'Без изменений',
    tone: netCartChange.value > 0 ? 'red' : netCartChange.value < 0 ? 'green' : 'blue',
  },
])

const latestCheckedAt = computed(() => {
  if (!products.value.length) {
    return null
  }

  return [...products.value]
    .map((item) => item.last_checked_at)
    .sort((left, right) => new Date(right).getTime() - new Date(left).getTime())[0]
})

const productsUpdateHint = computed(() => {
  const schedule =
    profile.value?.refresh_mode === 'every_5_minutes'
      ? 'Обновляется каждые 5 минут.'
      : 'Обновляется ежедневно в 08:00 по Казахстану.'
  const lastCheck = latestCheckedAt.value ? ` Последняя проверка: ${formatRelativeTime(latestCheckedAt.value)}.` : ''
  return `${schedule}${lastCheck}`
})

const analyticsSummary = computed(() => {
  const items = analyticsProducts.value
  const availableItems = items.filter((item) => item.is_available)
  const pointsCount = items.reduce((sum, item) => sum + item.history.length, 0)
  const activeDrops = availableItems.filter((item) => item.current_price <= item.min_price).length
  const averageVolatility = availableItems.length
    ? availableItems.reduce((sum, item) => {
        const base = item.min_price || item.current_price || 1
        return sum + ((Math.abs(item.current_price - item.min_price) / base) * 100)
      }, 0) / availableItems.length
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

    return {
      ...item,
      historyCount: history.length,
    }
  }),
)

const selectedNotificationOption = computed(
  () => notificationOptions.find((item) => item.value === notificationRule.value) ?? notificationOptions[0],
)

const amountChangeLabel = computed(() => formatPrice(notificationAmount.value))

const notificationFact = computed(() =>
  notificationChannel.value === 'telegram'
    ? 'Уведомления будут приходить в Telegram сразу после очередной проверки цены.'
    : 'Браузерные push-уведомления будут приходить в том браузере, где вы разрешили показ уведомлений, даже если сайт уже закрыт.',
)

const navItems = [
  { key: 'home', label: 'Главная', icon: House },
  { key: 'analytics', label: 'Аналитика', icon: ChartNoAxesColumn },
  { key: 'notifications', label: 'Уведомления', icon: Bell },
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

function getOutOfStockLabel() {
  return 'Пока нет в наличии'
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

function getTrackedChange(item) {
  if (!item.is_available) {
    return 0
  }
  return (item.current_price ?? 0) - (item.initial_price ?? item.current_price ?? 0)
}

function getTrackedChangePercent(item) {
  if (!item.is_available) {
    return '0.00'
  }
  const base = item.initial_price ?? 0
  if (!base) {
    return '0.00'
  }
  return ((Math.abs(getTrackedChange(item)) / base) * 100).toFixed(2)
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

function applyNotificationSettings(payload) {
  notificationChannel.value = payload.notification_channel ?? 'telegram'
  notificationRule.value = payload.notification_rule ?? 'every_change'
  notificationAmount.value = payload.notification_amount_threshold ?? 5000
  browserSoundEnabled.value = payload.browser_sound_enabled ?? true
  quietHoursEnabled.value = payload.quiet_hours_enabled ?? false
  outOfStockAlerts.value = payload.out_of_stock_alerts ?? true
  digestEnabled.value = payload.digest_enabled ?? false
}

async function loadNotificationSettings() {
  if (!session.value?.token) {
    notificationsReady.value = false
    return
  }

  const response = await fetch(`${apiBaseUrl}/api/v1/notifications/settings`, {
    headers: getAuthHeaders(),
  })
  const payload = await response.json()
  if (!response.ok) {
    throw new Error(payload.detail || 'Не удалось загрузить настройки уведомлений.')
  }
  applyNotificationSettings(payload)
  notificationsReady.value = true
}

async function saveNotificationSettings() {
  if (!session.value?.token || !notificationsReady.value) {
    return
  }

  await fetch(`${apiBaseUrl}/api/v1/notifications/settings`, {
    method: 'PUT',
    headers: getAuthHeaders(),
    body: JSON.stringify({
      notification_channel: notificationChannel.value,
      notification_rule: notificationRule.value,
      notification_amount_threshold: notificationAmount.value,
      browser_sound_enabled: browserSoundEnabled.value,
      quiet_hours_enabled: quietHoursEnabled.value,
      out_of_stock_alerts: outOfStockAlerts.value,
      digest_enabled: digestEnabled.value,
    }),
  })
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

async function deleteProduct(productId) {
  if (!session.value?.token || deletingProductId.value === productId) {
    return
  }

  deletingProductId.value = productId
  productsError.value = ''
  try {
    const response = await fetch(`${apiBaseUrl}/api/v1/products/${productId}`, {
      method: 'DELETE',
      headers: getAuthHeaders(),
    })

    if (!response.ok) {
      const payload = await response.json().catch(() => ({}))
      throw new Error(payload.detail || 'Не удалось удалить товар.')
    }

    products.value = products.value.filter((item) => item.id !== productId)
    analyticsProducts.value = analyticsProducts.value.filter((item) => item.id !== productId)
    swipedProductId.value = null
    swipeOffsetX.value = 0

    if (profile.value?.products_used) {
      profile.value.products_used = Math.max(profile.value.products_used - 1, 0)
    }
  } catch (error) {
    productsError.value = error.message || 'Не удалось удалить товар.'
  } finally {
    deletingProductId.value = null
  }
}

function isMobileViewport() {
  return typeof window !== 'undefined' && window.innerWidth <= 640
}

function handleProductTouchStart(productId, event) {
  if (!isMobileViewport()) {
    return
  }

  swipedProductId.value = swipedProductId.value === productId ? productId : null
  swipeStartX.value = event.changedTouches[0]?.clientX ?? 0
  swipeOffsetX.value = swipedProductId.value === productId ? -88 : 0
}

function handleProductTouchMove(productId, event) {
  if (!isMobileViewport()) {
    return
  }

  const currentX = event.changedTouches[0]?.clientX ?? swipeStartX.value
  const deltaX = currentX - swipeStartX.value

  if (deltaX < 0) {
    swipedProductId.value = productId
    swipeOffsetX.value = Math.max(deltaX, -88)
  } else if (swipedProductId.value === productId) {
    swipeOffsetX.value = Math.min(-88 + deltaX, 0)
  }
}

function handleProductTouchEnd(productId) {
  if (!isMobileViewport()) {
    return
  }

  if (swipedProductId.value !== productId) {
    swipeOffsetX.value = 0
    return
  }

  if (swipeOffsetX.value <= -44) {
    swipedProductId.value = productId
    swipeOffsetX.value = -88
  } else {
    swipedProductId.value = null
    swipeOffsetX.value = 0
  }
}

function resetProductSwipe() {
  swipedProductId.value = null
  swipeOffsetX.value = 0
}

function getProductRowStyle(productId) {
  if (swipedProductId.value !== productId) {
    return {}
  }

  return {
    transform: `translateX(${swipeOffsetX.value}px)`,
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

async function logout() {
  try {
    await syncPushSubscription(false)
  } catch {}
  stopBrowserNotificationPolling()
  window.localStorage.removeItem(STORAGE_KEY)
  session.value = null
  profile.value = null
  notificationsReady.value = false
  profileMenuOpen.value = false
  authMessage.value = ''
  authError.value = ''
  products.value = []
  analyticsProducts.value = []
  productsError.value = ''
  analyticsError.value = ''
  currentPage.value = 'home'
  notificationSelectOpen.value = false
  resetProductSwipe()
  clearUrlState()
}

function toggleProfileMenu() {
  profileMenuOpen.value = !profileMenuOpen.value
}

function handleDocumentPointerDown(event) {
  if (!notificationSelectOpen.value || !notificationSelectRef.value) {
    return
  }

  if (!notificationSelectRef.value.contains(event.target)) {
    notificationSelectOpen.value = false
  }
}

function selectNotificationRule(value) {
  notificationRule.value = value
  notificationSelectOpen.value = false
}

function urlBase64ToUint8Array(base64String) {
  const padding = '='.repeat((4 - (base64String.length % 4)) % 4)
  const normalized = (base64String + padding).replace(/-/g, '+').replace(/_/g, '/')
  const rawData = window.atob(normalized)
  return Uint8Array.from([...rawData].map((char) => char.charCodeAt(0)))
}

async function openPage(pageKey) {
  if (pageKey === 'settings') {
    return
  }
  currentPage.value = pageKey
  profileMenuOpen.value = false
  notificationSelectOpen.value = false
  resetProductSwipe()
  if (pageKey === 'analytics' && canAccessAnalytics.value && !analyticsProducts.value.length) {
    await loadAnalytics()
  }
}

async function requestBrowserNotificationPermission() {
  if (typeof window === 'undefined' || !('Notification' in window)) {
    return 'denied'
  }
  if (window.Notification.permission === 'granted') {
    return 'granted'
  }
  return window.Notification.requestPermission()
}

async function getServiceWorkerRegistration() {
  if (typeof window === 'undefined' || !('serviceWorker' in navigator)) {
    return null
  }

  const existing = await navigator.serviceWorker.getRegistration('/sw.js')
  if (existing) {
    return existing
  }

  return navigator.serviceWorker.register('/sw.js')
}

async function syncPushSubscription(enableBrowserPush) {
  const registration = await getServiceWorkerRegistration()
  if (!registration || !session.value?.token) {
    return
  }

  const existing = await registration.pushManager.getSubscription()
  if (!enableBrowserPush) {
    if (existing) {
      await existing.unsubscribe()
    }
    await fetch(`${apiBaseUrl}/api/v1/notifications/push/subscribe`, {
      method: 'DELETE',
      headers: getAuthHeaders(),
    })
    return
  }

  const keyResponse = await fetch(`${apiBaseUrl}/api/v1/notifications/push/public-key`, {
    headers: getAuthHeaders(),
  })
  const keyPayload = await keyResponse.json()
  if (!keyResponse.ok || !keyPayload.public_key) {
    return
  }

  const permission = await requestBrowserNotificationPermission()
  if (permission !== 'granted') {
    return
  }

  const subscription =
    existing ||
    (await registration.pushManager.subscribe({
      userVisibleOnly: true,
      applicationServerKey: urlBase64ToUint8Array(keyPayload.public_key),
    }))

  await fetch(`${apiBaseUrl}/api/v1/notifications/push/subscribe`, {
    method: 'POST',
    headers: getAuthHeaders(),
    body: JSON.stringify(subscription.toJSON()),
  })
}

function playBrowserNotificationSound() {
  if (!browserSoundEnabled.value || typeof window === 'undefined' || !window.AudioContext) {
    return
  }
  const context = new window.AudioContext()
  const oscillator = context.createOscillator()
  const gain = context.createGain()
  oscillator.type = 'sine'
  oscillator.frequency.value = 880
  gain.gain.value = 0.02
  oscillator.connect(gain)
  gain.connect(context.destination)
  oscillator.start()
  oscillator.stop(context.currentTime + 0.12)
}

async function markNotificationRead(eventId) {
  await fetch(`${apiBaseUrl}/api/v1/notifications/events/${eventId}/read`, {
    method: 'POST',
    headers: getAuthHeaders(),
  })
}

async function pollBrowserNotifications() {
  if (
    !session.value?.token ||
    notificationChannel.value !== 'browser' ||
    typeof window === 'undefined' ||
    !('Notification' in window) ||
    window.Notification.permission !== 'granted'
  ) {
    return
  }

  const response = await fetch(`${apiBaseUrl}/api/v1/notifications/events`, {
    headers: getAuthHeaders(),
  })
  const payload = await response.json()
  if (!response.ok || !payload.items?.length) {
    return
  }

  for (const item of payload.items) {
    new window.Notification(item.title, {
      body: item.message,
      icon: '/favicon.svg',
      tag: `ac-price-${item.id}`,
    })
    playBrowserNotificationSound()
    await markNotificationRead(item.id)
  }
}

function stopBrowserNotificationPolling() {
  if (browserNotificationTimer) {
    window.clearInterval(browserNotificationTimer)
    browserNotificationTimer = null
  }
}

function startBrowserNotificationPolling() {
  stopBrowserNotificationPolling()
  pollBrowserNotifications().catch(() => {})
  browserNotificationTimer = window.setInterval(() => {
    pollBrowserNotifications().catch(() => {})
  }, 30000)
}

onMounted(async () => {
  if (!notificationSelectListenerAttached) {
    document.addEventListener('pointerdown', handleDocumentPointerDown)
    notificationSelectListenerAttached = true
  }

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
      await loadNotificationSettings()
      await syncPushSubscription(notificationChannel.value === 'browser')
      await loadProducts()
      startBrowserNotificationPolling()
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
      await loadNotificationSettings()
      await syncPushSubscription(notificationChannel.value === 'browser')
      await loadProducts()
      startBrowserNotificationPolling()
    } catch (error) {
      authError.value = error.message || 'Не удалось загрузить данные аккаунта.'
    }
  }
})

watch(notificationChannel, async (value) => {
  if (value === 'browser') {
    await requestBrowserNotificationPermission()
  }
  await syncPushSubscription(value === 'browser')
  startBrowserNotificationPolling()
})

watch(
  [
    notificationChannel,
    notificationRule,
    notificationAmount,
    browserSoundEnabled,
    quietHoursEnabled,
    outOfStockAlerts,
    digestEnabled,
  ],
  async () => {
    await saveNotificationSettings()
  },
)

onBeforeUnmount(() => {
  if (notificationSelectListenerAttached) {
    document.removeEventListener('pointerdown', handleDocumentPointerDown)
    notificationSelectListenerAttached = false
  }
  stopBrowserNotificationPolling()
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
    <div class="mobile-topbar panel">
      <div class="mobile-brand">
        <div class="brand-mark">
          <TrendingUp aria-hidden="true" />
        </div>
        <div>
          <p class="brand-title">AC Price</p>
          <p class="brand-subtitle">Kaspi price tracker</p>
        </div>
      </div>

      <div class="mobile-topbar__actions">
        <button type="button" class="mobile-profile" @click="toggleProfileMenu">
          <span class="profile-chip__avatar">
            {{ (session?.username || 'U').slice(0, 2).toUpperCase() }}
          </span>
          <span class="mobile-profile__name">{{ session?.username || 'user' }}</span>
          <span class="profile-chip__arrow" :class="{ open: profileMenuOpen }">
            <ChevronDown aria-hidden="true" />
          </span>
        </button>

        <div v-if="profileMenuOpen" class="profile-menu profile-menu--mobile">
          <button type="button" class="profile-menu__item" @click="logout">Выйти</button>
        </div>
      </div>
    </div>

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

        <div v-if="profile?.tariff === 'free'" class="premium-card">
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
              <ChartNoAxesColumn v-else-if="item.icon === 'chart'" aria-hidden="true" />
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
              <p class="products-note">{{ productsUpdateHint }}</p>
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
          </div>

          <div class="table-scroll">
            <div v-if="isProductsLoading" class="empty-state">Загрузка товаров...</div>
            <div v-else-if="productsError" class="empty-state empty-state--error">{{ productsError }}</div>
            <div v-else-if="!products.length" class="empty-state">
              Пока нет товаров. Нажмите «Добавить товар» и вставьте ссылку на Kaspi.
            </div>
            <article
              v-for="item in products"
              :key="item.id"
              class="table-row"
              :class="{ 'table-row--swiped': swipedProductId === item.id }"
              @touchstart.passive="handleProductTouchStart(item.id, $event)"
              @touchmove="handleProductTouchMove(item.id, $event)"
              @touchend="handleProductTouchEnd(item.id)"
              @touchcancel="resetProductSwipe"
            >
              <button
                type="button"
                class="table-row__delete-action"
                :disabled="deletingProductId === item.id"
                @click="deleteProduct(item.id)"
              >
                <Trash2 aria-hidden="true" />
                <span>{{ deletingProductId === item.id ? 'Удаление...' : 'Удалить' }}</span>
              </button>
              <div class="table-row__content" :style="getProductRowStyle(item.id)">
                <div class="product-main">
                <div class="product-thumb">
                  <img
                    v-if="item.image_url"
                    :src="item.image_url"
                    :alt="item.product_name"
                    class="product-thumb__image"
                    loading="lazy"
                  />
                  <template v-else>{{ buildThumb(item.product_name) }}</template>
                </div>
                <div class="product-copy">
                  <strong :title="item.product_name">{{ smartShortenProductName(item.product_name) }}</strong>
                  <a :href="item.url" target="_blank" rel="noreferrer">{{ item.url }}</a>
                </div>
              </div>
              <div>
                <strong v-if="item.is_available" class="mono">{{ formatPrice(item.current_price) }}</strong>
                <strong v-else class="mono price-unavailable">{{ getOutOfStockLabel() }}</strong>
              </div>
              <div>
                <template v-if="item.is_available">
                  <strong class="price-min mono">{{ formatPrice(item.min_price) }}</strong>
                  <p>{{ formatShortDate(item.min_price_recorded_at) }}</p>
                </template>
                <strong v-else class="mono price-unavailable">{{ getOutOfStockLabel() }}</strong>
              </div>
              <div>
                <template v-if="item.is_available">
                  <strong class="price-change mono" :class="{ down: getTrackedChange(item) < 0 }">
                  {{ getTrackedChange(item) < 0 ? '↓' : getTrackedChange(item) > 0 ? '↑' : '•' }}
                  {{ formatPrice(Math.abs(getTrackedChange(item))) }}
                  </strong>
                  <p>
                    {{
                      `(${getTrackedChangePercent(item)}%)`
                    }}
                  </p>
                </template>
                <strong v-else class="mono price-unavailable">{{ getOutOfStockLabel() }}</strong>
              </div>
              <div class="product-row-actions">
                <button
                  type="button"
                  class="product-delete-button"
                  :disabled="deletingProductId === item.id"
                  @click="deleteProduct(item.id)"
                >
                  <Trash2 aria-hidden="true" />
                </button>
              </div>
              </div>
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
                <div class="product-thumb">
                  <img
                    v-if="item.image_url"
                    :src="item.image_url"
                    :alt="item.product_name"
                    class="product-thumb__image"
                    loading="lazy"
                  />
                  <template v-else>{{ buildThumb(item.product_name) }}</template>
                </div>
                <div class="analytics-row__copy">
                  <strong :title="item.product_name">{{ smartShortenProductName(item.product_name, 48) }}</strong>
                  <span>{{ item.historyCount }} точек истории</span>
                </div>
              </div>

              <div class="analytics-row__chart">
                <AnalyticsMiniChart
                  v-if="item.historyCount"
                  :history="item.history"
                  :format-price="formatPrice"
                  :format-short-date="formatShortDate"
                />
              </div>

              <div class="analytics-row__stats">
                <div>
                  <span>При добавлении</span>
                  <strong>{{ item.historyCount ? formatPrice(item.initial_price) : getOutOfStockLabel() }}</strong>
                </div>
                <div>
                  <span>Текущая цена</span>
                  <strong>{{ item.is_available ? formatPrice(item.current_price) : getOutOfStockLabel() }}</strong>
                </div>
                <div>
                  <span>Мин/Макс</span>
                  <strong class="price-min">
                    {{ item.historyCount ? `${formatPrice(item.min_price)} / ${formatPrice(item.max_price)}` : getOutOfStockLabel() }}
                  </strong>
                </div>
                <div>
                  <span>Изменение</span>
                  <strong v-if="item.is_available" :class="['price-change', { down: getTrackedChange(item) < 0 }]">
                    {{ getTrackedChange(item) < 0 ? '↓' : getTrackedChange(item) > 0 ? '↑' : '•' }}
                    {{ formatPrice(Math.abs(getTrackedChange(item))) }}
                  </strong>
                  <strong v-else class="price-unavailable">{{ getOutOfStockLabel() }}</strong>
                </div>
              </div>
            </article>
          </section>
        </template>
      </template>

      <template v-else-if="currentPage === 'notifications'">
        <header class="dashboard-header">
          <div>
            <h1 class="dashboard-title">Уведомления</h1>
            <p class="dashboard-subtitle">Настройте канал доставки, частоту и правила срабатывания уведомлений</p>
          </div>
        </header>

        <section class="notifications-layout">
          <article class="notification-panel panel">
            <div class="notification-panel__head">
              <div>
                <p class="section-kicker">Канал доставки</p>
                <h2>Куда отправлять сигналы</h2>
              </div>
              <div class="notification-status">
                <ShieldCheck aria-hidden="true" />
                <span>Доступно для всех тарифов</span>
              </div>
            </div>

            <div class="channel-switch">
              <button
                type="button"
                class="channel-switch__option"
                :class="{ active: notificationChannel === 'browser' }"
                @click="notificationChannel = 'browser'"
              >
                <Monitor aria-hidden="true" />
                <span>В браузер</span>
              </button>

              <div class="channel-switch__track" :class="{ telegram: notificationChannel === 'telegram' }">
                <span class="channel-switch__thumb"></span>
              </div>

              <button
                type="button"
                class="channel-switch__option"
                :class="{ active: notificationChannel === 'telegram' }"
                @click="notificationChannel = 'telegram'"
              >
                <MessageCircleMore aria-hidden="true" />
                <span>В Telegram</span>
              </button>
            </div>

            <p class="notification-panel__fact">{{ notificationFact }}</p>
          </article>

          <article class="notification-panel panel" :class="{ 'notification-panel--raised': notificationSelectOpen }">
            <div class="notification-panel__head">
              <div>
                <p class="section-kicker">Частота</p>
                <h2>Когда присылать уведомления</h2>
              </div>
            </div>

            <div ref="notificationSelectRef" class="custom-select">
              <button type="button" class="custom-select__trigger" @click="notificationSelectOpen = !notificationSelectOpen">
                <div>
                  <strong>{{ selectedNotificationOption.label }}</strong>
                  <span>{{ selectedNotificationOption.hint }}</span>
                </div>
                <span class="custom-select__arrow" :class="{ open: notificationSelectOpen }">
                  <ChevronDown aria-hidden="true" />
                </span>
              </button>

              <div v-if="notificationSelectOpen" class="custom-select__menu">
                <button
                  v-for="option in notificationOptions"
                  :key="option.value"
                  type="button"
                  class="custom-select__option"
                  :class="{ active: option.value === notificationRule }"
                  @click="selectNotificationRule(option.value)"
                >
                  <div>
                    <strong>{{ option.label }}</strong>
                    <span>{{ option.hint }}</span>
                  </div>
                  <Check v-if="option.value === notificationRule" aria-hidden="true" />
                </button>
              </div>
            </div>

            <div v-if="notificationRule === 'amount_change'" class="range-setting">
              <div class="range-setting__head">
                <span>Порог изменения</span>
                <strong>{{ amountChangeLabel }}</strong>
              </div>
              <input v-model="notificationAmount" class="range-setting__input" type="range" min="500" max="50000" step="500" />
              <div class="range-setting__scale">
                <span>500 ₸</span>
                <span>50 000 ₸</span>
              </div>
            </div>
          </article>

          <article class="notification-panel panel">
            <div class="notification-panel__head">
              <div>
                <p class="section-kicker">Дополнительно</p>
                <h2>Полезные настройки</h2>
              </div>
            </div>

            <div class="setting-list">
              <label class="setting-row">
                <div class="setting-row__copy">
                  <strong>Звук в браузере</strong>
                  <span>Добавляет короткий сигнал к браузерному push-уведомлению</span>
                </div>
                <input v-model="browserSoundEnabled" class="switch-input" type="checkbox" />
              </label>

              <label class="setting-row">
                <div class="setting-row__copy">
                  <strong>Не беспокоить ночью</strong>
                  <span>Откладывать несрочные уведомления с 23:00 до 08:00</span>
                </div>
                <input v-model="quietHoursEnabled" class="switch-input" type="checkbox" />
              </label>

              <label class="setting-row">
                <div class="setting-row__copy">
                  <strong>Товар пропал из наличия</strong>
                  <span>Присылать отдельный сигнал, если карточка стала недоступна</span>
                </div>
                <input v-model="outOfStockAlerts" class="switch-input" type="checkbox" />
              </label>

              <label class="setting-row">
                <div class="setting-row__copy">
                  <strong>Вечерняя сводка</strong>
                  <span>Один компактный отчёт по всем изменениям за день</span>
                </div>
                <input v-model="digestEnabled" class="switch-input" type="checkbox" />
              </label>
            </div>
          </article>

          <article class="notification-panel notification-preview panel">
            <div class="notification-panel__head">
              <div>
                <p class="section-kicker">Превью</p>
                <h2>Как это будет выглядеть</h2>
              </div>
            </div>

            <div class="notification-preview__card">
              <div class="notification-preview__icon" :class="notificationChannel">
                <BellRing aria-hidden="true" />
              </div>
              <div class="notification-preview__copy">
                <strong>Apple MacBook Air 13</strong>
                <p>Цена снизилась до 465 190 ₸. Минимум обновлён, можно проверять карточку.</p>
              </div>
            </div>

            <ul class="notification-preview__meta">
              <li>
                <SlidersHorizontal aria-hidden="true" />
                <span>{{ selectedNotificationOption.label }}</span>
              </li>
              <li>
                <MessageCircleMore v-if="notificationChannel === 'telegram'" aria-hidden="true" />
                <Monitor v-else aria-hidden="true" />
                <span>{{ notificationChannel === 'telegram' ? 'Telegram-канал' : 'Браузерный push' }}</span>
              </li>
              <li v-if="notificationRule === 'amount_change'">
                <CircleDollarSign aria-hidden="true" />
                <span>Порог: {{ amountChangeLabel }}</span>
              </li>
            </ul>
          </article>
        </section>
      </template>

      <template v-else>
        <section class="analytics-lock panel">
          <div class="empty-state">Раздел пока в разработке.</div>
        </section>
      </template>
    </section>

    <nav class="mobile-nav panel">
      <button
        v-for="item in navItems"
        :key="`mobile-${item.key}`"
        type="button"
        class="nav-item"
        :class="{ active: currentPage === item.key, muted: item.disabled }"
        @click="openPage(item.key)"
      >
        <component :is="item.icon" aria-hidden="true" />
        <span>{{ item.label }}</span>
      </button>
    </nav>

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

<script setup lang="ts">
import { computed, onBeforeUnmount, reactive, ref, watch } from 'vue'
import { RouterLink, useRoute, useRouter } from 'vue-router'
import monitoringServerVisual from '@/assets/monitoring-server-visual.svg'
import VoiceChatView from '@/views/VoiceChatView.vue'
import {
  demoMonitoringMenu,
  getDemoDashboardMetrics,
  loginDemoUser,
  registerDemoUser,
  type DemoMonitoringMenuKey,
} from '@/utils/dummyExperiences'

type AuthMode = 'login' | 'register'

const authMode = ref<AuthMode>('login')
const isAuthenticated = ref(false)
const isDarkMode = ref(true)
const isSidebarCollapsed = ref(false)
const route = useRoute()
const router = useRouter()
const requestedMenu = route.query.menu
const initialMenu: DemoMonitoringMenuKey =
  requestedMenu === 'voice-chat' || requestedMenu === 'location' ? requestedMenu : 'home'
const activeMenu = ref<DemoMonitoringMenuKey>(initialMenu)
const message = ref('')
const messageIsError = ref(false)
const form = reactive({
  name: '',
  email: '',
  password: '',
})

const metrics = getDemoDashboardMetrics()
const services = [
  { name: 'API Gateway', status: 'Healthy', latency: '84 ms', uptime: '99.99%' },
  { name: 'Auth Service', status: 'Healthy', latency: '112 ms', uptime: '99.97%' },
  { name: 'Notification Worker', status: 'Degraded', latency: '421 ms', uptime: '98.72%' },
]
const incidents = [
  { id: 'INC-024', title: 'Notification worker latency', severity: 'Medium', age: '18 min ago' },
  { id: 'INC-023', title: 'Scheduled maintenance completed', severity: 'Resolved', age: '2 hr ago' },
  { id: 'INC-022', title: 'API gateway timeout spike', severity: 'Resolved', age: 'Yesterday' },
]
const alerts = [
  { name: 'High latency', target: 'Notification Worker', state: 'Active', threshold: '> 300 ms' },
  { name: 'Error rate', target: 'API Gateway', state: 'Armed', threshold: '> 2%' },
  { name: 'Service down', target: 'All services', state: 'Armed', threshold: '0% uptime' },
]
type MapLocation = {
  latitude: number
  longitude: number
}

const defaultLocation: MapLocation = { latitude: -6.2088, longitude: 106.8456 }
const markedLocation = ref<MapLocation | null>(null)
const locationStatus = ref('')
type JourneyState = 'idle' | 'tracking' | 'stopped'
type JourneyEventType = 'start' | 'mark' | 'stop'
type JourneyEvent = {
  type: JourneyEventType
  location: MapLocation | null
  timestamp: number
}
type JourneyRecord = {
  id: number
  name: string
  startedAt: number
  stoppedAt: number | null
  events: JourneyEvent[]
}

const journeyState = ref<JourneyState>('idle')
const journeyStatus = ref('Ready to start a journey.')
const journeyStops = ref<MapLocation[]>([])
const journeyNameInput = ref('')
const isJourneyNamePromptOpen = ref(false)
const currentJourney = ref<JourneyRecord | null>(null)
const journeyHistory = ref<JourneyRecord[]>([])
let locationWatchId: number | null = null

const geolocationOptions: PositionOptions = {
  enableHighAccuracy: true,
  timeout: 10000,
  maximumAge: 0,
}

function createMapUrl(location: MapLocation, embed: boolean) {
  const latitudeOffset = 0.09
  const longitudeOffset = 0.085
  const minimumLongitude = (location.longitude - longitudeOffset).toFixed(6)
  const minimumLatitude = (location.latitude - latitudeOffset).toFixed(6)
  const maximumLongitude = (location.longitude + longitudeOffset).toFixed(6)
  const maximumLatitude = (location.latitude + latitudeOffset).toFixed(6)
  const marker = `${location.latitude.toFixed(6)},${location.longitude.toFixed(6)}`
  const center = `13/${marker}`

  if (embed) {
    return `https://www.openstreetmap.org/export/embed.html?bbox=${minimumLongitude}%2C${minimumLatitude}%2C${maximumLongitude}%2C${maximumLatitude}&layer=mapnik&marker=${marker}#map=${center}`
  }

  return `https://www.openstreetmap.org/?mlat=${location.latitude.toFixed(6)}&mlon=${location.longitude.toFixed(6)}#map=${center}`
}

function getWeekStart(timestamp: number) {
  const date = new Date(timestamp)
  const day = date.getDay()
  const daysFromMonday = day === 0 ? -6 : 1 - day
  date.setDate(date.getDate() + daysFromMonday)
  date.setHours(0, 0, 0, 0)
  return date
}

const locationMapUrl = computed(() => createMapUrl(markedLocation.value ?? defaultLocation, true))
const fullLocationMapUrl = computed(() => createMapUrl(markedLocation.value ?? defaultLocation, false))
const locationLabel = computed(() => (markedLocation.value ? 'My device location' : 'Jakarta, Indonesia'))
const weeklyJourneys = computed(() => {
  const groups = new Map<string, { key: string; label: string; journeys: JourneyRecord[] }>()

  for (const journey of journeyHistory.value) {
    const weekStart = getWeekStart(journey.startedAt)
    const key = `${weekStart.getFullYear()}-${weekStart.getMonth() + 1}-${weekStart.getDate()}`
    const group = groups.get(key) ?? {
      key,
      label: `Week of ${weekStart.toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' })}`,
      journeys: [],
    }
    group.journeys.push(journey)
    groups.set(key, group)
  }

  return [...groups.values()]
})

const authTitle = computed(() => (authMode.value === 'login' ? 'Masuk ke dashboard' : 'Buat akun demo'))
const menuIconClasses: Record<DemoMonitoringMenuKey, string> = {
  home: 'fa-house',
  overview: 'fa-chart-line',
  services: 'fa-server',
  incidents: 'fa-triangle-exclamation',
  alerts: 'fa-bell',
  'voice-chat': 'fa-microphone-lines',
  location: 'fa-location-dot',
  settings: 'fa-gear',
}

function setMode(mode: AuthMode) {
  authMode.value = mode
  message.value = ''
  messageIsError.value = false
}

function submitAuth() {
  const result =
    authMode.value === 'login'
      ? loginDemoUser(form.email, form.password)
      : registerDemoUser(form.name, form.email, form.password)

  message.value = result.message
  messageIsError.value = !result.ok

  if (result.ok && authMode.value === 'login') {
    isAuthenticated.value = true
  } else if (result.ok) {
    authMode.value = 'login'
  }
}

function logout() {
  if (journeyState.value === 'tracking') stopJourney()
  isAuthenticated.value = false
  activeMenu.value = 'home'
  form.password = ''
  message.value = 'Anda sudah keluar dari dashboard demo.'
  messageIsError.value = false
}

function selectMenu(key: DemoMonitoringMenuKey) {
  activeMenu.value = key
  void router.replace({ query: key === 'voice-chat' || key === 'location' ? { menu: key } : {} })
}

function toggleSidebar() {
  isSidebarCollapsed.value = !isSidebarCollapsed.value
}

function toggleTheme() {
  isDarkMode.value = !isDarkMode.value
}

function updateMarkedLocation(position: GeolocationPosition) {
  markedLocation.value = {
    latitude: position.coords.latitude,
    longitude: position.coords.longitude,
  }
}

function focusMyLocation() {
  if (!navigator.geolocation) {
    locationStatus.value = 'Geolocation tidak tersedia di browser ini.'
    return
  }

  locationStatus.value = 'Meminta izin lokasi perangkat...'
  navigator.geolocation.getCurrentPosition(
    (position) => {
      updateMarkedLocation(position)
      locationStatus.value = 'Map focused on your current location.'
    },
    (error) => {
      locationStatus.value =
        error.code === error.PERMISSION_DENIED
          ? 'Izin lokasi ditolak. Izinkan lokasi untuk menandainya di peta.'
          : 'Lokasi perangkat belum dapat ditemukan.'
    },
    geolocationOptions,
  )
}

function openJourneyStart() {
  if (journeyState.value === 'tracking') return
  journeyNameInput.value = ''
  isJourneyNamePromptOpen.value = true
  journeyStatus.value = 'Enter a journey name before starting.'
}

function cancelJourneyStart() {
  isJourneyNamePromptOpen.value = false
  journeyNameInput.value = ''
  journeyStatus.value = 'Ready to start a journey.'
}

function confirmJourneyStart() {
  const name = journeyNameInput.value.trim()
  if (!name) {
    journeyStatus.value = 'Enter a journey name before starting.'
    return
  }

  isJourneyNamePromptOpen.value = false
  startJourney(name)
}

function startJourney(name: string) {
  if (!navigator.geolocation) {
    journeyStatus.value = 'Geolocation is not available in this browser.'
    return
  }

  if (locationWatchId !== null) return

  journeyStops.value = []
  const startedAt = Date.now()
  currentJourney.value = {
    id: startedAt,
    name,
    startedAt,
    stoppedAt: null,
    events: [
      {
        type: 'start',
        location: markedLocation.value ? { ...markedLocation.value } : null,
        timestamp: startedAt,
      },
    ],
  }
  journeyState.value = 'tracking'
  journeyStatus.value = `Journey "${name}" started. Tracking your location...`
  locationWatchId = navigator.geolocation.watchPosition(
    (position) => {
      updateMarkedLocation(position)
      locationStatus.value = 'Live location updated.'
    },
    (error) => {
      journeyStatus.value =
        error.code === error.PERMISSION_DENIED
          ? 'Location permission denied. Allow location to track the journey.'
          : 'Unable to update the live location.'
    },
    geolocationOptions,
  )
}

function markJourneyStop() {
  if (journeyState.value !== 'tracking') {
    journeyStatus.value = 'Start a journey before marking a stop.'
    return
  }

  if (!markedLocation.value) {
    journeyStatus.value = 'Waiting for a location before marking a stop.'
    return
  }

  journeyStops.value.push({ ...markedLocation.value })
  currentJourney.value?.events.push({
    type: 'mark',
    location: { ...markedLocation.value },
    timestamp: Date.now(),
  })
  journeyStatus.value = `Stop ${journeyStops.value.length} marked at your current location.`
}

function stopJourney() {
  if (locationWatchId !== null) {
    navigator.geolocation.clearWatch(locationWatchId)
    locationWatchId = null
  }

  const stoppedAt = Date.now()
  if (currentJourney.value) {
    currentJourney.value.stoppedAt = stoppedAt
    currentJourney.value.events.push({
      type: 'stop',
      location: markedLocation.value ? { ...markedLocation.value } : null,
      timestamp: stoppedAt,
    })
    journeyHistory.value.unshift({
      ...currentJourney.value,
      events: [...currentJourney.value.events],
    })
    currentJourney.value = null
  }

  journeyState.value = 'stopped'
  journeyStatus.value = `Journey stopped. ${journeyStops.value.length} stop(s) marked.`
}

function clearLocation() {
  markedLocation.value = null
  locationStatus.value = 'Penanda lokasi dihapus. Peta kembali ke Jakarta.'
}

watch(
  [activeMenu, isAuthenticated],
  ([menu, authenticated]) => {
    if (menu === 'location' && authenticated && !markedLocation.value) focusMyLocation()
  },
  { immediate: true },
)

onBeforeUnmount(() => {
  if (locationWatchId !== null) navigator.geolocation.clearWatch(locationWatchId)
})
</script>

<template>
  <main
    class="demo-page demo-monitoring-page"
    :class="{ 'is-light': !isDarkMode, 'is-login-screen': !isAuthenticated }"
  >
    <section v-if="!isAuthenticated" class="demo-auth-screen">
      <div
        class="demo-auth-visual"
        :style="{
          backgroundImage:
            'linear-gradient(145deg, rgba(7, 24, 43, 0.08), rgba(4, 16, 31, 0.58)), url(' +
            monitoringServerVisual +
            ')',
        }"
        aria-hidden="true"
      >
        <div class="demo-auth-visual-overlay">
          <span class="demo-visual-kicker">WEB MONITORING</span>
          <span class="demo-visual-status"><i></i> SYSTEM ONLINE</span>
        </div>
        <div class="demo-auth-visual-tech" aria-hidden="true">
          <span title="GitLab"><i class="fa-brands fa-gitlab"></i></span>
          <span title="Remote server"><i class="fa-solid fa-server"></i></span>
          <span title="PCB"><i class="fa-solid fa-microchip"></i></span>
        </div>
      </div>

      <section class="demo-auth-layout" aria-label="Login Web Monitoring">
        <form
          class="demo-auth-form"
          :class="{ 'is-register': authMode === 'register' }"
          novalidate
          @submit.prevent="submitAuth"
        >
          <header class="demo-auth-card-header">
            <RouterLink class="demo-brand" to="/">
              <span class="demo-brand-mark">PX</span>
              <span>
                <strong>Project Example</strong>
                <small>Web Monitoring</small>
              </span>
            </RouterLink>
            <div class="demo-auth-card-actions">
              <button
                class="demo-theme-toggle"
                type="button"
                :aria-label="isDarkMode ? 'Gunakan light mode' : 'Gunakan dark mode'"
                :title="isDarkMode ? 'Gunakan light mode' : 'Gunakan dark mode'"
                @click="toggleTheme"
              >
                <i class="fa-solid" :class="isDarkMode ? 'fa-sun' : 'fa-moon'" aria-hidden="true"></i>
              </button>
              <RouterLink
                class="demo-back-link demo-icon-link"
                to="/"
                aria-label="Kembali ke portfolio"
                title="Kembali ke portfolio"
              >
                <i class="fa-solid fa-house" aria-hidden="true"></i>
              </RouterLink>
            </div>
          </header>

          <div class="demo-auth-card-body">
        <div class="demo-auth-tabs" role="tablist" aria-label="Autentikasi demo">
          <button
            class="demo-tab"
            :class="{ active: authMode === 'login' }"
            type="button"
            role="tab"
            :aria-selected="authMode === 'login'"
            @click="setMode('login')"
          >
            Login
          </button>
          <button
            class="demo-tab"
            :class="{ active: authMode === 'register' }"
            type="button"
            role="tab"
            :aria-selected="authMode === 'register'"
            @click="setMode('register')"
          >
            Register
          </button>
        </div>
        <p class="demo-eyebrow">PROJECT EXAMPLE</p>
        <h2>{{ authTitle }}</h2>
        <p class="demo-card-copy">Gunakan data contoh apa pun. Tidak ada kredensial yang dikirim.</p>

        <label v-if="authMode === 'register'">
          <span>Nama</span>
          <input v-model="form.name" type="text" placeholder="Demo User" autocomplete="name" />
        </label>
        <label>
          <span>Email</span>
          <input v-model="form.email" type="email" placeholder="demo@example.com" autocomplete="email" />
        </label>
        <label>
          <span>Password</span>
          <input
            v-model="form.password"
            type="password"
            placeholder="demo123"
            :autocomplete="authMode === 'login' ? 'current-password' : 'new-password'"
          />
        </label>
        <button class="demo-primary-button" type="submit">
          {{ authMode === 'login' ? 'Login' : 'Buat akun demo' }}
          <span aria-hidden="true">-&gt;</span>
        </button>
        <p v-if="message" class="demo-form-message" :class="{ error: messageIsError }" role="status">
          {{ message }}
        </p>
          </div>
        </form>
      </section>
    </section>

    <section
      v-else
      class="demo-dashboard-shell"
      :class="{ 'sidebar-collapsed': isSidebarCollapsed }"
    >
      <header v-if="isAuthenticated" class="demo-header">
        <RouterLink class="demo-brand" to="/">
          <span class="demo-brand-mark">PX</span>
          <span>
            <strong>Project Example</strong>
            <small>Web Monitoring</small>
          </span>
        </RouterLink>
        <div class="demo-header-actions">
          <button
            class="demo-theme-toggle"
            type="button"
            :aria-label="isDarkMode ? 'Gunakan light mode' : 'Gunakan dark mode'"
            :title="isDarkMode ? 'Gunakan light mode' : 'Gunakan dark mode'"
            @click="toggleTheme"
          >
            <i class="fa-solid" :class="isDarkMode ? 'fa-sun' : 'fa-moon'" aria-hidden="true"></i>
          </button>
          <span class="demo-badge">DUMMY / LOCAL ONLY</span>
          <button class="demo-ghost-button" type="button" @click="logout">Keluar</button>
        </div>
      </header>

      <aside class="demo-sidebar" aria-label="Sidebar dashboard monitoring">
        <div class="demo-sidebar-heading">
          <div>
            <p class="demo-eyebrow">WORKSPACE</p>
            <strong>Monitoring</strong>
          </div>
          <button
            class="demo-sidebar-toggle"
            type="button"
            :aria-label="isSidebarCollapsed ? 'Expand sidebar' : 'Collapse sidebar'"
            :title="isSidebarCollapsed ? 'Expand sidebar' : 'Collapse sidebar'"
            @click="toggleSidebar"
          >
            <i
              class="fa-solid"
              :class="isSidebarCollapsed ? 'fa-chevron-right' : 'fa-chevron-left'"
              aria-hidden="true"
            ></i>
          </button>
        </div>

        <nav class="demo-sidebar-nav" aria-label="Menu utama dashboard">
          <p class="demo-sidebar-label">MAIN MENU</p>
          <button
            v-for="item in demoMonitoringMenu"
            :key="item.key"
            class="demo-sidebar-link"
            :class="{ active: activeMenu === item.key }"
            type="button"
            :aria-current="activeMenu === item.key ? 'page' : undefined"
            :aria-label="`${item.label}: ${item.description}`"
            :title="item.description"
            @click="selectMenu(item.key)"
          >
            <span class="demo-sidebar-icon" aria-hidden="true">
              <i :class="['fa-solid', menuIconClasses[item.key]]"></i>
            </span>
            <span class="demo-sidebar-link-label">{{ item.label }}</span>
          </button>
        </nav>

        <div class="demo-sidebar-footer">
          <div class="demo-user-chip">
            <span class="demo-user-avatar">DU</span>
            <span class="demo-user-copy">
              <strong>Demo User</strong>
              <small>Local account</small>
            </span>
          </div>
          <button class="demo-sidebar-logout" type="button" @click="logout">
            <i class="fa-solid fa-right-from-bracket" aria-hidden="true"></i>
            <span class="demo-sidebar-link-label">Logout</span>
          </button>
        </div>
      </aside>

      <div class="demo-dashboard">
        <div class="demo-dashboard-content">
          <section v-if="activeMenu === 'home'" class="demo-dashboard-view">
            <div class="demo-view-heading">
              <div>
                <p class="demo-eyebrow">HOME</p>
                <h2>System status</h2>
              </div>
              <span class="demo-live-indicator"><i></i> SIMULATION LIVE</span>
            </div>

            <div class="demo-metric-grid">
              <article v-for="metric in metrics" :key="metric.label" class="demo-metric-card">
                <span>{{ metric.label }}</span>
                <strong>{{ metric.value }}</strong>
                <small :class="metric.tone">{{ metric.trend }}</small>
              </article>
            </div>

            <div class="demo-dashboard-grid">
              <section class="demo-panel demo-service-panel">
                <div class="demo-panel-heading">
                  <div>
                    <p class="demo-eyebrow">SERVICE HEALTH</p>
                    <h2>Service health</h2>
                  </div>
                  <span class="demo-panel-count">03 SERVICES</span>
                </div>
                <div class="demo-service-list">
                  <article v-for="service in services" :key="service.name" class="demo-service-row">
                    <span class="demo-service-dot" :class="{ degraded: service.status === 'Degraded' }"></span>
                    <div>
                      <strong>{{ service.name }}</strong>
                      <small>{{ service.status }} / {{ service.latency }}</small>
                    </div>
                    <b>{{ service.uptime }}</b>
                  </article>
                </div>
              </section>

              <section class="demo-panel demo-activity-panel">
                <div class="demo-panel-heading">
                  <div>
                    <p class="demo-eyebrow">RECENT ACTIVITY</p>
                    <h2>Recent activity</h2>
                  </div>
                  <span class="demo-panel-count">LOCAL DATA</span>
                </div>
                <ul class="demo-event-list">
                  <li><span>10:42</span><strong>Health check completed</strong><em>OK</em></li>
                  <li><span>10:37</span><strong>Worker latency increased</strong><em class="warning">WARN</em></li>
                  <li><span>10:21</span><strong>Dashboard session started</strong><em>INFO</em></li>
                </ul>
              </section>
            </div>
          </section>

          <section v-else-if="activeMenu === 'overview'" class="demo-dashboard-view">
            <div class="demo-view-heading">
              <div>
                <p class="demo-eyebrow">OVERVIEW</p>
                <h2>Performance overview</h2>
              </div>
              <span class="demo-panel-count">LAST 24 HOURS</span>
            </div>
            <div class="demo-metric-grid">
              <article v-for="metric in metrics" :key="`overview-${metric.label}`" class="demo-metric-card">
                <span>{{ metric.label }}</span>
                <strong>{{ metric.value }}</strong>
                <small :class="metric.tone">{{ metric.trend }}</small>
              </article>
            </div>
            <div class="demo-dashboard-grid">
              <section class="demo-panel">
                <div class="demo-panel-heading">
                  <div>
                    <p class="demo-eyebrow">REQUEST VOLUME</p>
                    <h2>Traffic trend</h2>
                  </div>
                  <span class="demo-panel-count">1.284 / MIN</span>
                </div>
                <div class="demo-bar-chart" aria-label="Dummy traffic trend chart">
                  <span style="--bar-size: 48%"></span>
                  <span style="--bar-size: 66%"></span>
                  <span style="--bar-size: 54%"></span>
                  <span style="--bar-size: 78%"></span>
                  <span style="--bar-size: 70%"></span>
                  <span style="--bar-size: 92%"></span>
                  <span style="--bar-size: 84%"></span>
                </div>
              </section>
              <section class="demo-panel">
                <div class="demo-panel-heading">
                  <div>
                    <p class="demo-eyebrow">RESPONSE TIME</p>
                    <h2>Latency trend</h2>
                  </div>
                  <span class="demo-panel-count">184 MS P95</span>
                </div>
                <div class="demo-trend-summary">
                  <strong>-12 ms</strong>
                  <span>Average latency improved compared with the previous period.</span>
                </div>
              </section>
            </div>
          </section>

          <section v-else-if="activeMenu === 'services'" class="demo-dashboard-view">
            <div class="demo-view-heading">
              <div>
                <p class="demo-eyebrow">SERVICES</p>
                <h2>Service catalog</h2>
              </div>
              <span class="demo-panel-count">03 REGISTERED</span>
            </div>
            <section class="demo-panel">
              <div class="demo-service-list">
                <article v-for="service in services" :key="`catalog-${service.name}`" class="demo-service-row">
                  <span class="demo-service-dot" :class="{ degraded: service.status === 'Degraded' }"></span>
                  <div>
                    <strong>{{ service.name }}</strong>
                    <small>{{ service.status }} / {{ service.latency }} / local dummy</small>
                  </div>
                  <b>{{ service.uptime }}</b>
                </article>
              </div>
            </section>
          </section>

          <section v-else-if="activeMenu === 'incidents'" class="demo-dashboard-view">
            <div class="demo-view-heading">
              <div>
                <p class="demo-eyebrow">INCIDENTS</p>
                <h2>Incident center</h2>
              </div>
              <span class="demo-panel-count">02 OPEN / 01 CLOSED</span>
            </div>
            <section class="demo-panel demo-record-panel">
              <article v-for="incident in incidents" :key="incident.id" class="demo-record-row">
                <span class="demo-record-id">{{ incident.id }}</span>
                <div>
                  <strong>{{ incident.title }}</strong>
                  <small>{{ incident.age }}</small>
                </div>
                <b :class="{ resolved: incident.severity === 'Resolved' }">{{ incident.severity }}</b>
              </article>
            </section>
          </section>

          <section v-else-if="activeMenu === 'alerts'" class="demo-dashboard-view">
            <div class="demo-view-heading">
              <div>
                <p class="demo-eyebrow">ALERTS</p>
                <h2>Alert rules</h2>
              </div>
              <span class="demo-panel-count">03 RULES</span>
            </div>
            <section class="demo-panel demo-record-panel">
              <article v-for="alert in alerts" :key="alert.name" class="demo-record-row">
                <span class="demo-alert-icon"><i class="fa-solid fa-bell" aria-hidden="true"></i></span>
                <div>
                  <strong>{{ alert.name }}</strong>
                  <small>{{ alert.target }} / {{ alert.threshold }}</small>
                </div>
                <b :class="{ active: alert.state === 'Active' }">{{ alert.state }}</b>
              </article>
            </section>
          </section>

          <section v-else-if="activeMenu === 'voice-chat'" class="demo-dashboard-view demo-voice-chat-view">
            <VoiceChatView :embedded="true" />
          </section>

          <section v-else-if="activeMenu === 'location'" class="demo-dashboard-view demo-location-view">
            <div class="demo-view-heading">
              <div>
                <p class="demo-eyebrow">LOCATION</p>
                <h2>Monitoring map</h2>
              </div>
              <span class="demo-panel-count">OpenStreetMap / CITY VIEW</span>
            </div>
            <section class="demo-panel demo-location-panel">
              <div class="demo-location-map-frame">
                <iframe
                  class="demo-location-map"
                  title="Peta lokasi monitoring"
                  :src="locationMapUrl"
                  loading="lazy"
                ></iframe>
                <div class="demo-location-map-tools">
                  <button class="demo-location-focus-button" type="button" aria-label="Focus on my location" @click="focusMyLocation">
                    <i class="fa-solid fa-location-crosshairs" aria-hidden="true"></i>
                    <span>Focus</span>
                  </button>
                  <a class="demo-location-full-map-link" :href="fullLocationMapUrl" target="_blank" rel="noreferrer">
                    <i class="fa-solid fa-up-right-from-square" aria-hidden="true"></i>
                    <span>Open full map</span>
                  </a>
                </div>
              </div>
              <div class="demo-location-meta">
                <div>
                  <span class="demo-eyebrow">ACTIVE LOCATION</span>
                  <strong>{{ locationLabel }}</strong>
                </div>
              </div>
              <p v-if="locationStatus" class="demo-location-status" role="status">{{ locationStatus }}</p>
            </section>
            <section class="demo-panel demo-journey-panel">
              <div class="demo-panel-heading">
                <div>
                  <p class="demo-eyebrow">JELAJAH</p>
                  <h2>Weekly journeys</h2>
                </div>
                <span class="demo-panel-count">SESSION ONLY</span>
              </div>
              <p class="demo-journey-copy">Name a journey before starting. Start, Mark, Stop, and Clear location stay in this session.</p>
              <form v-if="isJourneyNamePromptOpen" class="demo-journey-name-form" @submit.prevent="confirmJourneyStart">
                <label for="journey-name">Journey name</label>
                <div class="demo-journey-name-fields">
                  <input
                    id="journey-name"
                    v-model="journeyNameInput"
                    type="text"
                    placeholder="e.g. Morning route"
                    maxlength="80"
                    required
                    autofocus
                  />
                  <button class="demo-journey-button demo-journey-button-start" type="submit">
                    <i class="fa-solid fa-play" aria-hidden="true"></i>
                    <span>Start journey</span>
                  </button>
                  <button class="demo-journey-button demo-journey-button-cancel" type="button" @click="cancelJourneyStart">
                    Cancel
                  </button>
                </div>
              </form>
              <div class="demo-journey-controls" aria-label="Journey controls">
                <div class="demo-journey-buttons">
                  <button
                    class="demo-journey-button demo-journey-button-start"
                    type="button"
                    :disabled="journeyState === 'tracking'"
                    @click="openJourneyStart"
                  >
                    <i class="fa-solid fa-play" aria-hidden="true"></i>
                    <span>Start</span>
                  </button>
                  <button
                    class="demo-journey-button demo-journey-button-mark"
                    type="button"
                    :disabled="journeyState !== 'tracking' || !markedLocation"
                    @click="markJourneyStop"
                  >
                    <i class="fa-solid fa-location-dot" aria-hidden="true"></i>
                    <span>Mark</span>
                  </button>
                  <button
                    class="demo-journey-button demo-journey-button-stop"
                    type="button"
                    :disabled="journeyState !== 'tracking'"
                    @click="stopJourney"
                  >
                    <i class="fa-solid fa-stop" aria-hidden="true"></i>
                    <span>Stop</span>
                  </button>
                  <button
                    class="demo-journey-button demo-journey-button-clear"
                    type="button"
                    :disabled="!markedLocation"
                    @click="clearLocation"
                  >
                    <i class="fa-solid fa-location-xmark" aria-hidden="true"></i>
                    <span>Clear location</span>
                  </button>
                </div>
                <div class="demo-journey-status" role="status">
                  <i class="demo-journey-status-dot" :class="journeyState" aria-hidden="true"></i>
                  <span>{{ journeyStatus }}</span>
                  <small v-if="journeyStops.length">{{ journeyStops.length }} stop(s)</small>
                </div>
              </div>
              <div v-if="currentJourney" class="demo-current-journey">
                <span class="demo-eyebrow">ACTIVE JOURNEY</span>
                <strong>{{ currentJourney.name }}</strong>
              </div>
              <div class="demo-journey-history">
                <div v-if="weeklyJourneys.length" v-for="week in weeklyJourneys" :key="week.key" class="demo-journey-week">
                  <div class="demo-journey-week-heading">
                    <span>{{ week.label }}</span>
                    <small>{{ week.journeys.length }} journey(s)</small>
                  </div>
                  <article v-for="journey in week.journeys" :key="journey.id" class="demo-journey-history-row">
                    <div>
                      <strong>{{ journey.name }}</strong>
                      <small>{{ journey.events.length }} recorded point(s)</small>
                    </div>
                    <span>Completed</span>
                  </article>
                </div>
                <p v-else class="demo-journey-empty">No completed journeys this session.</p>
              </div>
            </section>
          </section>

          <section v-else-if="activeMenu === 'settings'" class="demo-dashboard-view">
            <div class="demo-view-heading">
              <div>
                <p class="demo-eyebrow">SETTINGS</p>
                <h2>Workspace settings</h2>
              </div>
              <span class="demo-panel-count">LOCAL PREFERENCES</span>
            </div>
            <div class="demo-settings-grid">
              <section class="demo-panel">
                <div class="demo-panel-heading">
                  <div>
                    <p class="demo-eyebrow">DISPLAY</p>
                    <h2>Interface</h2>
                  </div>
                </div>
                <div class="demo-setting-row"><span>Compact sidebar</span><b>Enabled</b></div>
                <div class="demo-setting-row"><span>Live simulation</span><b>Enabled</b></div>
              </section>
              <section class="demo-panel">
                <div class="demo-panel-heading">
                  <div>
                    <p class="demo-eyebrow">DATA SOURCE</p>
                    <h2>Connection</h2>
                  </div>
                </div>
                <div class="demo-setting-row"><span>Database</span><b>Not connected</b></div>
                <div class="demo-setting-row"><span>External API</span><b>Not connected</b></div>
              </section>
            </div>
          </section>
        </div>

        <div class="demo-dashboard-footer">
          <span><i></i> Tidak ada database / API eksternal</span>
          <RouterLink
            class="demo-back-link demo-icon-link"
            to="/"
            aria-label="Kembali ke portfolio"
            title="Kembali ke portfolio"
          >
            <i class="fa-solid fa-house" aria-hidden="true"></i>
          </RouterLink>
        </div>
      </div>
    </section>
  </main>
</template>

<style scoped>
.demo-page {
  --demo-bg: #07182b;
  --demo-surface: #102b45;
  --demo-surface-soft: #0d2238;
  --demo-text: #e8f1fb;
  --demo-heading: #f6fbff;
  --demo-muted: #8fa8bf;
  --demo-copy: #a9bed0;
  --demo-border: rgba(183, 216, 244, 0.18);
  --demo-divider: rgba(183, 216, 244, 0.12);
  --demo-input: #071d33;
  --demo-primary: #a7edff;
  --demo-primary-ink: #06172a;
  min-height: 100vh;
  padding: 28px clamp(20px, 5vw, 76px) 56px;
  color: var(--demo-text);
  background: var(--demo-bg);
  color-scheme: dark;
  transition: color 0.25s ease, background-color 0.25s ease;
}

.demo-page.is-light {
  --demo-bg: #eef5fb;
  --demo-surface: #ffffff;
  --demo-surface-soft: #f5f9fd;
  --demo-text: #18344d;
  --demo-heading: #071f38;
  --demo-muted: #3f5d73;
  --demo-copy: #294b65;
  --demo-border: rgba(39, 79, 112, 0.16);
  --demo-divider: rgba(39, 79, 112, 0.12);
  --demo-input: #f7fbff;
  --demo-primary: #0b6cbf;
  --demo-primary-ink: #ffffff;
  color-scheme: light;
}

.demo-page.is-login-screen {
  height: 100svh;
  min-height: 0;
  padding: 0;
  overflow: hidden;
}

.demo-page:not(.is-login-screen) {
  padding: 0;
}

.demo-auth-layout {
  width: min(1180px, 100%);
  margin-inline: auto;
}

.demo-header,
.demo-dashboard-shell,
.demo-dashboard-footer {
  width: 100%;
  margin-inline: 0;
}

.demo-dashboard-shell {
  display: grid;
  grid-template-columns: 224px minmax(0, 1fr);
  grid-template-rows: auto minmax(0, 1fr);
  width: 100%;
  min-height: 100svh;
  align-items: stretch;
  gap: 0;
  padding-top: 0;
}

.demo-dashboard {
  grid-column: 2;
  grid-row: 2;
  width: auto;
  min-width: 0;
  margin: 0;
  padding-left: 28px;
}

.demo-header {
  display: flex;
  grid-column: 2;
  grid-row: 1;
  align-items: center;
  justify-content: space-between;
  gap: 24px;
  padding: 28px clamp(20px, 5vw, 76px);
  border-bottom: 1px solid rgba(183, 216, 244, 0.15);
}

.demo-brand,
.demo-back-link {
  color: inherit;
  text-decoration: none;
}

.demo-brand {
  display: inline-flex;
  align-items: center;
  gap: 12px;
}

.demo-brand-mark {
  display: grid;
  width: 42px;
  height: 42px;
  color: #06172a;
  background: #a7edff;
  border-radius: 12px;
  place-items: center;
  font-weight: 900;
}

.demo-brand strong,
.demo-brand small {
  display: block;
}

.demo-brand strong { font-size: 0.86rem; }
.demo-brand small { color: var(--demo-muted); font-size: 0.68rem; }

.demo-header-actions,
.demo-live-indicator,
.demo-dashboard-footer {
  display: flex;
  align-items: center;
  gap: 14px;
}

.demo-badge,
.demo-panel-count {
  color: #8fa8bf;
  font-family: 'IBM Plex Mono', monospace;
  font-size: 0.62rem;
  letter-spacing: 0.08em;
}

.demo-back-link { color: var(--demo-primary); font-size: 0.76rem; font-weight: 800; }
.demo-back-link:hover { color: var(--demo-heading); }
.demo-icon-link { display: inline-grid; width: 36px; height: 36px; place-items: center; border: 1px solid var(--demo-border); border-radius: 9px; }

.demo-ghost-button,
.demo-primary-button,
.demo-tab {
  cursor: pointer;
  border: 0;
  font: inherit;
}

.demo-ghost-button {
  padding: 9px 13px;
  color: var(--demo-text);
  background: rgba(255, 255, 255, 0.08);
  border: 1px solid var(--demo-border);
  border-radius: 8px;
  font-size: 0.72rem;
}

.demo-theme-toggle {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 36px;
  height: 36px;
  gap: 7px;
  padding: 0;
  color: var(--demo-text);
  background: rgba(255, 255, 255, 0.08);
  border: 1px solid var(--demo-border);
  border-radius: 8px;
  font-size: 0.7rem;
  font-weight: 800;
}

.demo-theme-toggle i { font-size: 0.86rem; }

.demo-theme-toggle:hover,
.demo-ghost-button:hover {
  border-color: var(--demo-primary);
  background: rgba(167, 237, 255, 0.12);
}

.demo-auth-layout {
  display: flex;
  align-items: center;
  justify-content: center;
  width: auto;
  min-width: 0;
  height: 100%;
  min-height: 0;
  margin: 0;
  padding: clamp(28px, 6vw, 76px);
  overflow: hidden;
  background: var(--demo-surface-soft);
}

.demo-auth-screen {
  display: grid;
  height: 100%;
  min-height: 0;
  overflow: hidden;
  grid-template-columns: minmax(0, 1.15fr) minmax(0, 0.85fr);
}

.demo-auth-visual {
  position: relative;
  height: 100%;
  min-height: 0;
  overflow: hidden;
  background-color: #07182b;
  background-image: linear-gradient(145deg, rgba(7, 24, 43, 0.08), rgba(4, 16, 31, 0.58));
  background-position: center;
  background-size: cover;
}

.demo-auth-visual::after {
  position: absolute;
  inset: 0;
  content: '';
  background: linear-gradient(180deg, transparent 58%, rgba(4, 16, 31, 0.58));
}

.demo-auth-visual-overlay {
  position: absolute;
  right: clamp(24px, 5vw, 78px);
  bottom: clamp(24px, 5vw, 70px);
  left: clamp(24px, 5vw, 78px);
  z-index: 1;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 18px;
  color: #dff8ff;
  font-family: 'IBM Plex Mono', monospace;
  font-size: 0.62rem;
  font-weight: 700;
  letter-spacing: 0.1em;
}

.demo-auth-visual-tech {
  position: absolute;
  top: clamp(28px, 6vw, 78px);
  left: clamp(24px, 5vw, 78px);
  z-index: 1;
  display: flex;
  gap: 10px;
}

.demo-auth-visual-tech span {
  display: grid;
  width: 44px;
  height: 44px;
  color: #dff8ff;
  background: rgba(5, 22, 38, 0.58);
  border: 1px solid rgba(151, 226, 244, 0.35);
  border-radius: 12px;
  place-items: center;
  box-shadow: 0 12px 24px rgba(2, 11, 20, 0.18);
}

.demo-auth-visual-tech span:first-child i { color: #fc6d26; }
.demo-auth-visual-tech span:nth-child(2) i { color: #79e8a9; }
.demo-auth-visual-tech span:nth-child(3) i { color: #72d9f6; }

.demo-visual-status {
  display: inline-flex;
  align-items: center;
  gap: 9px;
  color: #79e8a9;
}

.demo-visual-status i {
  width: 7px;
  height: 7px;
  background: #79e8a9;
  border-radius: 50%;
  box-shadow: 0 0 0 5px rgba(121, 232, 169, 0.12);
}

.demo-eyebrow { margin-bottom: 12px; color: #72d9f6; font-family: 'IBM Plex Mono', monospace; font-size: 0.64rem; font-weight: 700; letter-spacing: 0.12em; }
.demo-intro h1,
.demo-dashboard-heading h1 { max-width: 700px; margin-bottom: 22px; color: var(--demo-heading); font-size: clamp(2.8rem, 6vw, 5.8rem); line-height: 0.95; letter-spacing: -0.07em; }
.demo-intro > p:not(.demo-eyebrow) { max-width: 100%; color: var(--demo-copy); font-size: 1rem; line-height: 1.8; white-space: nowrap; }

.demo-feature-list { display: grid; gap: 12px; margin-top: 32px; color: var(--demo-text); font-size: 0.8rem; }
.demo-feature-list span { display: inline-flex; align-items: center; gap: 10px; }
.demo-feature-list i,
.demo-dashboard-footer i { width: 7px; height: 7px; background: #79e8a9; border-radius: 50%; box-shadow: 0 0 0 5px rgba(121, 232, 169, 0.12); }

.demo-panel,
.demo-metric-card { background: var(--demo-surface); border: 1px solid var(--demo-border); border-radius: 12px; box-shadow: 0 25px 80px rgba(0, 0, 0, 0.22); }
.demo-auth-form { display: grid; box-sizing: border-box; width: min(100%, 520px); min-width: 0; min-height: 100%; gap: 0; padding: clamp(24px, 4vw, 48px); background: transparent; border: 0; border-radius: 0; box-shadow: none; }
.demo-auth-card-header { display: flex; min-width: 0; flex-wrap: wrap; align-items: center; justify-content: space-between; gap: 18px; padding-bottom: 24px; border-bottom: 1px solid var(--demo-divider); }
.demo-auth-card-actions { display: flex; min-width: 0; flex-wrap: wrap; align-items: center; justify-content: flex-end; gap: 12px; }
.demo-auth-card-actions .demo-theme-toggle { padding-inline: 9px; }
.demo-auth-card-body { display: grid; width: 100%; max-width: 520px; min-width: 0; min-height: 0; margin-inline: auto; gap: 16px; padding-top: 26px; }
.demo-auth-tabs { display: grid; grid-template-columns: 1fr 1fr; gap: 5px; padding: 5px; background: rgba(0, 0, 0, 0.2); border-radius: 10px; }
.demo-tab { padding: 10px; color: var(--demo-muted); background: transparent; border-radius: 7px; font-size: 0.75rem; font-weight: 800; }
.demo-tab.active { color: var(--demo-primary-ink); background: var(--demo-primary); }
.demo-auth-form h2 { color: var(--demo-heading); font-size: 1.8rem; letter-spacing: -0.04em; }
.demo-card-copy { color: var(--demo-muted); font-size: 0.75rem; }
.demo-auth-form label { display: grid; gap: 7px; }
.demo-auth-form label span { color: var(--demo-copy); font-size: 0.68rem; font-weight: 800; }
.demo-auth-form input { width: 100%; padding: 12px 13px; color: var(--demo-heading); background: var(--demo-input); border: 1px solid var(--demo-border); border-radius: 9px; outline: none; }
.demo-auth-form input:focus { border-color: #72d9f6; box-shadow: 0 0 0 3px rgba(114, 217, 246, 0.14); }
.demo-primary-button { display: flex; align-items: center; justify-content: space-between; gap: 12px; padding: 14px 16px; color: var(--demo-primary-ink); background: var(--demo-primary); border-radius: 9px; font-size: 0.75rem; font-weight: 900; }
.demo-primary-button:hover { filter: brightness(1.06); }
.demo-form-message { margin: 0; color: #79e8a9; font-size: 0.72rem; }
.demo-form-message.error { color: #ff9d9d; }

.demo-sidebar {
  position: sticky;
  top: 0;
  display: flex;
  grid-column: 1;
  grid-row: 1 / span 2;
  height: auto;
  min-width: 0;
  min-height: 100%;
  flex-direction: column;
  padding: 18px 14px;
  background: var(--demo-surface);
  border: 1px solid var(--demo-border);
  border-top: 0;
  border-bottom: 0;
  border-left: 0;
  border-radius: 0;
  box-shadow: none;
  transition: width 0.25s ease, padding 0.25s ease;
}

.demo-sidebar-heading {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  min-height: 52px;
  padding: 0 4px 18px;
  border-bottom: 1px solid var(--demo-divider);
}

.demo-sidebar-heading .demo-eyebrow {
  margin-bottom: 3px;
  font-size: 0.56rem;
}

.demo-sidebar-heading strong {
  color: var(--demo-heading);
  font-size: 0.9rem;
}

.demo-sidebar-toggle,
.demo-sidebar-logout {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  color: var(--demo-muted);
  background: transparent;
  border: 1px solid transparent;
  border-radius: 8px;
  cursor: pointer;
  font: inherit;
}

.demo-sidebar-toggle {
  width: 32px;
  height: 32px;
  flex: 0 0 auto;
  font-size: 1rem;
}

.demo-sidebar-toggle:hover,
.demo-sidebar-logout:hover {
  color: var(--demo-primary);
  background: rgba(167, 237, 255, 0.1);
  border-color: var(--demo-border);
}

.demo-sidebar-nav {
  display: grid;
  gap: 5px;
  margin-top: 24px;
}

.demo-sidebar-label {
  margin: 0 9px 7px;
  color: var(--demo-muted);
  font-family: 'IBM Plex Mono', monospace;
  font-size: 0.55rem;
  font-weight: 700;
  letter-spacing: 0.1em;
}

.demo-sidebar-link {
  display: flex;
  width: 100%;
  align-items: center;
  gap: 11px;
  padding: 11px 10px;
  color: var(--demo-muted);
  background: transparent;
  border: 1px solid transparent;
  border-radius: 10px;
  cursor: pointer;
  font: inherit;
  font-size: 0.75rem;
  font-weight: 800;
  text-align: left;
  transition: color 0.2s ease, background-color 0.2s ease, border-color 0.2s ease;
}

.demo-sidebar-link:hover {
  color: var(--demo-heading);
  background: rgba(167, 237, 255, 0.08);
}

.demo-sidebar-link.active {
  color: var(--demo-primary-ink);
  background: var(--demo-primary);
  border-color: var(--demo-primary);
  box-shadow: 0 8px 18px rgba(114, 217, 246, 0.16);
}

.demo-sidebar-icon {
  display: grid;
  width: 25px;
  height: 25px;
  flex: 0 0 auto;
  place-items: center;
  color: currentColor;
  border: 1px solid currentColor;
  border-radius: 7px;
  font-size: 0.72rem;
  line-height: 1;
}

.demo-sidebar-footer {
  display: grid;
  gap: 14px;
  margin-top: auto;
  padding-top: 18px;
  border-top: 1px solid var(--demo-divider);
}

.demo-user-chip {
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 0;
  padding: 5px;
}

.demo-user-avatar {
  display: grid;
  width: 30px;
  height: 30px;
  flex: 0 0 auto;
  color: var(--demo-primary-ink);
  background: var(--demo-primary);
  border-radius: 9px;
  place-items: center;
  font-size: 0.58rem;
  font-weight: 900;
}

.demo-user-copy {
  display: grid;
  min-width: 0;
  gap: 2px;
}

.demo-user-copy strong {
  overflow: hidden;
  color: var(--demo-heading);
  font-size: 0.7rem;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.demo-user-copy small {
  color: var(--demo-muted);
  font-size: 0.6rem;
}

.demo-sidebar-logout {
  justify-content: flex-start;
  gap: 11px;
  padding: 9px 10px;
  font-size: 0.72rem;
  font-weight: 800;
}

.sidebar-collapsed {
  grid-template-columns: 72px minmax(0, 1fr);
}

.sidebar-collapsed .demo-sidebar {
  align-items: center;
  padding-inline: 9px;
}

.sidebar-collapsed .demo-sidebar-heading {
  justify-content: center;
  padding-inline: 0;
}

.sidebar-collapsed .demo-sidebar-heading > div,
.sidebar-collapsed .demo-sidebar-label,
.sidebar-collapsed .demo-sidebar-link-label,
.sidebar-collapsed .demo-user-copy {
  display: none;
}

.sidebar-collapsed .demo-sidebar-nav,
.sidebar-collapsed .demo-sidebar-footer {
  width: 100%;
}

.sidebar-collapsed .demo-sidebar-link,
.sidebar-collapsed .demo-sidebar-logout {
  justify-content: center;
  padding-inline: 5px;
}

.sidebar-collapsed .demo-user-chip {
  justify-content: center;
}

.demo-dashboard { padding: 40px clamp(20px, 4vw, 48px) 0 28px; }
.demo-dashboard-content { min-width: 0; }
.demo-dashboard-view { min-width: 0; }
.demo-view-heading { display: flex; align-items: end; justify-content: space-between; gap: 20px; margin-bottom: 24px; }
.demo-view-heading .demo-eyebrow { margin-bottom: 6px; }
.demo-view-heading h2 { color: var(--demo-heading); font-size: clamp(1.25rem, 2.4vw, 1.8rem); letter-spacing: -0.04em; }
.demo-location-panel { overflow: hidden; }
.demo-location-map-frame { position: relative; min-height: 520px; background: #d9e5e8; }
.demo-location-map { display: block; width: 100%; height: 520px; border: 0; }
.demo-location-map-tools { position: absolute; right: 16px; bottom: 16px; z-index: 2; display: flex; align-items: center; gap: 8px; }
.demo-location-focus-button,
.demo-location-full-map-link { display: inline-flex; align-items: center; gap: 8px; padding: 10px 12px; color: #f6fbff; background: rgba(7, 24, 43, 0.9); border: 1px solid rgba(183, 216, 244, 0.35); border-radius: 9px; box-shadow: 0 10px 24px rgba(2, 11, 20, 0.25); font-size: 0.7rem; font-weight: 800; text-decoration: none; }
.demo-location-focus-button:hover,
.demo-location-full-map-link:hover { color: #a7edff; border-color: #a7edff; }
.demo-journey-panel { display: grid; gap: 16px; margin-top: 14px; }
.demo-journey-copy { margin: 0; color: var(--demo-muted); font-size: 0.72rem; }
.demo-journey-name-form { display: grid; gap: 8px; padding: 14px; background: rgba(9, 29, 48, 0.42); border: 1px solid var(--demo-divider); border-radius: 10px; }
.demo-journey-name-form label { color: var(--demo-copy); font-size: 0.68rem; font-weight: 800; }
.demo-journey-name-fields { display: flex; flex-wrap: wrap; gap: 8px; }
.demo-journey-name-fields input { min-width: min(280px, 100%); flex: 1 1 220px; padding: 9px 11px; color: var(--demo-text); background: var(--demo-input); border: 1px solid var(--demo-border); border-radius: 8px; font: inherit; font-size: 0.72rem; }
.demo-journey-name-fields input:focus { outline: 2px solid rgba(167, 237, 255, 0.45); outline-offset: 1px; }
.demo-journey-controls { display: flex; align-items: center; justify-content: space-between; gap: 16px; padding: 0; }
.demo-journey-buttons { display: flex; flex-wrap: wrap; gap: 8px; }
.demo-journey-button { display: inline-flex; align-items: center; gap: 7px; padding: 8px 12px; color: var(--demo-copy); background: transparent; border: 1px solid rgba(183, 216, 244, 0.28); border-radius: 8px; font-size: 0.68rem; font-weight: 800; }
.demo-journey-button:hover:not(:disabled) { color: var(--demo-heading); border-color: #a7edff; background: rgba(167, 237, 255, 0.08); }
.demo-journey-button:disabled { cursor: not-allowed; opacity: 0.4; }
.demo-journey-button-start { color: #9ff0c4; border-color: rgba(121, 232, 169, 0.35); }
.demo-journey-button-mark { color: #a7edff; border-color: rgba(167, 237, 255, 0.35); }
.demo-journey-button-stop { color: #ffc36b; border-color: rgba(255, 195, 107, 0.35); }
.demo-journey-button-clear { color: #ffb5b5; border-color: rgba(255, 181, 181, 0.35); }
.demo-journey-button-cancel { color: var(--demo-muted); }
.demo-journey-status { display: flex; align-items: center; justify-content: flex-end; gap: 8px; min-width: 0; color: var(--demo-muted); font-size: 0.68rem; text-align: right; }
.demo-journey-status > span { max-width: 360px; }
.demo-journey-status small { color: var(--demo-copy); font-family: 'IBM Plex Mono', monospace; font-size: 0.6rem; white-space: nowrap; }
.demo-journey-status-dot { width: 7px; height: 7px; flex: 0 0 auto; background: #8da4b4; border-radius: 50%; }
.demo-journey-status-dot.tracking { background: #79e8a9; box-shadow: 0 0 0 5px rgba(121, 232, 169, 0.12); }
.demo-journey-status-dot.stopped { background: #ffc36b; }
.demo-current-journey { display: flex; align-items: center; justify-content: space-between; gap: 14px; padding: 12px 0; border-top: 1px solid var(--demo-divider); border-bottom: 1px solid var(--demo-divider); }
.demo-current-journey .demo-eyebrow { margin: 0; }
.demo-current-journey strong { color: var(--demo-heading); font-size: 0.78rem; }
.demo-journey-history { display: grid; gap: 14px; }
.demo-journey-week { display: grid; gap: 7px; }
.demo-journey-week-heading { display: flex; align-items: center; justify-content: space-between; gap: 12px; color: var(--demo-copy); font-size: 0.68rem; font-weight: 800; }
.demo-journey-week-heading small { color: var(--demo-muted); font-size: 0.6rem; font-weight: 600; }
.demo-journey-history-row { display: flex; align-items: center; justify-content: space-between; gap: 12px; padding: 10px 12px; background: rgba(167, 237, 255, 0.04); border: 1px solid var(--demo-divider); border-radius: 8px; }
.demo-journey-history-row strong,
.demo-journey-history-row small { display: block; }
.demo-journey-history-row strong { color: var(--demo-text); font-size: 0.72rem; }
.demo-journey-history-row small { margin-top: 3px; color: var(--demo-muted); font-size: 0.6rem; }
.demo-journey-history-row > span { color: #79e8a9; font-family: 'IBM Plex Mono', monospace; font-size: 0.58rem; }
.demo-journey-empty { margin: 0; color: var(--demo-muted); font-size: 0.68rem; }
.demo-location-meta { display: flex; align-items: center; justify-content: space-between; gap: 18px; padding: 18px 20px; border-top: 1px solid var(--demo-border); }
.demo-location-meta > div { display: grid; gap: 5px; }
.demo-location-meta .demo-eyebrow { margin: 0; }
.demo-location-meta strong { color: var(--demo-heading); font-size: 0.95rem; }
.demo-location-status { margin: 0; padding: 0 20px 16px; color: var(--demo-muted); font-size: 0.7rem; }
.demo-live-indicator { color: #79e8a9; font-family: 'IBM Plex Mono', monospace; font-size: 0.62rem; font-weight: 800; letter-spacing: 0.08em; }
.demo-live-indicator i { width: 7px; height: 7px; background: #79e8a9; border-radius: 50%; box-shadow: 0 0 0 5px rgba(121, 232, 169, 0.12); }
.demo-metric-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 14px; }
.demo-metric-card { display: grid; gap: 9px; padding: 22px; box-shadow: none; }
.demo-metric-card > span { color: var(--demo-muted); font-size: 0.7rem; }
.demo-metric-card strong { color: var(--demo-heading); font-size: 1.8rem; letter-spacing: -0.04em; }
.demo-metric-card small { color: #79e8a9; font-family: 'IBM Plex Mono', monospace; font-size: 0.62rem; }
.demo-metric-card small.warning { color: #ffc36b; }
.demo-dashboard-grid { display: grid; grid-template-columns: 1.2fr 0.8fr; gap: 14px; min-width: 0; margin-top: 14px; }
.demo-dashboard-grid > * { min-width: 0; }
.demo-panel { padding: clamp(22px, 3vw, 30px); box-shadow: none; }
.demo-panel-heading { display: flex; align-items: start; justify-content: space-between; gap: 16px; padding-bottom: 20px; border-bottom: 1px solid var(--demo-divider); }
.demo-panel-heading h2 { color: var(--demo-heading); font-size: 1.3rem; }
.demo-service-list { display: grid; }
.demo-service-row { display: grid; grid-template-columns: 10px 1fr auto; align-items: center; gap: 12px; padding: 17px 0; border-bottom: 1px solid var(--demo-divider); }
.demo-service-row:last-child { border-bottom: 0; }
.demo-service-dot { width: 8px; height: 8px; background: #79e8a9; border-radius: 50%; }
.demo-service-dot.degraded { background: #ffc36b; }
.demo-service-row strong,
.demo-service-row small { display: block; }
.demo-service-row strong { color: var(--demo-text); font-size: 0.8rem; }
.demo-service-row small { margin-top: 4px; color: var(--demo-muted); font-size: 0.65rem; }
.demo-service-row b { color: #c7e6f5; font-family: 'IBM Plex Mono', monospace; font-size: 0.66rem; }
.demo-event-list { display: grid; gap: 2px; padding-top: 9px; list-style: none; }
.demo-event-list li { display: grid; grid-template-columns: 44px 1fr auto; align-items: center; gap: 10px; padding: 13px 0; border-bottom: 1px solid var(--demo-divider); }
.demo-event-list li:last-child { border-bottom: 0; }
.demo-event-list span { color: var(--demo-muted); font-family: 'IBM Plex Mono', monospace; font-size: 0.62rem; }
.demo-event-list strong { color: var(--demo-text); font-size: 0.72rem; font-weight: 600; }
.demo-event-list em { color: #79e8a9; font-family: 'IBM Plex Mono', monospace; font-size: 0.56rem; font-style: normal; }
.demo-event-list em.warning { color: #ffc36b; }
.demo-bar-chart { display: flex; align-items: end; gap: 10px; height: 180px; padding-top: 28px; }
.demo-bar-chart span { flex: 1; height: var(--bar-size); min-height: 20px; background: linear-gradient(180deg, #a7edff, #437ba4); border-radius: 7px 7px 2px 2px; }
.demo-trend-summary { display: grid; gap: 8px; min-height: 180px; align-content: center; }
.demo-trend-summary strong { color: #79e8a9; font-size: clamp(2.1rem, 4vw, 3.2rem); letter-spacing: -0.06em; }
.demo-trend-summary span { max-width: 28ch; color: var(--demo-muted); font-size: 0.76rem; line-height: 1.6; }
.demo-record-panel { padding-block: 8px; }
.demo-record-row { display: grid; grid-template-columns: 112px minmax(0, 1fr) auto; align-items: center; gap: 14px; padding: 18px 0; border-bottom: 1px solid var(--demo-divider); }
.demo-record-row:last-child { border-bottom: 0; }
.demo-record-row strong,
.demo-record-row small { display: block; }
.demo-record-row strong { color: var(--demo-text); font-size: 0.78rem; }
.demo-record-row small { margin-top: 5px; color: var(--demo-muted); font-size: 0.65rem; }
.demo-record-row b { color: #ffc36b; font-family: 'IBM Plex Mono', monospace; font-size: 0.62rem; }
.demo-record-row b.resolved,
.demo-record-row b.active { color: #79e8a9; }
.demo-record-id { color: var(--demo-muted); font-family: 'IBM Plex Mono', monospace; font-size: 0.62rem; }
.demo-alert-icon { display: grid; width: 28px; height: 28px; color: #ffc36b; border: 1px solid currentColor; border-radius: 8px; place-items: center; }
.demo-settings-grid { display: grid; grid-template-columns: repeat(2, minmax(0, 1fr)); gap: 14px; }
.demo-setting-row { display: flex; align-items: center; justify-content: space-between; gap: 16px; padding: 16px 0; border-bottom: 1px solid var(--demo-divider); color: var(--demo-text); font-size: 0.76rem; }
.demo-setting-row:last-child { border-bottom: 0; }
.demo-setting-row b { color: #79e8a9; font-family: 'IBM Plex Mono', monospace; font-size: 0.58rem; font-weight: 700; text-transform: uppercase; }
.demo-dashboard-footer { justify-content: space-between; margin-top: 28px; padding: 0 clamp(20px, 4vw, 48px) 28px 28px; color: var(--demo-muted); font-size: 0.68rem; }
.demo-dashboard-footer > span { display: inline-flex; align-items: center; gap: 9px; }
.demo-page.is-light .demo-eyebrow { color: #0b638e; }
.demo-page.is-light .demo-badge,
.demo-page.is-light .demo-panel-count { color: #45657d; }
.demo-page.is-light .demo-live-indicator { color: #16734c; }

@media (max-width: 820px) {
  .demo-dashboard-grid,
  .demo-dashboard-shell { grid-template-columns: 1fr; }
  .demo-dashboard-shell { grid-template-rows: auto auto auto; }
  .demo-header { grid-column: 1; grid-row: 1; }
  .demo-sidebar { grid-column: 1; grid-row: 2; }
  .demo-dashboard { grid-column: 1; grid-row: 3; }
  .demo-settings-grid { grid-template-columns: 1fr; }
  .demo-auth-screen { grid-template-columns: 1fr; grid-template-rows: minmax(150px, 28svh) minmax(0, 1fr); }
  .demo-auth-visual { height: auto; min-height: 0; }
  .demo-auth-layout { height: auto; min-height: 0; padding: 32px 24px; }
  .demo-auth-form { width: 100%; max-width: 640px; min-height: auto; padding: 0; }
  .demo-auth-card-header { align-items: flex-start; }
  .demo-auth-card-actions { width: 100%; justify-content: space-between; }
  .demo-metric-grid { grid-template-columns: repeat(2, 1fr); }
  .demo-dashboard-shell { gap: 14px; padding-top: 0; }
  .demo-dashboard { padding: 0; }
  .demo-sidebar { position: static; height: auto; min-height: 0; }
  .demo-intro > p:not(.demo-eyebrow) { white-space: normal; }
  .demo-sidebar-nav { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); overflow: visible; padding-bottom: 3px; }
  .demo-sidebar-link { width: 100%; }
  .demo-sidebar-label { display: none; }
  .demo-sidebar-footer { margin-top: 18px; }
}

@media (max-width: 560px) {
  .demo-page { padding-inline: 16px; }
  .demo-page:not(.is-login-screen) { padding-inline: 16px; }
  .demo-header,
  .demo-view-heading,
  .demo-dashboard-footer { align-items: flex-start; flex-direction: column; }
  .demo-header-actions { align-items: flex-start; flex-direction: column; gap: 10px; }
  .demo-sidebar-nav { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  .demo-auth-visual { min-height: 0; }
  .demo-auth-visual-overlay { align-items: flex-start; flex-direction: column; }
  .demo-auth-layout { padding: 12px 16px; }
  .demo-auth-form { width: 100%; min-height: 0; }
  .demo-auth-card-header { gap: 12px; padding-bottom: 12px; }
  .demo-auth-card-actions { align-items: stretch; }
  .demo-auth-card-actions .demo-theme-toggle,
  .demo-auth-card-actions .demo-back-link { flex: 1 1 0; justify-content: center; }
  .demo-auth-card-body { gap: 10px; padding-top: 12px; }
  .demo-auth-tabs { padding: 3px; }
  .demo-tab { padding: 7px; }
  .demo-auth-form h2 { font-size: clamp(1.3rem, 6vw, 1.6rem); }
  .demo-auth-form label { gap: 4px; }
  .demo-auth-form input { padding: 9px 11px; font-size: 16px; }
  .demo-primary-button { padding: 11px 14px; }
  .demo-metric-grid { grid-template-columns: 1fr 1fr; gap: 9px; }
  .demo-metric-card { padding: 16px 13px; }
  .demo-metric-card strong { font-size: 1.35rem; }
}

@media (max-width: 560px) and (max-height: 700px) {
  .demo-auth-screen { grid-template-rows: 120px minmax(0, 1fr); }
  .demo-auth-visual-overlay { right: 16px; bottom: 12px; left: 16px; gap: 8px; font-size: 0.5rem; }
  .demo-auth-layout { padding-block: 8px; }
  .demo-auth-card-header { gap: 8px; padding-bottom: 8px; }
  .demo-auth-card-body { gap: 6px; padding-top: 8px; }
  .demo-auth-form h2 { font-size: 1.2rem; }
  .demo-card-copy { font-size: 0.65rem; }
  .demo-auth-form input { padding-block: 7px; }
  .demo-primary-button { padding-block: 9px; }
}
</style>

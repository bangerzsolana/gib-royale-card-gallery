<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { priceService } from './services/PriceService.js'
import GameCard from './components/GameCard.vue'
import CardDetail from './components/CardDetail.vue'

const tokens = ref([])
const loading = ref(true)
const error = ref(null)
const selectedSymbol = ref(null)
const activeRole = ref('all')
const activeCategory = ref('all')
const sortBy = ref('power')
const searchQuery = ref('')
let unsubscribe = null
let refreshTimer = null

const roles = ['all', 'Tank', 'Fighter', 'Glass Cannon']
const categories = ['all', 'Meme', 'DeFi', 'Stock', 'Other']
const sorts = [
  { value: 'power', label: 'Power' },
  { value: 'hp', label: 'HP' },
  { value: 'damage', label: 'Damage' },
  { value: 'speed', label: 'Speed' },
  { value: 'marketCap', label: 'Market Cap' },
  { value: 'price', label: 'Price' },
  { value: 'name', label: 'Name' },
]

const filteredTokens = computed(() => {
  let list = [...tokens.value]

  // Search
  if (searchQuery.value) {
    const q = searchQuery.value.toUpperCase()
    list = list.filter(t => t.id.includes(q))
  }

  // Role filter
  if (activeRole.value !== 'all') {
    list = list.filter(t => priceService.getTokenRole(t.id) === activeRole.value)
  }

  // Category filter
  if (activeCategory.value !== 'all') {
    list = list.filter(t => priceService.getCategory(t.id) === activeCategory.value)
  }

  // Sort
  list.sort((a, b) => {
    switch (sortBy.value) {
      case 'power': return Math.abs(b.power) - Math.abs(a.power)
      case 'hp': return (priceService.getBaseHP(b.id)) - (priceService.getBaseHP(a.id))
      case 'damage': return (priceService.getBaseDamage(b.id)) - (priceService.getBaseDamage(a.id))
      case 'speed': return (priceService.getSpeedMultiplier(b.id)) - (priceService.getSpeedMultiplier(a.id))
      case 'marketCap': return (priceService.getMarketCap(b.id)) - (priceService.getMarketCap(a.id))
      case 'price': return (b.price || 0) - (a.price || 0)
      case 'name': return a.id.localeCompare(b.id)
      default: return b.power - a.power
    }
  })

  return list
})

const stats = computed(() => ({
  total: tokens.value.length,
  shown: filteredTokens.value.length,
  tanks: tokens.value.filter(t => priceService.getTokenRole(t.id) === 'Tank').length,
  fighters: tokens.value.filter(t => priceService.getTokenRole(t.id) === 'Fighter').length,
  cannons: tokens.value.filter(t => priceService.getTokenRole(t.id) === 'Glass Cannon').length,
  pumping: tokens.value.filter(t => t.power > 0).length,
  dumping: tokens.value.filter(t => t.power < 0).length,
}))

function refreshTokens() {
  tokens.value = priceService.getAllTokensWithPower()
}

onMounted(async () => {
  try {
    const [pythOk, railwayOk] = await Promise.all([
      priceService.fetchPrices(),
      priceService.fetchRailwayPrices(),
    ])
    await priceService.fetchAllMarketCaps()

    if (!pythOk && !railwayOk) {
      error.value = 'Failed to fetch prices. Check connection.'
      loading.value = false
      return
    }

    refreshTokens()
    loading.value = false

    unsubscribe = priceService.onPriceUpdate(() => {
      refreshTokens()
    })

    priceService.startPolling(3000)

    // Also refresh the full list periodically (catches market cap updates)
    refreshTimer = setInterval(refreshTokens, 5000)
  } catch (e) {
    error.value = e.message
    loading.value = false
  }
})

onUnmounted(() => {
  if (unsubscribe) unsubscribe()
  if (refreshTimer) clearInterval(refreshTimer)
  priceService.stopPolling()
})
</script>

<template>
  <div class="app">
    <!-- Header -->
    <header class="app-header">
      <h1>GIB ROYALE CARDS</h1>
      <p class="subtitle">Live card stats powered by Pyth Network + Railway API</p>
    </header>

    <!-- Stats bar -->
    <div class="stats-bar" v-if="!loading">
      <span>{{ stats.total }} coins</span>
      <span class="sep">|</span>
      <span style="color:#4488ff">{{ stats.tanks }} Tanks</span>
      <span class="sep">|</span>
      <span style="color:#ff8844">{{ stats.fighters }} Fighters</span>
      <span class="sep">|</span>
      <span style="color:#ff4444">{{ stats.cannons }} Glass Cannons</span>
      <span class="sep">|</span>
      <span style="color:#44ff44">{{ stats.pumping }} pumping</span>
      <span class="sep">|</span>
      <span style="color:#ff4444">{{ stats.dumping }} dumping</span>
    </div>

    <!-- Controls -->
    <div class="controls" v-if="!loading">
      <!-- Search -->
      <input
        v-model="searchQuery"
        class="search-input"
        placeholder="Search coins..."
        type="text"
      />

      <!-- Role filter -->
      <div class="filter-row">
        <span class="filter-label">Role:</span>
        <button
          v-for="r in roles"
          :key="r"
          class="filter-btn"
          :class="{ active: activeRole === r }"
          @click="activeRole = r"
        >
          {{ r === 'all' ? 'ALL' : r.toUpperCase() }}
        </button>
      </div>

      <!-- Category filter -->
      <div class="filter-row">
        <span class="filter-label">Category:</span>
        <button
          v-for="c in categories"
          :key="c"
          class="filter-btn"
          :class="{ active: activeCategory === c }"
          @click="activeCategory = c"
        >
          {{ c === 'all' ? 'ALL' : c.toUpperCase() }}
        </button>
      </div>

      <!-- Sort -->
      <div class="filter-row">
        <span class="filter-label">Sort:</span>
        <button
          v-for="s in sorts"
          :key="s.value"
          class="filter-btn"
          :class="{ active: sortBy === s.value }"
          @click="sortBy = s.value"
        >
          {{ s.label.toUpperCase() }}
        </button>
      </div>

      <div class="showing">Showing {{ filteredTokens.length }} of {{ stats.total }} cards</div>
    </div>

    <!-- Loading -->
    <div v-if="loading" class="loading">
      <div class="spinner"></div>
      <p>Fetching live data from Pyth Network...</p>
    </div>

    <!-- Error -->
    <div v-if="error" class="error-msg">{{ error }}</div>

    <!-- Card Grid -->
    <div class="card-grid" v-if="!loading && !error">
      <GameCard
        v-for="token in filteredTokens"
        :key="token.id"
        :token="token"
        @select="selectedSymbol = $event"
      />
    </div>

    <!-- Detail Modal -->
    <CardDetail
      v-if="selectedSymbol"
      :symbol="selectedSymbol"
      @close="selectedSymbol = null"
    />

    <!-- Mechanics Legend -->
    <footer class="legend" v-if="!loading">
      <h3>HOW CARDS WORK</h3>
      <div class="legend-grid">
        <div class="legend-item">
          <strong>HP</strong> = Market Cap (bigger coins = more health)
        </div>
        <div class="legend-item">
          <strong>DMG</strong> = 7-day Volatility (volatile coins hit harder)
        </div>
        <div class="legend-item">
          <strong>Speed</strong> = Inverse Market Cap (small coins are faster)
        </div>
        <div class="legend-item">
          <strong>Power</strong> = Real-time price momentum vs EMA
        </div>
        <div class="legend-item">
          <strong>ATK boost</strong> = Coin is pumping (positive momentum)
        </div>
        <div class="legend-item">
          <strong>DEF boost</strong> = Coin is dumping (negative momentum)
        </div>
        <div class="legend-item">
          <strong>Rug Pull</strong> = Dumps >25% -> unit explodes (area damage)
        </div>
        <div class="legend-item">
          <strong>Moon Shot</strong> = Pumps >50% -> 2x damage buff (5 sec)
        </div>
      </div>
    </footer>
  </div>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  background: #0d0d1a;
  color: #fff;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  -webkit-font-smoothing: antialiased;
}

.app {
  max-width: 1200px;
  margin: 0 auto;
  padding: 16px;
}

.app-header {
  text-align: center;
  padding: 24px 0 12px;
}
.app-header h1 {
  font-size: 28px;
  font-weight: 900;
  letter-spacing: 2px;
  background: linear-gradient(135deg, #ffcc00, #ff6600);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
}
.subtitle {
  font-size: 12px;
  color: #666;
  margin-top: 4px;
}

.stats-bar {
  display: flex;
  gap: 8px;
  justify-content: center;
  font-size: 12px;
  color: #888;
  padding: 8px 0;
  flex-wrap: wrap;
}
.sep { color: #333; }

.controls {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 12px 0;
  border-top: 1px solid #222;
  border-bottom: 1px solid #222;
  margin-bottom: 16px;
}

.search-input {
  background: #1a1a2e;
  border: 1px solid #333;
  border-radius: 8px;
  padding: 8px 12px;
  color: #fff;
  font-size: 14px;
  outline: none;
  width: 100%;
  max-width: 300px;
}
.search-input:focus { border-color: #ffcc00; }

.filter-row {
  display: flex;
  align-items: center;
  gap: 6px;
  flex-wrap: wrap;
}
.filter-label {
  font-size: 11px;
  color: #666;
  font-weight: 700;
  width: 70px;
  flex-shrink: 0;
}
.filter-btn {
  font-size: 10px;
  font-weight: 700;
  padding: 4px 10px;
  border-radius: 12px;
  border: 1px solid #444;
  background: #1a1a2e;
  color: #888;
  cursor: pointer;
  transition: all 0.15s;
}
.filter-btn:hover { border-color: #666; color: #ccc; }
.filter-btn.active {
  background: #ffcc00;
  color: #000;
  border-color: #ffcc00;
}

.showing {
  font-size: 11px;
  color: #555;
}

.loading {
  text-align: center;
  padding: 80px 20px;
  color: #888;
}
.spinner {
  width: 32px;
  height: 32px;
  border: 3px solid #333;
  border-top-color: #ffcc00;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
  margin: 0 auto 16px;
}
@keyframes spin { to { transform: rotate(360deg); } }

.error-msg {
  text-align: center;
  color: #ff4444;
  padding: 40px;
}

.card-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  gap: 12px;
}

@media (min-width: 768px) {
  .card-grid {
    grid-template-columns: repeat(auto-fill, minmax(165px, 1fr));
  }
}

.legend {
  margin-top: 32px;
  padding: 20px;
  background: #111122;
  border-radius: 12px;
  border: 1px solid #222;
}
.legend h3 {
  font-size: 12px;
  color: #666;
  letter-spacing: 2px;
  margin-bottom: 12px;
}
.legend-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 8px;
}
.legend-item {
  font-size: 12px;
  color: #888;
  line-height: 1.6;
}
.legend-item strong {
  color: #ffcc00;
}
</style>

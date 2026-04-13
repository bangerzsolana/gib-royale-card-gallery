<template>
  <div class="detail-overlay" @click.self="$emit('close')">
    <div class="detail-card" :style="cardBg">
      <button class="close-btn" @click="$emit('close')">X</button>

      <!-- Header -->
      <div class="detail-header">
        <div class="coin-circle-lg" :style="{ background: coinColor }">
          <span class="coin-name-lg">{{ symbol }}</span>
        </div>
        <div class="header-info">
          <h2 class="symbol-title">{{ symbol }}</h2>
          <div class="role-tag" :style="{ color: roleColor, borderColor: roleColor }">{{ role }}</div>
          <div class="category-tag">{{ category }}</div>
          <div class="price-big">{{ formattedPrice }}</div>
          <div class="source-tag">{{ sourceLabel }}</div>
        </div>
      </div>

      <!-- Power Section -->
      <div class="section">
        <h3>Live Power</h3>
        <div class="power-big" :style="{ color: powerColor }">
          {{ powerLabel }} {{ powerDisplay }}
        </div>
        <p class="explain">
          Power = ((price - EMA) / EMA) x 1000. Positive = pumping (ATK boost). Negative = dumping (DEF boost).
        </p>
      </div>

      <!-- Base Combat Stats -->
      <div class="section">
        <h3>Base Combat Stats</h3>
        <div class="stat-grid">
          <div class="stat-block">
            <div class="stat-label">HP</div>
            <div class="stat-big">{{ stats.hp }}</div>
            <div class="stat-source">from Market Cap</div>
            <div class="stat-detail">{{ formatMarketCap(stats.marketCap) }}</div>
          </div>
          <div class="stat-block">
            <div class="stat-label">Base DMG</div>
            <div class="stat-big">{{ stats.damage }}</div>
            <div class="stat-source">from 7d Volatility</div>
            <div class="stat-detail">{{ stats.volatility.toFixed(1) }}% abs change</div>
          </div>
          <div class="stat-block">
            <div class="stat-label">Speed</div>
            <div class="stat-big">{{ stats.speedMultiplier.toFixed(2) }}x</div>
            <div class="stat-source">from Inv. Market Cap</div>
            <div class="stat-detail">{{ speedDescription }}</div>
          </div>
        </div>
      </div>

      <!-- Market Cap Tier -->
      <div class="section">
        <h3>Market Cap Tiers (HP)</h3>
        <div class="tier-chart">
          <div v-for="tier in hpTiers" :key="tier.label" class="tier-row" :class="{ active: tier.active }">
            <span class="tier-label">{{ tier.label }}</span>
            <div class="tier-bar" :style="{ width: tier.width + '%', background: tier.color }"></div>
            <span class="tier-hp">{{ tier.hp }} HP</span>
          </div>
        </div>
      </div>

      <!-- Real-Time Multipliers -->
      <div class="section">
        <h3>Real-Time Multipliers (NOW)</h3>
        <div class="realtime-box" :style="realtimeBg">
          <div class="rt-label">{{ realtime.label }}</div>
          <div class="rt-stats">
            <span>DMG x{{ realtime.dmgMult.toFixed(2) }}</span>
            <span>DEF x{{ realtime.defMult.toFixed(2) }}</span>
          </div>
          <div class="rt-effective">
            Effective DMG: <strong>{{ effectiveDmg }}</strong> |
            Effective HP: <strong>{{ effectiveHP }}</strong>
          </div>
          <div v-if="realtime.specialEvent" class="special-event">
            {{ realtime.specialEvent === 'Rug Pull' ? 'UNIT EXPLODES! 50 area damage to nearby troops' : '2x DAMAGE BUFF for 5 seconds (gold tint)' }}
          </div>
        </div>
      </div>

      <!-- Multiplier Thresholds -->
      <div class="section">
        <h3>Multiplier Thresholds</h3>
        <table class="threshold-table">
          <thead>
            <tr><th>Price Action</th><th>DMG Mult</th><th>DEF Mult</th><th>Feel</th></tr>
          </thead>
          <tbody>
            <tr :class="{ highlight: realtime.pct > 10 }">
              <td style="color:#44ff44">Pumping >10%</td><td>2-3x</td><td>0.8x</td><td>Glass Cannon</td>
            </tr>
            <tr :class="{ highlight: realtime.pct > 1 && realtime.pct <= 10 }">
              <td style="color:#88ff88">Steady 1-10%</td><td>1-2x</td><td>1x</td><td>Attacker</td>
            </tr>
            <tr :class="{ highlight: realtime.pct >= -1 && realtime.pct <= 1 }">
              <td style="color:#888">Crabbing</td><td>1x</td><td>1x</td><td>Neutral</td>
            </tr>
            <tr :class="{ highlight: realtime.pct < -1 && realtime.pct >= -10 }">
              <td style="color:#ff8888">Dipping -1 to -10%</td><td>0.5x</td><td>1.5-2x</td><td>Shield</td>
            </tr>
            <tr :class="{ highlight: realtime.pct < -10 }">
              <td style="color:#ff4444">Dumping <-10%</td><td>0.25x</td><td>2-3x</td><td>Fortress</td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Special Events -->
      <div class="section">
        <h3>Special Events</h3>
        <div class="event-row">
          <div class="event-card rug" :class="{ triggered: realtime.specialEvent === 'Rug Pull' }">
            <div class="event-name">Rug Pull</div>
            <div class="event-trigger">Token dumps >25%</div>
            <div class="event-effect">Unit explodes, 50 area damage to ALL nearby troops (both sides)</div>
          </div>
          <div class="event-card moon" :class="{ triggered: realtime.specialEvent === 'Moon Shot' }">
            <div class="event-name">Moon Shot</div>
            <div class="event-trigger">Token pumps >50%</div>
            <div class="event-effect">2x damage buff for 5 seconds, gold tint visual</div>
          </div>
        </div>
      </div>

      <!-- Elixir Cost -->
      <div class="section">
        <h3>Elixir Cost</h3>
        <p class="explain">
          Cost is derived from combined power (HP + DMG x 3). Higher power = more expensive to deploy.
          Pumping coins also cost more (price modifier on elixir).
        </p>
        <div class="elixir-display">
          <div class="elixir-circle" :style="{ background: elixirColor }">{{ elixirCost }}</div>
          <span>elixir to deploy</span>
        </div>
      </div>

      <!-- Data Sources -->
      <div class="section">
        <h3>Data Sources</h3>
        <div class="data-sources">
          <div><strong>Price:</strong> {{ token?.source === 'railway' ? 'Railway API (CMC/CoinGecko)' : 'Pyth Network (real-time)' }}</div>
          <div><strong>Market Cap:</strong> Railway API (refreshes every 5 min)</div>
          <div><strong>Volatility:</strong> 7-day % change from Railway</div>
          <div><strong>Power:</strong> {{ token?.source === 'railway' ? 'Pre-computed by cron service' : 'EMA momentum from Pyth' }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed } from 'vue'
import { priceService } from '../services/PriceService.js'

const props = defineProps({
  symbol: { type: String, required: true },
})

defineEmits(['close'])

const token = computed(() => priceService.getTokenWithPower(props.symbol))
const coinColor = computed(() => priceService.getColor(props.symbol))
const role = computed(() => priceService.getTokenRole(props.symbol))
const category = computed(() => priceService.getCategory(props.symbol))
const stats = computed(() => priceService.getCombatStats(props.symbol) || { hp: 80, damage: 12, speedMultiplier: 1, marketCap: 0, volatility: 0 })
const realtime = computed(() => priceService.getRealtimeMultipliers(props.symbol))
const power = computed(() => token.value?.power || 0)
const powerColor = computed(() => power.value >= 0 ? '#44ff44' : '#ff4444')
const powerLabel = computed(() => power.value >= 0 ? 'ATK' : 'DEF')
const powerDisplay = computed(() => (power.value >= 0 ? '+' : '') + power.value.toFixed(1))

const roleColor = computed(() => {
  const map = { Tank: '#4488ff', Fighter: '#ff8844', 'Glass Cannon': '#ff4444' }
  return map[role.value] || '#fff'
})

const sourceLabel = computed(() => token.value?.source === 'railway' ? 'Railway API' : 'Pyth Network')

const formattedPrice = computed(() => {
  const p = token.value?.price
  if (!p) return '$—'
  if (p >= 100) return '$' + p.toFixed(2)
  if (p >= 1) return '$' + p.toFixed(4)
  if (p >= 0.01) return '$' + p.toFixed(6)
  return '$' + p.toExponential(3)
})

const effectiveDmg = computed(() => Math.round(stats.value.damage * realtime.value.dmgMult))
const effectiveHP = computed(() => Math.round(stats.value.hp * realtime.value.defMult))

const speedDescription = computed(() => {
  const s = stats.value.speedMultiplier
  if (s >= 1.3) return 'Zippy (micro-cap)'
  if (s >= 1.1) return 'Fast (small-cap)'
  if (s >= 0.9) return 'Normal (mid-cap)'
  if (s >= 0.7) return 'Heavy (large-cap)'
  return 'Slow (mega-cap)'
})

const hpTiers = computed(() => {
  const cap = stats.value.marketCap
  const logCap = cap > 0 ? Math.log10(cap) : 0
  return [
    { label: '>$50B', hp: 300, width: 100, color: '#4488ff', active: logCap >= 10.7 },
    { label: '$5B-$50B', hp: '200-300', width: 83, color: '#44aaff', active: logCap >= 9.7 && logCap < 10.7 },
    { label: '$500M-$5B', hp: '120-200', width: 66, color: '#ff8844', active: logCap >= 8.7 && logCap < 9.7 },
    { label: '$50M-$500M', hp: '70-120', width: 40, color: '#ffaa44', active: logCap >= 7.7 && logCap < 8.7 },
    { label: '$5M-$50M', hp: '40-70', width: 23, color: '#ff4444', active: logCap >= 6.7 && logCap < 7.7 },
    { label: '<$5M', hp: 20, width: 7, color: '#ff2222', active: logCap < 6.7 && logCap > 0 },
  ]
})

const elixirCost = computed(() => {
  const hp = stats.value.hp || 80
  const dmg = stats.value.damage || 12
  const total = hp + dmg * 3
  if (total >= 350) return 7
  if (total >= 280) return 6
  if (total >= 200) return 5
  if (total >= 140) return 4
  if (total >= 80) return 3
  return 2
})

const elixirColor = computed(() => {
  const costs = { 2: '#9b59b6', 3: '#3498db', 4: '#2ecc71', 5: '#f39c12', 6: '#e74c3c', 7: '#8e44ad' }
  return costs[elixirCost.value] || '#3498db'
})

const cardBg = computed(() => ({
  borderColor: `${coinColor.value}88`,
}))

const realtimeBg = computed(() => {
  const r = realtime.value
  if (r.specialEvent === 'Rug Pull') return { background: 'rgba(255,0,0,0.15)', border: '1px solid #ff0000' }
  if (r.specialEvent === 'Moon Shot') return { background: 'rgba(255,215,0,0.15)', border: '1px solid #ffd700' }
  if (r.pct > 1) return { background: 'rgba(68,255,68,0.08)', border: '1px solid rgba(68,255,68,0.3)' }
  if (r.pct < -1) return { background: 'rgba(255,68,68,0.08)', border: '1px solid rgba(255,68,68,0.3)' }
  return { background: 'rgba(255,255,255,0.03)', border: '1px solid #333' }
})

function formatMarketCap(cap) {
  if (!cap) return 'N/A'
  if (cap >= 1e12) return '$' + (cap / 1e12).toFixed(1) + 'T'
  if (cap >= 1e9) return '$' + (cap / 1e9).toFixed(1) + 'B'
  if (cap >= 1e6) return '$' + (cap / 1e6).toFixed(1) + 'M'
  if (cap >= 1e3) return '$' + (cap / 1e3).toFixed(0) + 'K'
  return '$' + cap.toFixed(0)
}
</script>

<style scoped>
.detail-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0,0,0,0.8);
  z-index: 1000;
  display: flex;
  justify-content: center;
  align-items: flex-start;
  overflow-y: auto;
  padding: 20px;
}

.detail-card {
  background: #1a1a2e;
  border: 2px solid;
  border-radius: 16px;
  max-width: 560px;
  width: 100%;
  padding: 24px;
  position: relative;
  color: #fff;
}

.close-btn {
  position: absolute;
  top: 12px;
  right: 12px;
  background: none;
  border: 1px solid #555;
  color: #fff;
  font-size: 14px;
  font-weight: 800;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
}
.close-btn:hover { background: #333; }

.detail-header {
  display: flex;
  gap: 16px;
  align-items: center;
  margin-bottom: 20px;
}

.coin-circle-lg {
  width: 72px;
  height: 72px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
.coin-name-lg {
  font-size: 12px;
  font-weight: 800;
  color: #fff;
  text-shadow: 0 1px 3px rgba(0,0,0,0.5);
}

.header-info { display: flex; flex-direction: column; gap: 4px; }
.symbol-title { font-size: 24px; font-weight: 800; margin: 0; }
.role-tag { font-size: 11px; font-weight: 800; letter-spacing: 1px; border: 1px solid; padding: 1px 8px; border-radius: 8px; width: fit-content; }
.category-tag { font-size: 10px; color: #888; }
.price-big { font-size: 16px; font-weight: 600; color: #ddd; font-variant-numeric: tabular-nums; }
.source-tag { font-size: 9px; color: #666; }

.section { margin-bottom: 20px; }
.section h3 { font-size: 13px; font-weight: 700; color: #888; text-transform: uppercase; letter-spacing: 1px; margin: 0 0 8px 0; border-bottom: 1px solid #333; padding-bottom: 4px; }

.power-big { font-size: 32px; font-weight: 800; font-variant-numeric: tabular-nums; }
.explain { font-size: 11px; color: #666; line-height: 1.5; margin: 4px 0 0; }

.stat-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 12px; }
.stat-block { text-align: center; background: rgba(255,255,255,0.03); border-radius: 8px; padding: 12px 8px; }
.stat-label { font-size: 10px; color: #888; font-weight: 700; letter-spacing: 1px; }
.stat-big { font-size: 28px; font-weight: 800; color: #fff; }
.stat-source { font-size: 9px; color: #555; margin-top: 4px; }
.stat-detail { font-size: 10px; color: #777; }

.tier-chart { display: flex; flex-direction: column; gap: 4px; }
.tier-row { display: flex; align-items: center; gap: 8px; padding: 3px 0; opacity: 0.5; }
.tier-row.active { opacity: 1; background: rgba(255,255,255,0.05); border-radius: 4px; padding: 3px 6px; }
.tier-label { font-size: 10px; color: #888; width: 90px; flex-shrink: 0; }
.tier-bar { height: 12px; border-radius: 3px; min-width: 4px; }
.tier-hp { font-size: 10px; color: #aaa; font-weight: 700; }

.realtime-box { padding: 12px; border-radius: 8px; text-align: center; }
.rt-label { font-size: 16px; font-weight: 800; margin-bottom: 4px; }
.rt-stats { display: flex; gap: 24px; justify-content: center; font-size: 14px; font-weight: 700; color: #ddd; }
.rt-effective { font-size: 12px; color: #aaa; margin-top: 8px; }
.special-event { font-size: 11px; color: #ff0; margin-top: 6px; font-weight: 700; }

.threshold-table { width: 100%; border-collapse: collapse; font-size: 11px; }
.threshold-table th { text-align: left; color: #666; font-weight: 600; padding: 4px 8px; border-bottom: 1px solid #333; }
.threshold-table td { padding: 4px 8px; color: #aaa; border-bottom: 1px solid #222; }
.threshold-table tr.highlight { background: rgba(255,255,255,0.05); }
.threshold-table tr.highlight td { color: #fff; font-weight: 700; }

.event-row { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
.event-card { border-radius: 8px; padding: 10px; opacity: 0.5; }
.event-card.triggered { opacity: 1; }
.event-card.rug { background: rgba(255,0,0,0.1); border: 1px solid rgba(255,0,0,0.3); }
.event-card.moon { background: rgba(255,215,0,0.1); border: 1px solid rgba(255,215,0,0.3); }
.event-name { font-size: 14px; font-weight: 800; margin-bottom: 2px; }
.event-card.rug .event-name { color: #ff4444; }
.event-card.moon .event-name { color: #ffd700; }
.event-trigger { font-size: 10px; color: #888; }
.event-effect { font-size: 10px; color: #aaa; margin-top: 4px; }

.elixir-display { display: flex; align-items: center; gap: 10px; margin-top: 8px; }
.elixir-circle { width: 36px; height: 36px; border-radius: 50%; display: flex; align-items: center; justify-content: center; font-size: 18px; font-weight: 800; color: #fff; }

.data-sources { font-size: 11px; color: #777; display: flex; flex-direction: column; gap: 4px; }
.data-sources strong { color: #aaa; }
</style>

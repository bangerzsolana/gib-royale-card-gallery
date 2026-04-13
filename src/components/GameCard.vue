<template>
  <div
    class="game-card"
    :style="cardStyle"
    @click="$emit('select', token.id)"
  >
    <!-- Elixir cost circle -->
    <div class="elixir-cost" :style="{ background: elixirColor }">
      {{ elixirCost }}
    </div>

    <!-- Coin circle -->
    <div class="coin-circle" :style="{ background: coinColor }">
      <span class="coin-symbol">{{ token.id }}</span>
    </div>

    <!-- Role badge -->
    <div class="role-badge" :style="{ color: roleColor }">{{ role }}</div>

    <!-- Stats row -->
    <div class="stats-row">
      <div class="stat">
        <span class="stat-icon">HP</span>
        <span class="stat-value">{{ stats.hp }}</span>
      </div>
      <div class="stat">
        <span class="stat-icon">DMG</span>
        <span class="stat-value">{{ stats.damage }}</span>
      </div>
      <div class="stat">
        <span class="stat-icon">SPD</span>
        <span class="stat-value">{{ speedDisplay }}</span>
      </div>
    </div>

    <!-- Power indicator -->
    <div class="power-indicator" :style="{ color: powerColor }">
      {{ powerLabel }} {{ powerDisplay }}
    </div>

    <!-- Live price -->
    <div class="price-line">{{ formattedPrice }}</div>

    <!-- Real-time multiplier bar -->
    <div class="multiplier-bar" :style="multBarStyle">
      {{ realtimeLabel }}
    </div>
  </div>
</template>

<script setup>
import { computed } from 'vue'
import { priceService } from '../services/PriceService.js'

const props = defineProps({
  token: { type: Object, required: true },
})

defineEmits(['select'])

const coinColor = computed(() => priceService.getColor(props.token.id))

const role = computed(() => priceService.getTokenRole(props.token.id))

const roleColor = computed(() => {
  const map = { Tank: '#4488ff', Fighter: '#ff8844', 'Glass Cannon': '#ff4444' }
  return map[role.value] || '#fff'
})

const stats = computed(() => priceService.getCombatStats(props.token.id) || { hp: 80, damage: 12, speedMultiplier: 1 })

const speedDisplay = computed(() => stats.value.speedMultiplier ? stats.value.speedMultiplier.toFixed(1) + 'x' : '1.0x')

const power = computed(() => props.token.power || 0)

const powerColor = computed(() => power.value >= 0 ? '#44ff44' : '#ff4444')

const powerLabel = computed(() => power.value >= 0 ? 'ATK' : 'DEF')

const powerDisplay = computed(() => (power.value >= 0 ? '+' : '') + power.value.toFixed(1))

const formattedPrice = computed(() => {
  const p = props.token.price
  if (!p) return '$—'
  if (p >= 100) return '$' + p.toFixed(0)
  if (p >= 1) return '$' + p.toFixed(2)
  if (p >= 0.01) return '$' + p.toFixed(4)
  if (p >= 0.0001) return '$' + p.toFixed(6)
  return '$' + p.toExponential(2)
})

const realtime = computed(() => priceService.getRealtimeMultipliers(props.token.id))

const realtimeLabel = computed(() => {
  const r = realtime.value
  if (r.specialEvent) return r.specialEvent
  return r.label
})

const multBarStyle = computed(() => {
  const r = realtime.value
  if (r.specialEvent === 'Rug Pull') return { background: '#ff0000', color: '#fff' }
  if (r.specialEvent === 'Moon Shot') return { background: '#ffd700', color: '#000' }
  if (r.pct > 1) return { background: 'rgba(68,255,68,0.2)', color: '#44ff44' }
  if (r.pct < -1) return { background: 'rgba(255,68,68,0.2)', color: '#ff4444' }
  return { background: 'rgba(255,255,255,0.05)', color: '#888' }
})

const elixirCost = computed(() => {
  // Elixir cost based on power quartiles (relative pricing)
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

const cardStyle = computed(() => {
  const color = coinColor.value
  return {
    background: `linear-gradient(135deg, ${color}33, ${color}11)`,
    borderColor: `${color}66`,
  }
})
</script>

<style scoped>
.game-card {
  border-radius: 12px;
  border: 1px solid;
  padding: 12px 10px 8px;
  cursor: pointer;
  transition: transform 0.15s, box-shadow 0.15s;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  position: relative;
  min-height: 200px;
}
.game-card:hover {
  transform: translateY(-4px);
  box-shadow: 0 8px 24px rgba(0,0,0,0.4);
}

.elixir-cost {
  position: absolute;
  top: 6px;
  left: 6px;
  width: 22px;
  height: 22px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 12px;
  font-weight: 800;
  color: #fff;
}

.coin-circle {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-top: 4px;
}
.coin-symbol {
  font-size: 9px;
  font-weight: 800;
  color: #fff;
  text-shadow: 0 1px 2px rgba(0,0,0,0.5);
  letter-spacing: 0.3px;
}

.role-badge {
  font-size: 9px;
  font-weight: 800;
  letter-spacing: 1px;
  text-transform: uppercase;
}

.stats-row {
  display: flex;
  gap: 8px;
  margin: 2px 0;
}
.stat {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1px;
}
.stat-icon {
  font-size: 8px;
  color: #888;
  font-weight: 700;
}
.stat-value {
  font-size: 13px;
  font-weight: 800;
  color: #fff;
  font-variant-numeric: tabular-nums;
}

.power-indicator {
  font-size: 12px;
  font-weight: 800;
  font-variant-numeric: tabular-nums;
}

.price-line {
  font-size: 10px;
  color: #888;
  font-variant-numeric: tabular-nums;
}

.multiplier-bar {
  font-size: 8px;
  font-weight: 700;
  padding: 2px 8px;
  border-radius: 8px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
  margin-top: 2px;
}
</style>

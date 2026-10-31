<template>
  <a-card :bordered="false" class="budget-dashboard-card section-shellless">
    <!-- 顶部总览 -->
    <div class="bd-header">
      <div class="bd-title-wrap">
        <h2 class="bd-title">预算看板</h2>
        <p class="bd-subtitle">预算与实际支出对比 · 分类占比 · 每日趋势 · 成员分摊结算</p>
      </div>
      <div class="bd-header-actions">
        <a-button type="default" size="small" @click="resetDemo">恢复初始数据</a-button>
      </div>
    </div>

    <!-- 四个关键指标 -->
    <div class="bd-kpi-grid">
      <div class="bd-kpi">
        <span class="bd-kpi-label">预算总额</span>
        <span class="bd-kpi-value">{{ money(totalBudget) }}</span>
      </div>
      <div class="bd-kpi">
        <span class="bd-kpi-label">已支出</span>
        <span class="bd-kpi-value">{{ money(totalSpent) }}</span>
      </div>
      <div class="bd-kpi">
        <span class="bd-kpi-label">剩余预算</span>
        <span class="bd-kpi-value" :class="remaining >= 0 ? 'is-good' : 'is-bad'">
          {{ money(remaining) }}
        </span>
      </div>
      <div class="bd-kpi">
        <span class="bd-kpi-label">预算使用率</span>
        <span class="bd-kpi-value" :class="usageRate > 100 ? 'is-bad' : 'is-good'">
          {{ usageRate.toFixed(1) }}%
        </span>
      </div>
    </div>

    <!-- 预算进度条 -->
    <div class="bd-progress-wrap">
      <div class="bd-progress-track">
        <div
          class="bd-progress-bar"
          :class="{ 'is-over': usageRate > 100 }"
          :style="{ width: Math.min(usageRate, 100) + '%' }"
        ></div>
      </div>
      <div class="bd-progress-meta">
        <span>已用 {{ money(totalSpent) }}</span>
        <span v-if="remaining >= 0">还可支出 {{ money(remaining) }}</span>
        <span v-else class="is-bad">已超支 {{ money(-remaining) }}</span>
      </div>
    </div>

    <!-- 主体两栏 -->
    <div class="bd-main-grid">
      <!-- 左：分类对比 + 每日趋势 -->
      <div class="bd-col">
        <div class="bd-panel">
          <div class="bd-panel-head">
            <h3>分类预算对比</h3>
            <span class="bd-panel-hint">预算 vs 实际</span>
          </div>
          <div class="bd-cat-list">
            <div v-for="row in categoryRows" :key="row.key" class="bd-cat-row">
              <div class="bd-cat-head">
                <span class="bd-cat-name">
                  <i class="bd-dot" :style="{ background: row.color }"></i>{{ row.label }}
                </span>
                <span class="bd-cat-num" :class="{ 'is-bad': row.spent > row.budget }">
                  {{ money(row.spent) }} / {{ money(row.budget) }}
                </span>
              </div>
              <div class="bd-cat-track">
                <div
                  class="bd-cat-budget"
                  :style="{ width: '100%', background: row.color + '33' }"
                ></div>
                <div
                  class="bd-cat-spent"
                  :style="{
                    width: Math.min(row.ratio, 100) + '%',
                    background: row.spent > row.budget ? '#ff6b6b' : row.color,
                  }"
                ></div>
              </div>
              <div class="bd-cat-foot">
                <span>占比 {{ row.share.toFixed(1) }}%</span>
                <span>{{ row.ratio.toFixed(0) }}%</span>
              </div>
            </div>
          </div>
        </div>

        <div class="bd-panel">
          <div class="bd-panel-head">
            <h3>每日支出趋势</h3>
            <span class="bd-panel-hint">按行程天数统计</span>
          </div>
          <div ref="trendChartRef" class="bd-chart"></div>
        </div>
      </div>

      <!-- 右：分类占比 + 成员结算 -->
      <div class="bd-col">
        <div class="bd-panel">
          <div class="bd-panel-head">
            <h3>支出分类占比</h3>
            <span class="bd-panel-hint">{{ expenses.length }} 笔支出</span>
          </div>
          <div ref="pieChartRef" class="bd-chart bd-chart-pie"></div>
        </div>

        <div class="bd-panel">
          <div class="bd-panel-head">
            <h3>成员分摊与结算建议</h3>
            <span class="bd-panel-hint">{{ members.length }} 人</span>
          </div>

          <div class="bd-member-list">
            <div v-for="m in memberRows" :key="m.name" class="bd-member-row">
              <span class="bd-member-name">{{ m.name }}</span>
              <span class="bd-member-cell">已付 {{ money(m.paid) }}</span>
              <span class="bd-member-cell">应摊 {{ money(m.owed) }}</span>
              <span
                class="bd-member-net"
                :class="m.net >= 0 ? 'is-good' : 'is-bad'"
              >
                {{ m.net >= 0 ? '应收 ' + money(m.net) : '应付 ' + money(-m.net) }}
              </span>
            </div>
          </div>

          <div v-if="settlements.length > 0" class="bd-settle-list">
            <div class="bd-settle-title">结算建议</div>
            <div v-for="(s, i) in settlements" :key="i" class="bd-settle-row">
              <span class="bd-settle-from">{{ s.from }}</span>
              <span class="bd-settle-arrow">→</span>
              <span class="bd-settle-to">{{ s.to }}</span>
              <span class="bd-settle-amount">{{ money(s.amount) }}</span>
            </div>
          </div>
          <a-empty v-else description="当前无需结算" :image-style="{ height: '48px' }" />
        </div>
      </div>
    </div>

    <!-- 支出录入 -->
    <div class="bd-panel bd-panel-form">
      <div class="bd-panel-head">
        <h3>记一笔支出</h3>
        <span class="bd-panel-hint">录入后自动参与对比、占比与结算</span>
      </div>

      <div class="bd-form-grid">
        <a-input v-model:value="form.title" placeholder="支出名称，如：兵马俑门票" class="bd-form-item" />
        <a-select v-model:value="form.category" class="bd-form-item">
          <a-select-option v-for="c in categoryOptions" :key="c.value" :value="c.value">
            {{ c.label }}
          </a-select-option>
        </a-select>
        <a-input-number
          v-model:value="form.amount"
          :min="0"
          :precision="2"
          placeholder="金额"
          class="bd-form-item"
          style="width: 100%"
        />
        <a-select v-model:value="form.dayIndex" class="bd-form-item">
          <a-select-option v-for="d in dayOptions" :key="d.value" :value="d.value">
            {{ d.label }}
          </a-select-option>
        </a-select>
        <a-select v-model:value="form.paidBy" placeholder="付款人" class="bd-form-item">
          <a-select-option v-for="m in members" :key="m" :value="m">{{ m }}</a-select-option>
        </a-select>
        <a-button type="primary" class="bd-form-item bd-submit" @click="addExpense">添加支出</a-button>
      </div>

      <div class="bd-share-row">
        <span class="bd-share-label">分摊成员</span>
        <a-checkbox-group v-model:value="form.sharedBy" :options="memberOptions" />
      </div>

      <div v-if="expenses.length > 0" class="bd-expense-list">
        <div class="bd-expense-head">
          <span>名称</span><span>分类</span><span>天数</span><span>付款人</span><span>金额</span><span>操作</span>
        </div>
        <div v-for="e in expenses" :key="e.id" class="bd-expense-row">
          <span class="bd-expense-title">{{ e.title }}</span>
          <span>{{ categoryLabel(e.category) }}</span>
          <span>第 {{ e.dayIndex }} 天</span>
          <span>{{ e.paidBy }}</span>
          <span class="bd-expense-amount">{{ money(e.amount) }}</span>
          <a class="bd-expense-del" @click="removeExpense(e.id)">删除</a>
        </div>
      </div>
      <a-empty v-else description="还没有支出记录，先记一笔试试" :image-style="{ height: '48px' }" />
    </div>
  </a-card>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch, nextTick } from 'vue'
import * as echarts from 'echarts'
import type { TripPlan } from '../types'

const props = defineProps<{
  tripPlan: TripPlan | null
  planId?: string
}>()

type Category = 'attraction' | 'hotel' | 'meal' | 'transport' | 'other'

interface Expense {
  id: string
  title: string
  category: Category
  amount: number
  dayIndex: number
  paidBy: string
  sharedBy: string[]
  createdAt: number
}

const STORAGE_PREFIX = 'tripstar-budget-dashboard-v1'

const categoryOptions: { value: Category; label: string; color: string }[] = [
  { value: 'attraction', label: '景点门票', color: '#6ee7ff' },
  { value: 'hotel', label: '住宿', color: '#a78bfa' },
  { value: 'meal', label: '餐饮', color: '#ffd5c6' },
  { value: 'transport', label: '交通', color: '#7ee787' },
  { value: 'other', label: '其他', color: '#f0b429' },
]

const defaultMembers = ['我', '同行 A', '同行 B']

const members = ref<string[]>([...defaultMembers])
const expenses = ref<Expense[]>([])

const form = reactive<{
  title: string
  category: Category
  amount: number | null
  dayIndex: number
  paidBy: string
  sharedBy: string[]
}>({
  title: '',
  category: 'attraction',
  amount: null,
  dayIndex: 1,
  paidBy: defaultMembers[0],
  sharedBy: [...defaultMembers],
})

const storageKey = computed(() => `${STORAGE_PREFIX}:${props.planId || 'local'}`)

const dayCount = computed(() => {
  const n = props.tripPlan?.days?.length || 0
  return n > 0 ? n : 1
})

const dayOptions = computed(() =>
  Array.from({ length: dayCount.value }, (_, i) => {
    const date = props.tripPlan?.days?.[i]?.date
    return {
      value: i + 1,
      label: `第 ${i + 1} 天${date ? '（' + date + '）' : ''}`,
    }
  }),
)

const memberOptions = computed(() => members.value.map((m) => ({ label: m, value: m })))

/* ---------------- 预算基线 ---------------- */

const budgetByCategory = computed<Record<Category, number>>(() => {
  const b = props.tripPlan?.budget
  return {
    attraction: Number(b?.total_attractions || 0),
    hotel: Number(b?.total_hotels || 0),
    meal: Number(b?.total_meals || 0),
    transport:
      Number(b?.total_transportation || 0) + Number(b?.total_inter_city_transport || 0),
    other: 0,
  }
})

const totalBudget = computed(() => Number(props.tripPlan?.budget?.total || 0))

/* ---------------- 支出汇总 ---------------- */

const totalSpent = computed(() => expenses.value.reduce((s, e) => s + e.amount, 0))
const remaining = computed(() => totalBudget.value - totalSpent.value)
const usageRate = computed(() =>
  totalBudget.value > 0 ? (totalSpent.value / totalBudget.value) * 100 : 0,
)

const spentByCategory = computed<Record<Category, number>>(() => {
  const acc: Record<Category, number> = {
    attraction: 0,
    hotel: 0,
    meal: 0,
    transport: 0,
    other: 0,
  }
  for (const e of expenses.value) acc[e.category] = (acc[e.category] || 0) + e.amount
  return acc
})

const categoryRows = computed(() =>
  categoryOptions.map((c) => {
    const budget = budgetByCategory.value[c.value] || 0
    const spent = spentByCategory.value[c.value] || 0
    return {
      key: c.value,
      label: c.label,
      color: c.color,
      budget,
      spent,
      ratio: budget > 0 ? (spent / budget) * 100 : spent > 0 ? 100 : 0,
      share: totalSpent.value > 0 ? (spent / totalSpent.value) * 100 : 0,
    }
  }),
)

/* ---------------- 成员分摊 ---------------- */

const memberRows = computed(() => {
  return members.value.map((name) => {
    let paid = 0
    let owed = 0
    for (const e of expenses.value) {
      if (e.paidBy === name) paid += e.amount
      const sharers = e.sharedBy.length > 0 ? e.sharedBy : members.value
      if (sharers.includes(name)) owed += e.amount / sharers.length
    }
    return { name, paid, owed, net: paid - owed }
  })
})

const settlements = computed(() => {
  const creditors = memberRows.value
    .filter((m) => m.net > 0.005)
    .map((m) => ({ name: m.name, amount: m.net }))
    .sort((a, b) => b.amount - a.amount)
  const debtors = memberRows.value
    .filter((m) => m.net < -0.005)
    .map((m) => ({ name: m.name, amount: -m.net }))
    .sort((a, b) => b.amount - a.amount)

  const out: { from: string; to: string; amount: number }[] = []
  let i = 0
  let j = 0
  while (i < debtors.length && j < creditors.length) {
    const pay = Math.min(debtors[i].amount, creditors[j].amount)
    if (pay > 0.005) out.push({ from: debtors[i].name, to: creditors[j].name, amount: pay })
    debtors[i].amount -= pay
    creditors[j].amount -= pay
    if (debtors[i].amount <= 0.005) i++
    if (creditors[j].amount <= 0.005) j++
  }
  return out
})

/* ---------------- 图表 ---------------- */

const trendChartRef = ref<HTMLDivElement | null>(null)
const pieChartRef = ref<HTMLDivElement | null>(null)
let trendChart: echarts.ECharts | null = null
let pieChart: echarts.ECharts | null = null

const dailyTotals = computed(() => {
  const arr = new Array(dayCount.value).fill(0)
  for (const e of expenses.value) {
    const idx = Math.min(Math.max(e.dayIndex - 1, 0), arr.length - 1)
    arr[idx] += e.amount
  }
  return arr
})

function renderTrend() {
  if (!trendChartRef.value) return
  if (!trendChart) trendChart = echarts.init(trendChartRef.value)
  const labels = Array.from({ length: dayCount.value }, (_, i) => `D${i + 1}`)
  trendChart.setOption({
    backgroundColor: 'transparent',
    grid: { left: 48, right: 16, top: 24, bottom: 32 },
    tooltip: { trigger: 'axis', valueFormatter: (v: number) => `¥${Number(v).toFixed(2)}` },
    xAxis: {
      type: 'category',
      data: labels,
      axisLine: { lineStyle: { color: 'rgba(255,255,255,0.15)' } },
      axisLabel: { color: '#98a2b8' },
    },
    yAxis: {
      type: 'value',
      splitLine: { lineStyle: { color: 'rgba(255,255,255,0.06)' } },
      axisLabel: { color: '#98a2b8' },
    },
    series: [
      {
        type: 'bar',
        data: dailyTotals.value,
        barMaxWidth: 28,
        itemStyle: {
          borderRadius: [6, 6, 0, 0],
          color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
            { offset: 0, color: '#6ee7ff' },
            { offset: 1, color: 'rgba(110,231,255,0.18)' },
          ]),
        },
      },
    ],
  })
}

function renderPie() {
  if (!pieChartRef.value) return
  if (!pieChart) pieChart = echarts.init(pieChartRef.value)
  const data = categoryOptions
    .map((c) => ({ name: c.label, value: spentByCategory.value[c.value] || 0, itemStyle: { color: c.color } }))
    .filter((d) => d.value > 0)

  pieChart.setOption({
    backgroundColor: 'transparent',
    tooltip: { trigger: 'item', valueFormatter: (v: number) => `¥${Number(v).toFixed(2)}` },
    legend: {
      bottom: 0,
      textStyle: { color: '#98a2b8' },
      icon: 'circle',
      itemWidth: 8,
      itemHeight: 8,
    },
    series: [
      {
        type: 'pie',
        radius: ['46%', '70%'],
        center: ['50%', '44%'],
        avoidLabelOverlap: true,
        itemStyle: { borderColor: 'rgba(12,14,24,0.9)', borderWidth: 2 },
        label: { color: '#e8ecf5', formatter: '{b}\n{d}%', fontSize: 11 },
        labelLine: { lineStyle: { color: 'rgba(255,255,255,0.2)' } },
        data: data.length > 0 ? data : [{ name: '暂无支出', value: 1, itemStyle: { color: 'rgba(255,255,255,0.08)' } }],
      },
    ],
  })
}

function renderCharts() {
  nextTick(() => {
    renderTrend()
    renderPie()
  })
}

function onResize() {
  trendChart?.resize()
  pieChart?.resize()
}

/* ---------------- 持久化 ---------------- */

function loadState() {
  try {
    const raw = localStorage.getItem(storageKey.value)
    if (!raw) {
      seedFromPlan()
      return
    }
    const parsed = JSON.parse(raw)
    if (!parsed || typeof parsed !== 'object') {
      seedFromPlan()
      return
    }
    if (Array.isArray(parsed.members) && parsed.members.length > 0) {
      members.value = parsed.members.filter((m: unknown) => typeof m === 'string' && m)
    }
    if (Array.isArray(parsed.expenses)) {
      expenses.value = parsed.expenses
        .filter((e: any) => e && typeof e.amount === 'number' && e.amount >= 0)
        .map((e: any) => ({
          id: String(e.id || genId()),
          title: String(e.title || '未命名支出'),
          category: (categoryOptions.some((c) => c.value === e.category) ? e.category : 'other') as Category,
          amount: Number(e.amount),
          dayIndex: Math.min(Math.max(Number(e.dayIndex) || 1, 1), dayCount.value),
          paidBy: String(e.paidBy || members.value[0]),
          sharedBy: Array.isArray(e.sharedBy) && e.sharedBy.length > 0 ? e.sharedBy : [...members.value],
          createdAt: Number(e.createdAt) || Date.now(),
        }))
    }
    if (form.paidBy && !members.value.includes(form.paidBy)) form.paidBy = members.value[0]
  } catch (err) {
    console.warn('[budget-dashboard] 读取本地数据失败，使用默认数据', err)
    seedFromPlan()
  }
}

function saveState() {
  try {
    localStorage.setItem(
      storageKey.value,
      JSON.stringify({ members: members.value, expenses: expenses.value }),
    )
  } catch (err) {
    console.warn('[budget-dashboard] 保存失败', err)
  }
}

/** 首次进入时用行程自身的预算明细生成几条示例支出，让看板不为空 */
function seedFromPlan() {
  const b = budgetByCategory.value
  const seeds: Expense[] = []
  const push = (title: string, category: Category, amount: number, dayIndex: number) => {
    if (amount > 0) {
      seeds.push({
        id: genId(),
        title,
        category,
        amount: Math.round(amount * 100) / 100,
        dayIndex: Math.min(Math.max(dayIndex, 1), dayCount.value),
        paidBy: members.value[0],
        sharedBy: [...members.value],
        createdAt: Date.now() - seeds.length * 1000,
      })
    }
  }
  if (b.attraction > 0) push('景点门票（行程预算）', 'attraction', b.attraction, 1)
  if (b.hotel > 0) push('住宿（行程预算）', 'hotel', b.hotel, 1)
  if (b.meal > 0) push('餐饮（行程预算）', 'meal', b.meal, 1)
  if (b.transport > 0) push('交通（行程预算）', 'transport', b.transport, 1)
  expenses.value = seeds
}

/* ---------------- 交互 ---------------- */

function genId() {
  return `${Date.now().toString(36)}-${Math.random().toString(36).slice(2, 8)}`
}

function money(v: number) {
  const n = Number(v) || 0
  return '¥' + n.toLocaleString('zh-CN', { minimumFractionDigits: 2, maximumFractionDigits: 2 })
}

function categoryLabel(c: Category) {
  return categoryOptions.find((o) => o.value === c)?.label || '其他'
}

function addExpense() {
  const amount = Number(form.amount || 0)
  if (!form.title.trim()) {
    alert('请填写支出名称')
    return
  }
  if (!(amount > 0)) {
    alert('请填写大于 0 的金额')
    return
  }
  expenses.value.unshift({
    id: genId(),
    title: form.title.trim(),
    category: form.category,
    amount,
    dayIndex: form.dayIndex,
    paidBy: form.paidBy || members.value[0],
    sharedBy: form.sharedBy.length > 0 ? [...form.sharedBy] : [...members.value],
    createdAt: Date.now(),
  })
  form.title = ''
  form.amount = null
}

function removeExpense(id: string) {
  expenses.value = expenses.value.filter((e) => e.id !== id)
}

function resetDemo() {
  expenses.value = []
  seedFromPlan()
}

/* ---------------- 生命周期 ---------------- */

onMounted(() => {
  loadState()
  renderCharts()
  window.addEventListener('resize', onResize)
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', onResize)
  trendChart?.dispose()
  pieChart?.dispose()
  trendChart = null
  pieChart = null
})

watch(
  () => [expenses.value, members.value],
  () => {
    saveState()
    renderCharts()
  },
  { deep: true },
)

watch(
  () => props.planId,
  () => {
    loadState()
    renderCharts()
  },
)

// 切换回本区域时补一次 resize，避免隐藏容器宽度为 0 导致图表错位
watch(
  () => props.tripPlan,
  () => {
    loadState()
    renderCharts()
  },
  { deep: false },
)
</script>

<style scoped>
.budget-dashboard-card {
  background: rgba(16, 18, 30, 0.72);
  border: 1px solid rgba(255, 255, 255, 0.08);
  border-radius: 18px;
  backdrop-filter: blur(18px);
  padding: 20px 22px 26px;
}

.bd-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 18px;
}

.bd-title {
  margin: 0;
  font-size: 22px;
  font-weight: 700;
  color: #f5f7fb;
  letter-spacing: 0.5px;
}

.bd-subtitle {
  margin: 6px 0 0;
  font-size: 12.5px;
  color: #98a2b8;
}

.bd-kpi-grid {
  display: grid;
  grid-template-columns: repeat(4, minmax(0, 1fr));
  gap: 12px;
}

.bd-kpi {
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding: 14px 16px;
  border-radius: 14px;
  background: rgba(255, 255, 255, 0.04);
  border: 1px solid rgba(255, 255, 255, 0.06);
}

.bd-kpi-label {
  font-size: 12px;
  color: #98a2b8;
}

.bd-kpi-value {
  font-size: 20px;
  font-weight: 700;
  color: #f5f7fb;
  font-variant-numeric: tabular-nums;
}

.bd-kpi-value.is-good {
  color: #7ee787;
}

.bd-kpi-value.is-bad {
  color: #ff8080;
}

.bd-progress-wrap {
  margin: 16px 0 22px;
}

.bd-progress-track {
  height: 10px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.08);
  overflow: hidden;
}

.bd-progress-bar {
  height: 100%;
  border-radius: 999px;
  background: linear-gradient(90deg, #6ee7ff, #7ee787);
  transition: width 0.45s ease;
}

.bd-progress-bar.is-over {
  background: linear-gradient(90deg, #ff9f6b, #ff6b6b);
}

.bd-progress-meta {
  display: flex;
  justify-content: space-between;
  margin-top: 8px;
  font-size: 12px;
  color: #98a2b8;
}

.bd-main-grid {
  display: grid;
  grid-template-columns: minmax(0, 1.15fr) minmax(0, 1fr);
  gap: 16px;
}

.bd-col {
  display: flex;
  flex-direction: column;
  gap: 16px;
  min-width: 0;
}

.bd-panel {
  padding: 16px 18px;
  border-radius: 16px;
  background: rgba(255, 255, 255, 0.035);
  border: 1px solid rgba(255, 255, 255, 0.06);
}

.bd-panel-form {
  margin-top: 16px;
}

.bd-panel-head {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  margin-bottom: 12px;
}

.bd-panel-head h3 {
  margin: 0;
  font-size: 15px;
  font-weight: 600;
  color: #e8ecf5;
}

.bd-panel-hint {
  font-size: 12px;
  color: #7c879c;
}

.bd-cat-list {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.bd-cat-head {
  display: flex;
  justify-content: space-between;
  font-size: 13px;
  color: #cbd5e6;
  margin-bottom: 6px;
}

.bd-cat-name {
  display: inline-flex;
  align-items: center;
  gap: 8px;
}

.bd-dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  display: inline-block;
}

.bd-cat-num {
  font-variant-numeric: tabular-nums;
  color: #98a2b8;
}

.bd-cat-num.is-bad {
  color: #ff8080;
}

.bd-cat-track {
  position: relative;
  height: 8px;
  border-radius: 999px;
  background: rgba(255, 255, 255, 0.06);
  overflow: hidden;
}

.bd-cat-budget,
.bd-cat-spent {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  border-radius: 999px;
}

.bd-cat-foot {
  display: flex;
  justify-content: space-between;
  margin-top: 5px;
  font-size: 11.5px;
  color: #7c879c;
}

.bd-chart {
  width: 100%;
  height: 220px;
}

.bd-chart-pie {
  height: 240px;
}

.bd-member-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.bd-member-row {
  display: grid;
  grid-template-columns: 1fr auto auto auto;
  gap: 10px;
  align-items: center;
  padding: 9px 12px;
  border-radius: 10px;
  background: rgba(255, 255, 255, 0.03);
  font-size: 12.5px;
  color: #cbd5e6;
}

.bd-member-name {
  font-weight: 600;
  color: #e8ecf5;
}

.bd-member-cell {
  color: #98a2b8;
  font-variant-numeric: tabular-nums;
}

.bd-member-net {
  font-weight: 600;
  font-variant-numeric: tabular-nums;
}

.bd-member-net.is-good {
  color: #7ee787;
}

.bd-member-net.is-bad {
  color: #ff8080;
}

.bd-settle-list {
  margin-top: 14px;
  padding-top: 12px;
  border-top: 1px dashed rgba(255, 255, 255, 0.1);
}

.bd-settle-title {
  font-size: 12px;
  color: #7c879c;
  margin-bottom: 8px;
}

.bd-settle-row {
  display: grid;
  grid-template-columns: 1fr auto 1fr auto;
  gap: 8px;
  align-items: center;
  padding: 7px 10px;
  border-radius: 8px;
  background: rgba(110, 231, 255, 0.06);
  font-size: 12.5px;
  color: #cbd5e6;
  margin-bottom: 6px;
}

.bd-settle-arrow {
  color: #6ee7ff;
}

.bd-settle-amount {
  font-weight: 600;
  color: #6ee7ff;
  font-variant-numeric: tabular-nums;
}

.bd-form-grid {
  display: grid;
  grid-template-columns: 1.4fr 1fr 1fr 1fr 1fr auto;
  gap: 10px;
}

.bd-form-item {
  min-width: 0;
}

.bd-submit {
  white-space: nowrap;
}

.bd-share-row {
  display: flex;
  align-items: center;
  gap: 14px;
  margin-top: 12px;
  flex-wrap: wrap;
}

.bd-share-label {
  font-size: 12.5px;
  color: #98a2b8;
}

.bd-expense-list {
  margin-top: 16px;
  border-top: 1px dashed rgba(255, 255, 255, 0.1);
  padding-top: 12px;
}

.bd-expense-head,
.bd-expense-row {
  display: grid;
  grid-template-columns: 1.6fr 0.8fr 0.7fr 0.8fr 0.9fr 0.6fr;
  gap: 8px;
  align-items: center;
  font-size: 12.5px;
  padding: 8px 6px;
}

.bd-expense-head {
  color: #7c879c;
  border-bottom: 1px solid rgba(255, 255, 255, 0.06);
}

.bd-expense-row {
  color: #cbd5e6;
  border-bottom: 1px solid rgba(255, 255, 255, 0.04);
}

.bd-expense-title {
  color: #e8ecf5;
}

.bd-expense-amount {
  font-weight: 600;
  font-variant-numeric: tabular-nums;
  color: #ffd5c6;
}

.bd-expense-del {
  color: #ff8080;
  cursor: pointer;
}

.bd-expense-del:hover {
  text-decoration: underline;
}

@media (max-width: 1200px) {
  .bd-main-grid {
    grid-template-columns: minmax(0, 1fr);
  }
  .bd-form-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
  .bd-kpi-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}
</style>

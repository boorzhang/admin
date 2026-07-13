<script setup lang="ts">
import { computed, onMounted, reactive, ref } from 'vue'
import { useI18n } from 'vue-i18n'
import { adminAPI } from '@/api/admin'
import { Button } from '@/components/ui/button'
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card'
import { Input } from '@/components/ui/input'
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select'
import { formatMoney, getLocalizedText } from '@/utils/format'
import { formatSkuDisplayLabel } from '@/utils/sku'
import type { AdminDashboardInventoryAlert } from '@/api/types'
import DashboardAd from '@/components/admin/DashboardAd.vue'

interface DashboardAlertItem {
  type: string
  level: string
  value: number
}

interface DashboardFunnel {
  orders_created: number
  payments_created: number
  payments_success: number
  orders_paid: number
  orders_completed: number
  payment_conversion_rate: string
  completion_rate: string
}

interface DashboardOverview {
  range: string
  from: string
  to: string
  timezone: string
  currency?: string
  kpi: {
    orders_total: number
    paid_orders: number
    completed_orders: number
    pending_payment_orders: number
    processing_orders: number
    gmv_paid: string
    total_cost: string
    total_profit: string
    profit_margin: string
    payments_total: number
    payments_success: number
    payments_failed: number
    payment_success_rate: string
    new_users: number
    active_products: number
    out_of_stock_products: number
    low_stock_products: number
    out_of_stock_skus: number
    low_stock_skus: number
    auto_available_secrets: number
    manual_available_units: number
    total_user_balance: string
  }
  funnel: DashboardFunnel
  alerts: DashboardAlertItem[]
}

interface DashboardTrendPoint {
  date: string
  orders_total: number
  orders_paid: number
  payments_success: number
  payments_failed: number
  gmv_paid: string
  profit: string
}

interface DashboardTrends {
  range: string
  from: string
  to: string
  timezone: string
  points: DashboardTrendPoint[]
}

interface DashboardProductRanking {
  product_id: number
  sku_id?: number
  sku_code?: string
  sku_spec_values?: Record<string, unknown>
  title: string
  paid_orders: number
  quantity: number
  paid_amount: string
  total_cost: string
  profit: string
}

interface DashboardChannelRanking {
  channel_id: number
  channel_name: string
  provider_type: string
  channel_type: string
  success_count: number
  failed_count: number
  success_amount: string
  success_rate: string
}

interface DashboardRankings {
  range: string
  from: string
  to: string
  timezone: string
  top_products: DashboardProductRanking[]
  top_channels: DashboardChannelRanking[]
}

const { t, locale } = useI18n()

const rankingSkuLabel = (item: DashboardProductRanking) =>
  formatSkuDisplayLabel({ sku_code: item.sku_code, spec_values: item.sku_spec_values }, locale.value)

const loadingOverview = ref(false)
const loadingTrends = ref(false)
const loadingRankings = ref(false)
const loadingInventoryAlerts = ref(false)
const loadingDimensions = ref(false)
const dashboardError = ref('')
const overview = ref<DashboardOverview | null>(null)
const trends = ref<DashboardTrends | null>(null)
const rankings = ref<DashboardRankings | null>(null)
const inventoryAlerts = ref<AdminDashboardInventoryAlert[]>([])

interface DashboardDimensionsBucket {
  key: string
  total: number
  paid: number
}
interface DashboardDimensionsMerchant {
  merchant_id: number | null
  slug: string
  name: string
  total: number
  paid: number
  paid_amount: number | string
}
interface DashboardDimensionsContact {
  contact: string
  merchant_slug: string
  order_count: number
  device_os: string
  browser: string
  paid_amount?: number | string | null
  paid_currency?: string
  last_paid_at?: string | null
  last_created_at?: string | null
}
interface DashboardDimensions {
  hours: number
  since: string
  by_merchant: DashboardDimensionsMerchant[]
  by_os: DashboardDimensionsBucket[]
  by_browser: DashboardDimensionsBucket[]
  recent_contacts: DashboardDimensionsContact[]
}
const dimensions = ref<DashboardDimensions | null>(null)
const dimensionHours = ref<number>(24)

const filters = reactive({
  range: '7d',
  from: '',
  to: '',
})

const rangeOptions = computed(() => [
  { value: 'today', label: t('admin.dashboard.range.today') },
  { value: '7d', label: t('admin.dashboard.range.last7Days') },
  { value: '30d', label: t('admin.dashboard.range.last30Days') },
  { value: 'custom', label: t('admin.dashboard.range.custom') },
])

const isCustomRange = computed(() => filters.range === 'custom')

const trendPoints = computed(() => trends.value?.points || [])

const maxOrderTrend = computed(() => {
  let maxValue = 1
  trendPoints.value.forEach((point) => {
    maxValue = Math.max(maxValue, point.orders_total, point.orders_paid)
  })
  return maxValue
})

const maxPaymentTrend = computed(() => {
  let maxValue = 1
  trendPoints.value.forEach((point) => {
    maxValue = Math.max(maxValue, point.payments_success, point.payments_failed)
  })
  return maxValue
})

const funnelSteps = computed(() => {
  const funnel = overview.value?.funnel
  if (!funnel) return []
  return [
    { key: 'ordersCreated', label: t('admin.dashboard.funnel.ordersCreated'), value: funnel.orders_created },
    { key: 'paymentsCreated', label: t('admin.dashboard.funnel.paymentsCreated'), value: funnel.payments_created },
    { key: 'paymentsSuccess', label: t('admin.dashboard.funnel.paymentsSuccess'), value: funnel.payments_success },
    { key: 'ordersPaid', label: t('admin.dashboard.funnel.ordersPaid'), value: funnel.orders_paid },
    { key: 'ordersCompleted', label: t('admin.dashboard.funnel.ordersCompleted'), value: funnel.orders_completed },
  ]
})

const maxFunnelValue = computed(() => {
  let maxValue = 1
  funnelSteps.value.forEach((item) => {
    maxValue = Math.max(maxValue, item.value)
  })
  return maxValue
})

const orderTotalHeight = (value: number) => `${Math.max(4, Math.round((value / maxOrderTrend.value) * 100))}%`
const orderPaidHeight = (value: number) => `${Math.max(4, Math.round((value / maxOrderTrend.value) * 100))}%`
const paymentSuccessHeight = (value: number) => `${Math.max(4, Math.round((value / maxPaymentTrend.value) * 100))}%`
const paymentFailedHeight = (value: number) => `${Math.max(4, Math.round((value / maxPaymentTrend.value) * 100))}%`
const funnelWidth = (value: number) => `${Math.max(4, Math.round((value / maxFunnelValue.value) * 100))}%`

const shortDate = (value?: string) => {
  if (!value) return '-'
  const date = new Date(value)
  if (Number.isNaN(date.getTime())) return value
  return date.toLocaleDateString(undefined, { month: '2-digit', day: '2-digit' })
}

const makeRangeDate = (raw: string, endOfDay: boolean) => {
  if (!raw) return undefined
  const suffix = endOfDay ? 'T23:59:59' : 'T00:00:00'
  const date = new Date(`${raw}${suffix}`)
  if (Number.isNaN(date.getTime())) return undefined
  return date.toISOString()
}

const buildQuery = (forceRefresh = false) => {
  const params: Record<string, any> = {
    range: filters.range,
    tz: Intl.DateTimeFormat().resolvedOptions().timeZone,
  }

  if (isCustomRange.value) {
    const from = makeRangeDate(filters.from, false)
    const to = makeRangeDate(filters.to, true)
    if (!from || !to) {
      return null
    }
    params.from = from
    params.to = to
  }

  if (forceRefresh) {
    params.force_refresh = true
  }

  return params
}

const loadOverview = async (forceRefresh = false) => {
  loadingOverview.value = true
  try {
    const params = buildQuery(forceRefresh)
    if (!params) {
      dashboardError.value = t('admin.dashboard.errors.customRangeRequired')
      overview.value = null
      return
    }
    const response = await adminAPI.getDashboardOverview(params)
    overview.value = response.data.data as unknown as DashboardOverview
  } finally {
    loadingOverview.value = false
  }
}

const loadTrends = async (forceRefresh = false) => {
  loadingTrends.value = true
  try {
    const params = buildQuery(forceRefresh)
    if (!params) {
      trends.value = null
      return
    }
    const response = await adminAPI.getDashboardTrends(params)
    trends.value = response.data.data as unknown as DashboardTrends
  } finally {
    loadingTrends.value = false
  }
}

const loadRankings = async (forceRefresh = false) => {
  loadingRankings.value = true
  try {
    const params = buildQuery(forceRefresh)
    if (!params) {
      rankings.value = null
      return
    }
    const response = await adminAPI.getDashboardRankings(params)
    rankings.value = response.data.data as unknown as DashboardRankings
  } finally {
    loadingRankings.value = false
  }
}

const loadInventoryAlerts = async () => {
  loadingInventoryAlerts.value = true
  try {
    const response = await adminAPI.getDashboardInventoryAlerts()
    inventoryAlerts.value = (response.data.data as unknown as AdminDashboardInventoryAlert[]) || []
  } catch {
    inventoryAlerts.value = []
  } finally {
    loadingInventoryAlerts.value = false
  }
}

const loadDimensions = async () => {
  loadingDimensions.value = true
  try {
    const response = await adminAPI.getDashboardDimensions({ hours: dimensionHours.value })
    dimensions.value = response.data.data as unknown as DashboardDimensions
  } catch {
    dimensions.value = null
  } finally {
    loadingDimensions.value = false
  }
}

const handleDimensionHoursChange = (value: unknown) => {
  const next = Number(value)
  if (!Number.isFinite(next) || next <= 0) return
  dimensionHours.value = next
  loadDimensions()
}

const formatLastSeen = (iso?: string | null) => {
  if (!iso) return '-'
  const d = new Date(iso)
  if (Number.isNaN(d.getTime())) return iso
  return d.toLocaleString(undefined, { month: '2-digit', day: '2-digit', hour: '2-digit', minute: '2-digit' })
}

const conversionPct = (paid: number, total: number) => {
  if (!total) return '0.0'
  return ((paid * 100) / total).toFixed(1)
}

const loadDashboard = async (forceRefresh = false) => {
  dashboardError.value = ''
  try {
    await Promise.all([loadOverview(forceRefresh), loadTrends(forceRefresh), loadRankings(forceRefresh), loadInventoryAlerts(), loadDimensions()])
  } catch (error: any) {
    dashboardError.value = error?.message || t('admin.dashboard.errors.fetchFailed')
  }
}

const handleRangeChange = (value: unknown) => {
  const nextValue = String(value || '').trim() || '7d'
  filters.range = nextValue
  if (nextValue !== 'custom') {
    filters.from = ''
    filters.to = ''
    loadDashboard()
    return
  }

  const today = new Date()
  const start = new Date(today)
  start.setDate(start.getDate() - 6)
  filters.from = start.toISOString().slice(0, 10)
  filters.to = today.toISOString().slice(0, 10)
  loadDashboard()
}

const handleCustomRangeChange = () => {
  if (!isCustomRange.value) return
  loadDashboard()
}

const refreshDashboard = () => {
  loadDashboard(true)
}

const alertClass = (level: string) => {
  if (level === 'error') return 'border-rose-500/30 bg-rose-500/10 text-rose-700 dark:text-rose-300'
  if (level === 'warning') return 'border-amber-500/30 bg-amber-500/10 text-amber-700 dark:text-amber-300'
  return 'border-border bg-muted/40 text-foreground'
}

const alertLabel = (type: string) => {
  const key = `admin.dashboard.alertTypes.${type}`
  const translated = t(key)
  return translated === key ? type : translated
}

const inventoryAlertLabel = (item: AdminDashboardInventoryAlert) => {
  return item.alert_type === 'out_of_stock_products'
    ? t('admin.dashboard.inventoryAlerts.outOfStock')
    : t('admin.dashboard.inventoryAlerts.lowStock')
}

const inventoryAlertBadgeClass = (item: AdminDashboardInventoryAlert) => {
  return item.alert_type === 'out_of_stock_products'
    ? 'bg-rose-500/10 text-rose-700 dark:text-rose-300'
    : 'bg-amber-500/10 text-amber-700 dark:text-amber-300'
}

const skuSpecLabel = (item: AdminDashboardInventoryAlert) => {
  if (!item.sku_spec_values || Object.keys(item.sku_spec_values).length === 0) return ''
  return Object.values(item.sku_spec_values).join(' / ')
}

const quickActions = computed(() => [
  { label: t('admin.nav.orders'), path: '/orders' },
  { label: t('admin.nav.payments'), path: '/payments' },
  { label: t('admin.nav.products'), path: '/products' },
  { label: t('admin.nav.cardSecrets'), path: '/card-secrets' },
  { label: t('admin.nav.users'), path: '/users' },
])

const channelLabel = (item: DashboardChannelRanking) => {
  if (item.channel_name) return item.channel_name
  return `${item.provider_type || '-'} / ${item.channel_type || '-'}`
}

onMounted(() => {
  loadDashboard()
})
</script>

<template>
  <div class="space-y-6">
    <div class="flex flex-col gap-3 xl:flex-row xl:items-start xl:justify-between">
      <div>
        <h1 class="text-2xl font-semibold tracking-tight">{{ t('admin.dashboard.title') }}</h1>
        <p class="mt-1 text-sm text-muted-foreground">{{ t('admin.dashboard.subtitle') }}</p>
      </div>
      <div class="flex flex-col gap-2 sm:flex-row sm:flex-wrap sm:items-center">
        <div class="w-full sm:w-[150px]">
          <Select v-model="filters.range" @update:modelValue="handleRangeChange">
            <SelectTrigger class="h-9">
              <SelectValue :placeholder="t('admin.dashboard.filters.range')" />
            </SelectTrigger>
            <SelectContent>
              <SelectItem v-for="item in rangeOptions" :key="item.value" :value="item.value">
                {{ item.label }}
              </SelectItem>
            </SelectContent>
          </Select>
        </div>
        <Input
          v-if="isCustomRange"
          v-model="filters.from"
          type="date"
          class="h-9 w-full sm:w-[150px]"
          :placeholder="t('admin.dashboard.filters.from')"
          @update:modelValue="handleCustomRangeChange"
        />
        <Input
          v-if="isCustomRange"
          v-model="filters.to"
          type="date"
          class="h-9 w-full sm:w-[150px]"
          :placeholder="t('admin.dashboard.filters.to')"
          @update:modelValue="handleCustomRangeChange"
        />
        <Button size="sm" variant="outline" class="h-9 w-full sm:w-auto" :disabled="loadingOverview || loadingTrends || loadingRankings" @click="refreshDashboard">
          {{ t('admin.dashboard.actions.refreshNow') }}
        </Button>
      </div>
    </div>

    <div v-if="dashboardError" class="rounded-xl border border-destructive/40 bg-destructive/10 px-4 py-3 text-sm text-destructive">
      {{ dashboardError }}
    </div>

    <div class="min-w-0">
      <DashboardAd slot-code="dashboard_top_banner" layout="banner" />
    </div>

    <div class="grid gap-4 sm:grid-cols-2 xl:grid-cols-4 [&>*]:min-w-0">
      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-xs font-medium text-muted-foreground">{{ t('admin.dashboard.kpi.ordersTotal') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div class="text-2xl font-semibold">{{ overview?.kpi.orders_total ?? 0 }}</div>
          <div class="mt-1 text-xs text-muted-foreground">{{ t('admin.dashboard.kpi.paidOrders') }}: {{ overview?.kpi.paid_orders ?? 0 }}</div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-xs font-medium text-muted-foreground">{{ t('admin.dashboard.kpi.gmvPaid') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div class="text-2xl font-semibold">{{ formatMoney(overview?.kpi.gmv_paid, overview?.currency) }}</div>
          <div class="mt-1 text-xs text-muted-foreground">{{ t('admin.dashboard.kpi.paymentSuccessRate') }}: {{ overview?.kpi.payment_success_rate ?? '0.00' }}%</div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-xs font-medium text-muted-foreground">{{ t('admin.dashboard.kpi.totalCost') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div class="text-2xl font-semibold">{{ formatMoney(overview?.kpi.total_cost, overview?.currency) }}</div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-xs font-medium text-muted-foreground">{{ t('admin.dashboard.kpi.totalProfit') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div class="text-2xl font-semibold">{{ formatMoney(overview?.kpi.total_profit, overview?.currency) }}</div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-xs font-medium text-muted-foreground">{{ t('admin.dashboard.kpi.profitMargin') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div class="text-2xl font-semibold">{{ overview?.kpi.profit_margin ?? '0.00' }}%</div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-xs font-medium text-muted-foreground">{{ t('admin.dashboard.kpi.pendingOrders') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div class="text-2xl font-semibold">{{ overview?.kpi.pending_payment_orders ?? 0 }}</div>
          <div class="mt-1 text-xs text-muted-foreground">{{ t('admin.dashboard.kpi.processingOrders') }}: {{ overview?.kpi.processing_orders ?? 0 }}</div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-xs font-medium text-muted-foreground">{{ t('admin.dashboard.kpi.newUsers') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div class="text-2xl font-semibold">{{ overview?.kpi.new_users ?? 0 }}</div>
          <div class="mt-1 text-xs text-muted-foreground">{{ t('admin.dashboard.kpi.activeProducts') }}: {{ overview?.kpi.active_products ?? 0 }}</div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-xs font-medium text-muted-foreground">{{ t('admin.dashboard.kpi.totalUserBalance') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div class="text-2xl font-semibold">{{ formatMoney(overview?.kpi.total_user_balance, overview?.currency) }}</div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-xs font-medium text-muted-foreground">{{ t('admin.dashboard.kpi.lowStockProducts') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div class="text-2xl font-semibold">{{ overview?.kpi.low_stock_products ?? 0 }}</div>
          <div class="mt-1 text-xs text-muted-foreground">{{ t('admin.dashboard.kpi.outOfStockProducts') }}: {{ overview?.kpi.out_of_stock_products ?? 0 }}</div>
          <div class="mt-1 text-xs text-muted-foreground">{{ t('admin.dashboard.kpi.outOfStockSKUs') }}: {{ overview?.kpi.out_of_stock_skus ?? 0 }} / {{ t('admin.dashboard.kpi.lowStockSKUs') }}: {{ overview?.kpi.low_stock_skus ?? 0 }}</div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-xs font-medium text-muted-foreground">{{ t('admin.dashboard.kpi.autoAvailableSecrets') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div class="text-2xl font-semibold">{{ overview?.kpi.auto_available_secrets ?? 0 }}</div>
          <div class="mt-1 text-xs text-muted-foreground">{{ t('admin.dashboard.kpi.manualAvailableUnits') }}: {{ overview?.kpi.manual_available_units ?? 0 }}</div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-xs font-medium text-muted-foreground">{{ t('admin.dashboard.kpi.paymentsSuccess') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div class="text-2xl font-semibold">{{ overview?.kpi.payments_success ?? 0 }}</div>
          <div class="mt-1 text-xs text-muted-foreground">{{ t('admin.dashboard.kpi.paymentsFailed') }}: {{ overview?.kpi.payments_failed ?? 0 }}</div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-xs font-medium text-muted-foreground">{{ t('admin.dashboard.period') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div class="text-sm font-medium">{{ shortDate(overview?.from) }} - {{ shortDate(overview?.to) }}</div>
          <div class="mt-1 text-xs text-muted-foreground">{{ overview?.timezone || '-' }}</div>
        </CardContent>
      </Card>

      <div class="min-w-0">
        <DashboardAd slot-code="dashboard_kpi_card" layout="card" />
      </div>
    </div>

    <div class="grid gap-4 xl:grid-cols-2 [&>*]:min-w-0">
      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-sm">{{ t('admin.dashboard.trends.orderTitle') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div v-if="loadingTrends" class="text-sm text-muted-foreground">{{ t('admin.common.loading') }}</div>
          <div v-else-if="trendPoints.length === 0" class="text-sm text-muted-foreground">{{ t('admin.dashboard.emptyTrend') }}</div>
          <div v-else class="space-y-3">
            <div class="flex flex-wrap items-center gap-3 text-xs text-muted-foreground">
              <span class="inline-flex items-center gap-1"><span class="h-2 w-2 rounded-full bg-primary"></span>{{ t('admin.dashboard.trends.ordersTotal') }}</span>
              <span class="inline-flex items-center gap-1"><span class="h-2 w-2 rounded-full bg-emerald-500"></span>{{ t('admin.dashboard.trends.ordersPaid') }}</span>
            </div>
            <div class="overflow-x-auto">
              <div class="inline-flex items-end gap-2 rounded-lg bg-muted/20 px-2 py-3">
                <div v-for="point in trendPoints" :key="point.date" class="flex shrink-0 flex-col items-center gap-1">
                  <div class="flex h-32 items-end gap-0.5">
                    <div class="w-2 rounded-t bg-primary/80" :style="{ height: orderTotalHeight(point.orders_total) }" :title="`${t('admin.dashboard.trends.ordersTotal')}: ${point.orders_total}`"></div>
                    <div class="w-2 rounded-t bg-emerald-500/80" :style="{ height: orderPaidHeight(point.orders_paid) }" :title="`${t('admin.dashboard.trends.ordersPaid')}: ${point.orders_paid}`"></div>
                  </div>
                  <div class="whitespace-nowrap text-[10px] text-muted-foreground">{{ shortDate(point.date) }}</div>
                  <div class="whitespace-nowrap text-[10px] font-medium text-emerald-600 dark:text-emerald-400">{{ formatMoney(point.profit, overview?.currency) }}</div>
                </div>
              </div>
            </div>
          </div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-sm">{{ t('admin.dashboard.trends.paymentTitle') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div v-if="loadingTrends" class="text-sm text-muted-foreground">{{ t('admin.common.loading') }}</div>
          <div v-else-if="trendPoints.length === 0" class="text-sm text-muted-foreground">{{ t('admin.dashboard.emptyTrend') }}</div>
          <div v-else class="space-y-3">
            <div class="flex flex-wrap items-center gap-3 text-xs text-muted-foreground">
              <span class="inline-flex items-center gap-1"><span class="h-2 w-2 rounded-full bg-sky-500"></span>{{ t('admin.dashboard.trends.paymentsSuccess') }}</span>
              <span class="inline-flex items-center gap-1"><span class="h-2 w-2 rounded-full bg-rose-500"></span>{{ t('admin.dashboard.trends.paymentsFailed') }}</span>
            </div>
            <div class="overflow-x-auto">
              <div class="inline-flex items-end gap-2 rounded-lg bg-muted/20 px-2 py-3">
                <div v-for="point in trendPoints" :key="`${point.date}-payment`" class="flex shrink-0 flex-col items-center gap-1">
                  <div class="flex h-32 items-end gap-0.5">
                    <div class="w-2 rounded-t bg-sky-500/80" :style="{ height: paymentSuccessHeight(point.payments_success) }" :title="`${t('admin.dashboard.trends.paymentsSuccess')}: ${point.payments_success}`"></div>
                    <div class="w-2 rounded-t bg-rose-500/80" :style="{ height: paymentFailedHeight(point.payments_failed) }" :title="`${t('admin.dashboard.trends.paymentsFailed')}: ${point.payments_failed}`"></div>
                  </div>
                  <div class="whitespace-nowrap text-[10px] text-muted-foreground">{{ shortDate(point.date) }}</div>
                </div>
              </div>
            </div>
          </div>
        </CardContent>
      </Card>
    </div>

    <div class="grid gap-4 xl:grid-cols-3 [&>*]:min-w-0">
      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-sm">{{ t('admin.dashboard.funnel.title') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div v-if="!overview" class="text-sm text-muted-foreground">{{ t('admin.common.loading') }}</div>
          <div v-else class="space-y-3">
            <div v-for="item in funnelSteps" :key="item.key" class="space-y-1">
              <div class="flex items-center justify-between text-xs text-muted-foreground">
                <span>{{ item.label }}</span>
                <span class="font-mono text-foreground">{{ item.value }}</span>
              </div>
              <div class="h-2 rounded-full bg-muted">
                <div class="h-2 rounded-full bg-primary/70" :style="{ width: funnelWidth(item.value) }"></div>
              </div>
            </div>
            <div class="grid grid-cols-2 gap-2 border-t border-border pt-3 text-xs">
              <div class="rounded-md border border-border bg-muted/30 px-2 py-1">
                <div class="text-muted-foreground">{{ t('admin.dashboard.funnel.paymentConversionRate') }}</div>
                <div class="font-semibold">{{ overview.funnel.payment_conversion_rate }}%</div>
              </div>
              <div class="rounded-md border border-border bg-muted/30 px-2 py-1">
                <div class="text-muted-foreground">{{ t('admin.dashboard.funnel.completionRate') }}</div>
                <div class="font-semibold">{{ overview.funnel.completion_rate }}%</div>
              </div>
            </div>
          </div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-sm">{{ t('admin.dashboard.rankings.topProductsTitle') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div v-if="loadingRankings" class="text-sm text-muted-foreground">{{ t('admin.common.loading') }}</div>
          <div v-else-if="!rankings || rankings.top_products.length === 0" class="text-sm text-muted-foreground">{{ t('admin.dashboard.rankings.empty') }}</div>
          <div v-else class="space-y-2">
            <div v-for="item in rankings.top_products" :key="`${item.product_id}-${item.sku_id || 0}`" class="min-w-0 rounded-lg border border-border px-3 py-2">
              <div class="line-clamp-1 min-w-0 text-sm font-medium">{{ item.title }}</div>
              <div v-if="rankingSkuLabel(item)" class="mt-0.5 line-clamp-1 text-xs text-muted-foreground">
                {{ t('admin.dashboard.rankings.skuLabel') }}: {{ rankingSkuLabel(item) }}
              </div>
              <div class="mt-1 flex flex-col gap-1 text-xs text-muted-foreground sm:flex-row sm:items-center sm:justify-between">
                <span>{{ t('admin.dashboard.rankings.paidOrders') }}: {{ item.paid_orders }}</span>
                <span>{{ t('admin.dashboard.rankings.quantity') }}: {{ item.quantity }}</span>
              </div>
              <div class="mt-1 flex flex-col gap-1 text-xs font-semibold text-foreground sm:flex-row sm:items-center sm:justify-between">
                <span>{{ t('admin.dashboard.rankings.paidAmount') }}: {{ formatMoney(item.paid_amount, overview?.currency) }}</span>
                <span>{{ t('admin.dashboard.rankings.profit') }}: {{ formatMoney(item.profit, overview?.currency) }}</span>
              </div>
            </div>
          </div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-sm">{{ t('admin.dashboard.rankings.topChannelsTitle') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div v-if="loadingRankings" class="text-sm text-muted-foreground">{{ t('admin.common.loading') }}</div>
          <div v-else-if="!rankings || rankings.top_channels.length === 0" class="text-sm text-muted-foreground">{{ t('admin.dashboard.rankings.empty') }}</div>
          <div v-else class="space-y-2">
            <div v-for="item in rankings.top_channels" :key="`${item.channel_id}-${item.channel_type}`" class="min-w-0 rounded-lg border border-border px-3 py-2">
              <div class="line-clamp-1 min-w-0 text-sm font-medium">{{ channelLabel(item) }}</div>
              <div class="mt-1 flex flex-col gap-1 text-xs text-muted-foreground sm:flex-row sm:items-center sm:justify-between">
                <span>{{ t('admin.dashboard.rankings.successCount') }}: {{ item.success_count }}</span>
                <span>{{ t('admin.dashboard.rankings.failedCount') }}: {{ item.failed_count }}</span>
              </div>
              <div class="mt-1 flex flex-col gap-1 text-xs sm:flex-row sm:items-center sm:justify-between">
                <span class="font-semibold text-foreground">{{ t('admin.dashboard.rankings.successAmount') }}: {{ formatMoney(item.success_amount, overview?.currency) }}</span>
                <span class="text-muted-foreground">{{ t('admin.dashboard.rankings.successRate') }}: {{ item.success_rate }}%</span>
              </div>
            </div>
          </div>
        </CardContent>
      </Card>
    </div>

    <div class="min-w-0">
      <DashboardAd slot-code="dashboard_sponsored" layout="compact" />
    </div>

    <div class="grid gap-4 xl:grid-cols-2 [&>*]:min-w-0">
      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-sm">{{ t('admin.dashboard.alerts.title') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <!-- 通用告警（待支付订单、失败支付等） -->
          <div v-if="overview && overview.alerts.filter(a => a.type !== 'out_of_stock_products' && a.type !== 'low_stock_products').length > 0" class="space-y-2 mb-3">
            <div
              v-for="alert in overview.alerts.filter(a => a.type !== 'out_of_stock_products' && a.type !== 'low_stock_products')"
              :key="`${alert.type}-${alert.value}`"
              class="rounded-lg border px-3 py-2 text-sm"
              :class="alertClass(alert.level)"
            >
              <div class="flex items-center justify-between gap-2">
                <span class="font-medium">{{ alertLabel(alert.type) }}</span>
                <span class="font-mono text-xs">{{ alert.value }}</span>
              </div>
            </div>
          </div>
          <!-- SKU 级别库存告警 -->
          <div v-if="inventoryAlerts.length > 0" class="space-y-2">
            <div class="text-xs font-medium text-muted-foreground mb-1">{{ t('admin.dashboard.inventoryAlerts.title') }}</div>
            <div
              v-for="(item, idx) in inventoryAlerts"
              :key="`inv-${item.product_id}-${item.sku_id || 0}-${idx}`"
              class="rounded-lg border border-border px-3 py-2 text-sm"
            >
              <div class="flex items-center justify-between gap-2">
                <div class="min-w-0 flex-1">
                  <router-link :to="{ path: '/products', query: { product_id: item.product_id } }" class="font-medium text-primary underline-offset-4 hover:underline break-words">
                    {{ getLocalizedText(item.product_title) }}
                  </router-link>
                  <div v-if="skuSpecLabel(item)" class="text-xs text-muted-foreground mt-0.5">
                    SKU: {{ skuSpecLabel(item) }}
                    <span v-if="item.sku_code" class="ml-1">({{ item.sku_code }})</span>
                  </div>
                </div>
                <div class="flex shrink-0 items-center gap-2">
                  <span class="font-mono text-xs text-muted-foreground">{{ item.available_stock }}</span>
                  <span class="inline-flex rounded-full px-2 py-0.5 text-[10px] font-medium" :class="inventoryAlertBadgeClass(item)">
                    {{ inventoryAlertLabel(item) }}
                  </span>
                </div>
              </div>
            </div>
          </div>
          <div v-if="(!overview || overview.alerts.length === 0) && inventoryAlerts.length === 0" class="text-sm text-muted-foreground">
            {{ t('admin.dashboard.alerts.empty') }}
          </div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-sm">{{ t('admin.dashboard.quickActions.title') }}</CardTitle>
        </CardHeader>
        <CardContent>
          <div class="grid grid-cols-2 gap-2 md:grid-cols-3">
            <Button
              v-for="action in quickActions"
              :key="action.path"
              variant="outline"
              size="sm"
              class="justify-start"
              as-child
            >
              <router-link :to="action.path">{{ action.label }}</router-link>
            </Button>
          </div>
        </CardContent>
      </Card>
    </div>

    <!-- 实时维度面板（按商户/OS/浏览器分布 + 最近活跃账户）。时间窗独立，默认 24h。 -->
    <div class="flex flex-col gap-2 pt-2 sm:flex-row sm:items-center sm:justify-between">
      <h2 class="text-base font-semibold">实时维度（订单按商户 / OS / 浏览器分布）</h2>
      <div class="w-[140px]">
        <Select :model-value="String(dimensionHours)" @update:modelValue="handleDimensionHoursChange">
          <SelectTrigger class="h-9">
            <SelectValue placeholder="时间窗" />
          </SelectTrigger>
          <SelectContent>
            <SelectItem value="1">近 1 小时</SelectItem>
            <SelectItem value="6">近 6 小时</SelectItem>
            <SelectItem value="12">近 12 小时</SelectItem>
            <SelectItem value="24">近 24 小时</SelectItem>
            <SelectItem value="72">近 3 天</SelectItem>
            <SelectItem value="168">近 7 天</SelectItem>
          </SelectContent>
        </Select>
      </div>
    </div>

    <div class="grid gap-4 xl:grid-cols-3 [&>*]:min-w-0">
      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-sm">按商户 / 域名分布</CardTitle>
        </CardHeader>
        <CardContent>
          <div v-if="loadingDimensions" class="text-sm text-muted-foreground">{{ t('admin.common.loading') }}</div>
          <div v-else-if="!dimensions || dimensions.by_merchant.length === 0" class="text-sm text-muted-foreground">暂无订单</div>
          <div v-else class="space-y-2">
            <div v-for="item in dimensions.by_merchant" :key="item.slug" class="rounded-lg border border-border px-3 py-2">
              <div class="flex items-center justify-between gap-2">
                <span class="min-w-0 truncate text-sm font-medium">{{ item.name }}</span>
                <span class="shrink-0 font-mono text-xs text-muted-foreground">{{ item.slug }}</span>
              </div>
              <div class="mt-1 flex items-center justify-between text-xs text-muted-foreground">
                <span>新单 {{ item.total }} / 支付 {{ item.paid }}</span>
                <span class="font-semibold text-foreground">{{ conversionPct(item.paid, item.total) }}%</span>
              </div>
            </div>
          </div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-sm">按操作系统分布</CardTitle>
        </CardHeader>
        <CardContent>
          <div v-if="loadingDimensions" class="text-sm text-muted-foreground">{{ t('admin.common.loading') }}</div>
          <div v-else-if="!dimensions || dimensions.by_os.length === 0" class="text-sm text-muted-foreground">暂无数据</div>
          <div v-else class="space-y-2">
            <div v-for="item in dimensions.by_os" :key="item.key" class="rounded-lg border border-border px-3 py-2">
              <div class="flex items-center justify-between gap-2">
                <span class="text-sm font-medium uppercase">{{ item.key }}</span>
                <span class="font-mono text-xs">{{ item.total }} 单</span>
              </div>
              <div class="mt-1 flex items-center justify-between text-xs text-muted-foreground">
                <span>已支付 {{ item.paid }}</span>
                <span class="font-semibold text-foreground">{{ conversionPct(item.paid, item.total) }}%</span>
              </div>
            </div>
          </div>
        </CardContent>
      </Card>

      <Card class="min-w-0">
        <CardHeader class="pb-2">
          <CardTitle class="text-sm">按浏览器分布</CardTitle>
        </CardHeader>
        <CardContent>
          <div v-if="loadingDimensions" class="text-sm text-muted-foreground">{{ t('admin.common.loading') }}</div>
          <div v-else-if="!dimensions || dimensions.by_browser.length === 0" class="text-sm text-muted-foreground">暂无数据</div>
          <div v-else class="space-y-2">
            <div v-for="item in dimensions.by_browser" :key="item.key" class="rounded-lg border border-border px-3 py-2">
              <div class="flex items-center justify-between gap-2">
                <span class="text-sm font-medium uppercase">{{ item.key }}</span>
                <span class="font-mono text-xs">{{ item.total }} 单</span>
              </div>
              <div class="mt-1 flex items-center justify-between text-xs text-muted-foreground">
                <span>已支付 {{ item.paid }}</span>
                <span class="font-semibold text-foreground">{{ conversionPct(item.paid, item.total) }}%</span>
              </div>
            </div>
          </div>
        </CardContent>
      </Card>
    </div>

    <Card class="min-w-0">
      <CardHeader class="pb-2">
        <CardTitle class="text-sm">最近活跃账户（邮箱 / QQ / 手机号）</CardTitle>
      </CardHeader>
      <CardContent>
        <div v-if="loadingDimensions" class="text-sm text-muted-foreground">{{ t('admin.common.loading') }}</div>
        <div v-else-if="!dimensions || dimensions.recent_contacts.length === 0" class="text-sm text-muted-foreground">暂无活跃账户</div>
        <div v-else class="overflow-x-auto">
          <table class="w-full text-sm">
            <thead>
              <tr class="text-left text-xs text-muted-foreground">
                <th class="py-2 pr-3">账户</th>
                <th class="py-2 pr-3">商户</th>
                <th class="py-2 pr-3">订单数</th>
                <th class="py-2 pr-3">操作系统</th>
                <th class="py-2 pr-3">浏览器</th>
                <th class="py-2 pr-3">支付金额</th>
                <th class="py-2 pr-3">最近支付</th>
                <th class="py-2">最近下单</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in dimensions.recent_contacts" :key="`${item.contact}-${item.merchant_slug}`" class="border-t border-border">
                <td class="break-all py-2 pr-3 font-mono text-xs">{{ item.contact }}</td>
                <td class="py-2 pr-3 text-xs text-muted-foreground">{{ item.merchant_slug }}</td>
                <td class="py-2 pr-3">{{ item.order_count }}</td>
                <td class="py-2 pr-3 text-xs uppercase">{{ item.device_os }}</td>
                <td class="py-2 pr-3 text-xs uppercase">{{ item.browser }}</td>
                <td class="py-2 pr-3 whitespace-nowrap font-mono text-xs">
                  {{ item.paid_amount == null ? '-' : formatMoney(item.paid_amount, item.paid_currency || overview?.currency) }}
                </td>
                <td class="py-2 pr-3 text-xs">{{ formatLastSeen(item.last_paid_at) }}</td>
                <td class="py-2 text-xs">{{ formatLastSeen(item.last_created_at) }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </CardContent>
    </Card>

  </div>
</template>

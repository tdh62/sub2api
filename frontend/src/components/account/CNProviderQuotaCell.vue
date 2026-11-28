<template>
  <div
    v-if="visible"
    data-test="cn-provider-quota"
    class="min-w-[220px] space-y-1"
  >
    <!-- Tier rows: 5h + weekly utilization bars from the current snapshot/probe.
         复用账号页 UsageProgressBar：同阈值配色、同倒计时格式。 -->
    <div v-if="data?.success && data.tiers?.length" class="space-y-1">
      <UsageProgressBar
        v-for="tier in data.tiers"
        :key="tier.window"
        data-test="cn-provider-quota-tier"
        :label="windowLabel(tier.window)"
        :color="tier.window === 'weekly' ? 'emerald' : 'indigo'"
        :utilization="tier.used_percent"
        :resets-at="tier.reset_at"
      />
    </div>

    <!-- Explicit refresh action (aligned with the OpenAI "Query" / Grok "Probe"
         buttons): a verb label tells users this chip is clickable. The previous
         noun label ("5h/weekly") read as a passive caption and users could not
         discover the manual refresh. -->
    <div class="flex flex-wrap items-center gap-1.5">
      <button
        type="button"
        data-test="cn-provider-quota-probe"
        class="inline-flex items-center gap-0.5 whitespace-nowrap rounded px-1.5 py-0.5 text-[10px] font-medium leading-4 text-blue-600 transition-colors hover:bg-blue-50 disabled:cursor-not-allowed disabled:opacity-50 dark:text-blue-400 dark:hover:bg-blue-900/30"
        :disabled="loading"
        :title="t('admin.accounts.cnProviders.probeTooltip')"
        @click="handleProbe()"
      >
        <svg
          class="h-2.5 w-2.5"
          :class="{ 'animate-spin': loading }"
          fill="none"
          stroke="currentColor"
          viewBox="0 0 24 24"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2"
            d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15"
          />
        </svg>
        {{ t('admin.accounts.cnProviders.probe') }}
      </button>
    </div>

    <div
      v-if="error"
      class="truncate text-[10px] leading-4 text-red-600 dark:text-red-400"
      :title="error"
    >
      {{ truncatedError }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, ref, watch } from 'vue'
import { useI18n } from 'vue-i18n'
import { adminAPI } from '@/api/admin'
import type { CNProviderQuotaProbeResult } from '@/api/admin/cnProviders'
import type { Account } from '@/types'
import { cnQuotaCellVisible } from './credentialsBuilder'
import UsageProgressBar from './UsageProgressBar.vue'

const props = defineProps<{
  account: Account
  manualRefreshToken?: number
}>()

const { t } = useI18n()

const visible = computed(() => cnQuotaCellVisible(props.account))

const loading = ref(false)
const error = ref<string | null>(null)
const probeData = ref<CNProviderQuotaProbeResult | null>(null)
let probeGeneration = 0

const readExtraNumber = (key: string): number | null => {
  const v = (props.account.extra as Record<string, unknown> | undefined)?.[key]
  return typeof v === 'number' && Number.isFinite(v) ? v : null
}

const readExtraString = (key: string): string => {
  const v = (props.account.extra as Record<string, unknown> | undefined)?.[key]
  return typeof v === 'string' ? v : ''
}

// 从持久化快照构造展示数据（缺少 5h/weekly 两档键时返回 null）。
const snapshotData = computed<CNProviderQuotaProbeResult | null>(() => {
  const platform = props.account.platform
  const used5h = readExtraNumber(`${platform}_5h_used_percent`)
  const usedWeekly = readExtraNumber(`${platform}_weekly_used_percent`)
  const usedMonthly = readExtraNumber(`${platform}_monthly_used_percent`)
  if (used5h == null && usedWeekly == null && usedMonthly == null) return null
  const tiers: CNProviderQuotaProbeResult['tiers'] = []
  if (used5h != null) {
    tiers.push({ window: '5h', used_percent: used5h, reset_at: readExtraString(`${platform}_5h_reset_at`) || undefined })
  }
  if (usedWeekly != null) {
    tiers.push({ window: 'weekly', used_percent: usedWeekly, reset_at: readExtraString(`${platform}_weekly_reset_at`) || undefined })
  }
  if (usedMonthly != null) {
    tiers.push({ window: 'monthly', used_percent: usedMonthly, reset_at: readExtraString(`${platform}_monthly_reset_at`) || undefined })
  }
  return { success: true, tiers } as CNProviderQuotaProbeResult
})

const data = computed(() => probeData.value ?? snapshotData.value)

// A list refresh replaces account.extra even when its quota snapshot is
// unchanged. Keep a manual probe until the quota values or snapshot version
// actually change, rather than reverting it on unrelated account updates.
watch(
  [snapshotData, () => readExtraString(`${props.account.platform}_usage_updated_at`)],
  (next, previous) => {
    if (JSON.stringify(next) !== JSON.stringify(previous)) probeData.value = null
  }
)

const extractErrorMessage = (e: unknown): string => {
  const err = e as {
    message?: string
    reason?: string
    response?: { data?: { message?: string; error?: string } }
  }
  return (
    err?.message ||
    err?.reason ||
    err?.response?.data?.message ||
    err?.response?.data?.error ||
    t('common.error')
  )
}

const truncatedError = computed(() => {
  if (!error.value) return ''
  return error.value.length > 80 ? `${error.value.slice(0, 80)}...` : error.value
})

const windowLabel = (window: string) => {
  if (window === 'weekly') return t('admin.accounts.cnProviders.windowWeekly')
  if (window === 'monthly') return t('admin.accounts.cnProviders.windowMonthly')
  return t('admin.accounts.cnProviders.window5h')
}

const handleProbe = async () => {
  if (loading.value) return
  const generation = ++probeGeneration
  const accountID = props.account.id
  loading.value = true
  error.value = null
  try {
    const result = await adminAPI.cnProviders.queryQuota(accountID)
    if (generation !== probeGeneration) return
    // 失败时保留已渲染的快照条形图（仅显示错误行），成功才覆盖。
    if (result.success) {
      probeData.value = result
    } else {
      error.value = result.error || t('common.error')
    }
  } catch (e) {
    if (generation === probeGeneration) error.value = extractErrorMessage(e)
  } finally {
    if (generation === probeGeneration) loading.value = false
  }
}

watch(() => props.manualRefreshToken, () => {
  if (visible.value) void handleProbe()
})

watch(
  [() => props.account.id, () => props.account.platform],
  () => {
    probeGeneration++
    probeData.value = null
    error.value = null
    loading.value = false
  }
)
</script>

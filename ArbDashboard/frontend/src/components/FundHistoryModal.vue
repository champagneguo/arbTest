<template>
  <!-- 历史记录弹窗（静态估值）：2026-10-07 从 Dashboard.vue 抽为共享组件，
       主看板与实时估值页（TradeWorkbench → LazyMode/Analysis）共用，逻辑与样式保持一致 -->
  <n-modal :show="show" preset="card" :title="modalTitle" style="width: 96vw; max-width: 96vw; height: 90vh;" @update:show="(v: boolean) => emit('update:show', v)">
    <div class="history-modal-body">
      <n-tabs v-model:value="historyView" type="segment" size="small" style="margin-bottom: 10px;">
        <!-- [AI-2026-10-08 东哥拍板] TAB 名随内容改：表已换成带实时价的持仓表 -->
        <n-tab-pane name="detail" tab="持仓估值" />
        <n-tab-pane name="error" tab="历史误差" />
      </n-tabs>

      <!-- 视图A：基金详情 -->
      <div v-if="historyView === 'detail'">
        <!-- 估值基准日提示：= 静态估值公式所用基准净值日期（≠T，T是实盘交易日） -->
        <div class="hd-tbar" v-if="baseDateLabel">估值基准日 <b>{{ baseDateLabel }}</b></div>
        <!-- 基础信息条：复现主看板该行关键字段 -->
        <div class="hd-baseinfo" v-if="fundRow">
          <div class="hd-bi-cell"><span class="hd-bi-k">现价</span><span class="hd-bi-v">{{ fmtPrice(curPrice(fundRow)) }}</span></div>
          <div class="hd-bi-cell"><span class="hd-bi-k">涨跌</span><span class="hd-bi-v" :style="{ color: priceColor(fundRow.chg ?? 0) }">{{ fmtChg(fundRow.chg) }}</span></div>
          <div class="hd-bi-cell"><span class="hd-bi-k">成交额</span><span class="hd-bi-v">{{ fmtAmount(fundRow.amount) }}</span></div>
          <div class="hd-bi-cell"><span class="hd-bi-k">{{ navLabel }}</span><span class="hd-bi-v">{{ fmtVal(fundRow.nav) }}</span></div>
          <div class="hd-bi-cell"><span class="hd-bi-k">{{ valLabel }}</span><span class="hd-bi-v">{{ fmtVal(fundRow.static_val) }}</span></div>
          <div class="hd-bi-cell"><span class="hd-bi-k">{{ premLabel }}</span><span class="hd-bi-v" :style="{ color: priceColor(fundRow.static_premium ?? 0) }">{{ fmtPrem(fundRow.static_premium) }}</span></div>
          <div class="hd-bi-cell"><span class="hd-bi-k">实时估值</span><span class="hd-bi-v">{{ fmtVal(fundRow.rt_val) }}</span></div>
          <div class="hd-bi-cell"><span class="hd-bi-k">实时溢价</span><span class="hd-bi-v" :style="{ color: priceColor(fundRow.rt_premium ?? 0) }">{{ fmtPrem(fundRow.rt_premium) }}</span></div>
          <!-- 汇率组：并入同一行，竖线分隔、间距加大 -->
          <div class="hd-bi-fx" v-if="!isCashManagementFund && fxRows.length">
            <span class="hd-fx-label">汇率 · {{ fxCcyText }}</span>
            <div class="hd-fx-cell" v-for="fx in fxRows" :key="fx.tag">
              <span class="hd-fx-tag">{{ fx.tag }}</span>
              <span class="hd-fx-val">{{ fx.rate }}</span>
              <span class="hd-fx-chg" :style="{ color: priceColor(fx.chg) }">{{ fx.chgTxt }}</span>
            </div>
          </div>
        </div>

        <!-- 持仓实时估值（2026-10-08 东哥拍板：替换原「估值明细」表）
             数据源 = useValuationCalculator.holdingRows（valuation_meta 现成字段，含 realtime_quotes），
             表格 = 共享组件 RtHoldingTable，与「实时估值」页那张表**同一个组件**（格式/配色永不走偏）。 -->
        <RtHoldingTable
          :rows="rtHoldingRows"
          title="持仓实时估值"
          hint="盘中涨跌 = 实时价 ÷ T-1收盘 − 1 · 估值影响 = 占比 × 盘中涨跌"
          show-empty
        />

        <!-- 估值算法：直接展示公式（不折叠，单行紧凑） -->
        <div class="hd-algo">
          <span class="hd-algo-title">估值算法</span>
          <span class="hd-algo-body">静态估值 = 基准日净值 × ( 1 + Σᵢ 占净值ᵢ × 人民币口径涨跌ᵢ )，其中 人民币口径涨跌ᵢ = ( 1 + 标的涨跌ᵢ ) × ( 1 + 汇率涨跌ᵢ ) − 1</span>
        </div>

        <!-- [AI-2026-10-07 方案A·东哥拍板] 补采=应急入口，收进维护折叠，平时不挡眼 -->
        <n-collapse style="margin-top: 12px;">
          <n-collapse-item title="维护 · 补采历史并重算静态估值（应急）" name="maint">
            <div style="display: flex; align-items: center; gap: 10px; flex-wrap: wrap;">
              <n-button size="small" type="primary" :loading="reconciling" @click="reconcileStaticValHandler">核对静态估值（补采近10日+重算）</n-button>
              <span style="font-size: 12px; color: #64748b;">补采该基金历史价格/净值，并级联补底层ETF日价后重算静态估值；仅在自动采集缺数时使用</span>
            </div>
          </n-collapse-item>
        </n-collapse>
      </div>

      <!-- 视图B：历史误差（极简四列，每页15天） -->
      <div v-else>
        <n-data-table
          :columns="errorColumns"
          :data="pagedErrorRows"
          size="small"
          bordered
          :row-props="errorRowProps"
          style="width: 100%;"
        />
        <div class="hd-error-foot">
          <n-pagination
            v-model:page="errorPage"
            :page-size="errorPageSize"
            :item-count="errorRows.length"
            :page-slot="7"
            size="small"
          />
          <span class="hd-note" style="margin: 0;">共 {{ errorRows.length }} 天 · 每页 {{ errorPageSize }} 天 · 点击任意一行 → 弹出该日各估值标的变动明细</span>
        </div>
      </div>
    </div>
  </n-modal>

  <!-- 视图B 行点击：某日各持仓股变动弹窗 -->
  <n-modal v-model:show="showDayModal" preset="card" :title="`估值标的变动 · ${dayDetailRow?.date || ''}`" style="width: 70vw; max-width: 900px;">
    <n-data-table v-if="dayDetailRow" :columns="dayHoldColumns" :data="dayHoldRows" size="small" bordered :row-key="(r: any) => r.sym" style="width: 100%;" />
  </n-modal>
</template>

<script setup lang="ts">
import { ref, computed, watch, onUnmounted, h } from 'vue'
import { storeToRefs } from 'pinia'
import { NModal, NTabs, NTabPane, NDataTable, NPagination, NButton, NCollapse, NCollapseItem, useMessage } from 'naive-ui'
import type { DataTableColumns } from 'naive-ui'
import { useFundStore } from '../store'
import { formatPrice, formatValuation, formatPremium, formatVolume, priceColor, shortDate } from '../utils'
import { reconcileStaticVal } from '../api'
import { useValuationCalculator } from '../composables/useValuationCalculator'
// [AI-2026-10-08] 持仓实时价表 = 与「实时估值」页同一个组件（格式/配色/排版唯一来源）
import { RtHoldingTable } from './shared'

const props = defineProps<{
  show: boolean
  fundCode: string
  /** 可选：主看板行（含 price/nav/static_val/rt_val 等全量字段）。
   *  实时估值页等无看板行的场景可传 {fund_code, fund_name}，组件内部会尝试从
   *  看板 store 查找完整行并用历史行兜底，缺的字段显示 '-'。 */
  fund?: any
}>()

const emit = defineEmits<{
  (e: 'update:show', v: boolean): void
}>()

const fundStore = useFundStore()
const message = useMessage()
const { fundHistory, tableData } = storeToRefs(fundStore)

// [AI-2026-08-03] 「现价」优先显示盘中 LOF 实时价(realtime_price)，非交易时段回退收盘价。
const curPrice = (row: any): number => {
  const rp = row?.realtime_price
  if (rp && rp > 0) return rp
  return row?.price || 0
}

// 基金行解析：优先用调用方传入的完整行；否则从看板 store 查找，再用历史首行补 nav/static_val。
const fundRow = computed<any>(() => {
  if (props.fund && props.fund.price != null) return props.fund
  const board = (tableData.value || []).find((r: any) => r.fund_code === props.fundCode)
  const hist: any = (fundHistory.value || [])[0] || {}
  const merged: any = { ...(board || {}), ...(props.fund || {}), fund_code: props.fundCode }
  if ((merged.nav == null) && hist.nav != null) merged.nav = hist.nav
  if ((merged.static_val == null) && hist.static_val != null) merged.static_val = hist.static_val
  if ((merged.static_premium == null) && hist.static_premium != null) merged.static_premium = hist.static_premium
  return merged
})

const modalTitle = computed(() => {
  const name = fundRow.value?.fund_name ? ` - ${fundRow.value.fund_name}` : ''
  // [AI-2026-10-08 东哥拍板] 「[历史记录]」→「基金详情」（这已不是纯历史表，含实时持仓）
  return `[基金详情] ${props.fundCode}${name}`
})

const isCashManagementFund = computed(() => {
  return ['511880', '511360', '511520'].includes(props.fundCode)
})

// [AI-2026-10-06] 历史弹窗重构：视图A(基金详情) / 视图B(历史误差) 两视图切换
const historyView = ref<'detail' | 'error'>('detail')
const showDayModal = ref(false)
const dayDetailRow = ref<any>(null)
// 详情视图本地格式化助手（复用 utils 已有 formatter）
const fmtPrice = (v: any) => (v == null ? '-' : formatPrice(v))
// [AI-2026-10-07 东哥要求] 顶部信息条 净值/估值/实时估值 保留 4 位小数
const fmtVal = (v: any) => (v == null ? '-' : formatValuation(v))
const fmtPrem = (v: any) => (v == null ? '-' : formatPremium(v))
const fmtChg = (v: any) => (v == null ? '-' : (v >= 0 ? '+' : '') + Number(v).toFixed(2) + '%')
const fmtAmount = (v: any) => (v == null ? '-' : formatVolume(v))

// [AI-2026-10-07] 估值来源持仓：以基金配置 valuation_portfolio + 跟踪指数 related_index 为权威，
// 不再扫描历史行 *_price 键（指数法基金无篮子→空白/错显、且会串显其它基金标的）。
// 指数行取 index_close/index_pct（跟踪指数点位与日涨跌），参考ETF行取 {sym}_price/{sym}_price_chg
// （篮子基金有值，指数法基金为 '-'）。即：指数占 100% 为估值基准，ETF 作参考，符合 woody 口径。
const holdingSource = computed<any[]>(() => {
  const rows = fundHistory.value || []
  const r0 = rows[0] || {}
  const vp: any[] = Array.isArray(r0.valuation_portfolio) ? r0.valuation_portfolio : []
  const relIdx: string = (r0.related_index && r0.related_index !== '-') ? String(r0.related_index) : ''
  const out: any[] = []
  if (relIdx) {
    out.push({
      sym: relIdx,
      weight: 100,
      kind: '指数',
      t_price: r0.index_close ?? null,
      t_chg: r0.index_pct ?? null,
    })
  }
  for (const item of vp) {
    const s = item?.symbol || ''
    if (!s || s === relIdx) continue
    out.push({
      sym: s,
      weight: item.weight ?? null,
      kind: '参考',
      t_price: r0[`${s}_price`] ?? null,
      t_chg: r0[`${s}_price_chg`] ?? null,
    })
  }
  return out
})
// [AI-2026-10-08 东哥拍板] 「估值明细」→「持仓实时估值」
// 数据源不复用本文件原有的 fundHistory 派生（那是 T 日收盘口径、盘中不跳动），
// 改为复用 useValuationCalculator.holdingRows —— 与「实时估值」页同一 composable，
// 其 rt 来自 valuation_meta.realtime_quotes（LOFarb 从 IB/FUTU 读），
// ⇒ 盘中跳动，且两处口径必然一致。
// [AI-2026-10-08 东哥二次拍板] 表格**格式/颜色/字体/排版**也要与实时估值页完全一致
//   ⇒ 不再自建 n-data-table（那是另一套视觉语言），直接用共享组件 RtHoldingTable，
//      与 ValuationCalculator.vue 共用同一实现 ⇒ 以后改样式只改一处，不会漂移。
const vc = useValuationCalculator()
const rtHoldingRows = computed<any[]>(() => {
  if (!props.fundCode) return []
  if (vc.fundCode.value !== props.fundCode) {
    vc.fundCode.value = props.fundCode
    vc.fetchValuationMeta()
  }
  return vc.holdingRows.value as any[]
})

// 视图B：历史误差四列（日期/净值/估值/误差），每页15天（东哥拍板 2026-10-07）
const errorPage = ref(1)
const errorPageSize = 15
// [AI-2026-10-07 修复] 必须 ...r 展开整行：点行弹窗（dayHoldRows）要从同一对象读 {sym}_price/_price_chg/_weight，
// 只留四列会导致弹窗全"-"（估值标的现由 valuation_portfolio/related_index 派生，故行对象须展开整行）
const errorRows = computed<any[]>(() => (fundHistory.value || []).map((r: any) => ({
  ...r,
  date: r.date,
  nav: r.nav,
  val: r.static_val,
  err: (r.static_val != null && r.nav != null) ? r.static_val - r.nav : null
})))
const pagedErrorRows = computed(() => {
  const start = (errorPage.value - 1) * errorPageSize
  return errorRows.value.slice(start, start + errorPageSize)
})
watch(fundHistory, () => { errorPage.value = 1 })
const errorColumns = computed<DataTableColumns<any>>(() => [
  { title: '日期', key: 'date', width: 100, align: 'center', render: (r: any) => h('span', { class: 'num-cell' }, shortDate(r.date)) },
  // [AI-2026-10-07 东哥要求] 净值列保留 4 位小数（与估值列同精度）
  { title: '净值', key: 'nav', width: 120, align: 'center', render: (r: any) => h('span', { class: 'num-cell' }, formatValuation(r.nav)) },
  { title: '估值', key: 'val', width: 120, align: 'center', render: (r: any) => h('span', { class: 'num-cell' }, formatValuation(r.val)) },
  { title: '误差', key: 'err', width: 110, align: 'center', render: (r: any) => {
    if (r.err == null) return h('span', { style: 'color: var(--co-neutral)' }, '-')
    const color = r.err > 0 ? 'var(--co-up)' : (r.err < 0 ? 'var(--co-down)' : 'var(--co-text)')
    return h('span', { style: `color:${color};font-weight:600;` }, (r.err >= 0 ? '+' : '') + Number(r.err).toFixed(4))
  } },
  // [AI-2026-10-07 东哥要求] 误差比例 = 误差 ÷ 净值，红正绿负，百分比 2 位小数
  { title: '误差比例', key: 'errPct', width: 110, align: 'center', render: (r: any) => {
    if (r.err == null || !r.nav) return h('span', { style: 'color: var(--co-neutral)' }, '-')
    const pct = (r.err / r.nav) * 100
    const color = pct > 0 ? 'var(--co-up)' : (pct < 0 ? 'var(--co-down)' : 'var(--co-text)')
    return h('span', { style: `color:${color};font-weight:600;` }, (pct >= 0 ? '+' : '') + pct.toFixed(2) + '%')
  } },
])
const errorRowProps = (row: any) => ({ onClick: () => { dayDetailRow.value = row; showDayModal.value = true } })

// 视图B 行点击：该日各持仓股变动
const dayHoldRows = computed<any[]>(() => {
  const r = dayDetailRow.value
  if (!r) return []
  return holdingSource.value.map((s, i) => ({
    idx: i + 1, sym: s.sym, kind: s.kind,
    weight: s.weight,
    // [AI-2026-10-07 东哥抓错] 指数行只认 index_close/index_pct；`{sym}_price` 会被同名美股 ETF 污染
    // （DAX 指数 ~25266 vs Global X DAX ETF 44.71，历史 ETF 数据只到 09-24 ⇒ 越往前越错）。
    price: s.kind === '指数' ? (r.index_close ?? null) : (r[`${s.sym}_price`] ?? null),
    chg: s.kind === '指数' ? (r.index_pct ?? null) : (r[`${s.sym}_price_chg`] ?? null),
  })).sort((a, b) => (b.weight ?? -1) - (a.weight ?? -1))
    .map((r, i) => ({ ...r, idx: i + 1 }))
})
const dayHoldColumns = computed<DataTableColumns<any>>(() => [
  { title: '序号', key: 'idx', width: 52, align: 'center', render: (r: any) => h('span', { style: 'color:#94a3b8;' }, r.idx) },
  { title: '估值标的', key: 'sym', width: 110, align: 'center', render: (r: any) => h('span', { style: 'font-weight:600;color:#1e293b;' }, r.sym) },
  { title: '类型', key: 'kind', width: 64, align: 'center', render: (r: any) => h('span', { style: `font-size:12px;padding:1px 6px;border-radius:4px;${r.kind === '指数' ? 'background:#ECFDF5;color:#059669;' : 'background:#F1F5F9;color:#64748b;'}` }, r.kind) },
  { title: '占比', key: 'weight', width: 90, align: 'center', render: (r: any) => pctCell(r.weight) },
  { title: '收盘价', key: 'price', width: 110, align: 'center', render: (r: any) => priceCell(r.price) },
  { title: '涨跌幅', key: 'chg', width: 100, align: 'center', render: (r: any) => chgCell(r.chg) },
])

// 汇率行 T-2 → T-1 → T
// [AI-2026-10-07 二修·东哥抓出错位] 标签跟数据走：净值/估值取自 fundHistory 哪一行，就算出相对 T 的 offset 动态命名。
// 盘中活日：最新已公布净值=nav(T-2)、最新预测=static_val(T-1) → 自动显示 T-2净值/T-1估值（与主看板口径一致）；
// 净值公布后/节假日冻结：净值前移一行 → 自动变 T-1净值/T日估值，不再错位。fundHistory 为降序（[0]=T 日）。
const _US_T = ['黄金原油', 'QDII欧美']
const _ASIA_T = ['QDII日本', 'QDII亚洲', '国内LOF', '白银']
const _offTxt = (i: number) => (i <= 0 ? 'T日' : `T-${i}`)
const navRowIdx = computed(() => (fundHistory.value || []).findIndex(r => r.nav != null && r.nav > 0))
const valRowIdx = computed(() => (fundHistory.value || []).findIndex(r => r.static_val != null && r.static_val > 0))
const navLabel = computed(() => {
  const i = navRowIdx.value
  if (i >= 0) return `${_offTxt(i)}净值`
  const c = fundRow.value?.category
  if (_US_T.includes(c)) return 'T-2净值'
  if (_ASIA_T.includes(c)) return 'T-1净值'
  return '净值'
})
const valLabel = computed(() => {
  const i = valRowIdx.value
  if (i >= 0) return `${_offTxt(i)}估值`
  const c = fundRow.value?.category
  if (_US_T.includes(c)) return 'T-1估值'
  if (_ASIA_T.includes(c)) return 'T日估值'
  return '估值'
})
// 溢价的分母=上面显示的那条估值，标签随 valLabel 同步
const premLabel = computed(() => {
  const i = valRowIdx.value
  if (i >= 0) return `${_offTxt(i)}溢价`
  return 'T-1溢价'
})
// [AI-2026-10-07] 估值基准日 = 静态估值公式所用基准净值日期（≠T，T是实盘交易日）
// 黄金原油/QDII欧美 滞后2日→基准=T-2；其余滞后1日→基准=T-1。日期取 fundHistory 对应偏移行。
const baseOffset = computed(() => {
  const c = fundRow.value?.category
  if (_US_T.includes(c)) return 2
  if (_ASIA_T.includes(c)) return 1
  return 1
})
const baseDateLabel = computed(() => {
  const off = baseOffset.value
  const rows = fundHistory.value || []
  const r = rows[off] || rows[rows.length - 1]
  const d = r?.date ? String(r.date).slice(5) : ''
  return `T-${off} = ${d}`
})
const fxCcy = computed<string>(() => {
  const rows = fundHistory.value || []
  const c = rows[0]?.fx_currency
  return c || 'USD'
})
const fxCcyText = computed<string>(() => ({ USD: '美元', EUR: '欧元', JPY: '日元', HKD: '港币' } as Record<string, string>)[fxCcy.value] || fxCcy.value)
const fxRows = computed<any[]>(() => {
  if (isCashManagementFund.value) return []
  const rows = fundHistory.value || []
  if (!rows.length) return []
  const tags = ['T-2', 'T-1', 'T']
  const ccy = fxCcy.value
  return [2, 1, 0].map((idx, i) => {
    const r = rows[idx]
    if (!r) return null
    const rate = r.usd_cny_mid
    const chg = r.usd_cny_mid_chg
    return {
      tag: tags[i],
      ccy,
      rate: rate != null ? Number(rate).toFixed(4) : '-',
      chg: chg ?? 0,
      chgTxt: chg == null ? '-' : (chg >= 0 ? '+' : '') + Number(chg).toFixed(2) + '%'
    }
  }).filter(Boolean) as any[]
})

// 弹窗打开/切换基金：重置视图 + 拉历史；切换 fundCode 也重拉，避免沿用上一支基金的持仓/汇率（串显）
watch([() => props.show, () => props.fundCode], ([v]) => {
  if (!v) {
    // 弹窗关闭：停轮询（否则定时器泄漏 + 后台白刷 valuation_meta）
    vc.stopPolling()
    return
  }
  historyView.value = 'detail'
  if (props.fundCode) fundStore.fetchFundHistory(props.fundCode)
  if (!(props.fund && props.fund.price != null) && tableData.value.length === 0) {
    fundStore.fetchDashboard(true).catch(() => {})
  }
  // [AI-2026-10-08] 「T 实时价」要盘中跳动 ⇒ 开轮询（composable 自带，关闭时由上面 stopPolling 收回）
  if (props.fundCode) {
    vc.fundCode.value = props.fundCode
    vc.fetchValuationMeta()
    vc.startPolling()
  }
})

// 组件卸载兜底停轮询
onUnmounted(() => vc.stopPolling())

// [AI-2026-08-04] 单基金「核对静态估值」：补采该基金近10日历史价格/净值
// （级联底层ETF如VGT日价）并重算静态估值，仅影响当前基金。
const reconciling = ref(false)
const reconcileStaticValHandler = async () => {
  if (!props.fundCode) return
  reconciling.value = true
  try {
    const res = await reconcileStaticVal(props.fundCode, 10)
    if (res.data?.status === 'ok') {
      const s = res.data.data?.stats || {}
      message.success(`「${props.fundCode}」核对完成：价格${s.lof_price || 0}天 / 净值${s.lof_nav || 0}天 / 底层ETF${Object.keys(s.etf || {}).join(',') || '无'}已补采，静态估值已重算`)
      await fundStore.fetchFundHistory(props.fundCode)
      errorPage.value = 1
    } else {
      message.error(`核对失败: ${res.data?.message || '未知错误'}`)
    }
  } catch (e: any) {
    message.error(`核对失败: ${e?.message || e}`)
  } finally {
    reconciling.value = false
  }
}
</script>

<style scoped>
.history-modal-body {
  display: flex;
  flex-direction: column;
  flex: 1 1 auto;
  min-height: 0;
  width: 100%;
}
/* [AI-2026-10-07] 历史弹窗：T基准日提示 / 估值算法卡片 */
.hd-tbar {
  font-size: var(--co-fs-sm); color: var(--co-text-3); margin-bottom: 8px;
}
.hd-tbar b {
  color: var(--co-text); font-weight: 700; font-variant-numeric: tabular-nums;
  background: var(--co-accent-soft); padding: 1px 7px; border-radius: 5px; margin-left: 2px;
}
.hd-algo {
  margin-top: 12px; background: var(--co-surface-2); border: 1px solid var(--co-border);
  border-radius: var(--co-radius); padding: 7px 12px;
  display: flex; align-items: baseline; gap: 10px; white-space: nowrap;
}
.hd-algo-title { font-size: var(--co-fs-sm); font-weight: 700; color: var(--co-text); flex-shrink: 0; }
.hd-algo-body {
  font-size: var(--co-fs-sm); color: var(--co-text-2);
  font-variant-numeric: tabular-nums; overflow: hidden; text-overflow: ellipsis;
}
/* [AI-2026-10-06] 历史弹窗重构：视图A 基础信息条 / 汇率行 / 持仓标题 / 注释 */
.hd-baseinfo {
  display: flex; flex-wrap: wrap; gap: 6px 18px;
  background: var(--co-surface-2); border: 1px solid var(--co-border); border-radius: var(--co-radius);
  padding: 8px 12px; margin-bottom: 8px;
}
.hd-bi-cell { display: flex; align-items: baseline; gap: 5px; }
.hd-bi-k { font-size: var(--co-fs-xs); color: var(--co-text-3); }
.hd-bi-v { font-size: var(--co-fs-base); font-weight: 700; color: var(--co-text); font-variant-numeric: tabular-nums; }
.hd-bi-fx {
  display: flex; align-items: baseline; gap: 8px; flex-wrap: wrap;
  margin-left: 14px; padding-left: 22px;
  border-left: 1px solid var(--co-border);
}
.hd-fx-label { font-size: var(--co-fs-xs); color: var(--co-text-3); font-weight: 600; }
.hd-fx-cell { display: flex; align-items: baseline; gap: 5px; }
.hd-fx-tag { font-size: var(--co-fs-xs); color: var(--co-text-2); font-weight: 600; }
.hd-fx-val { font-size: var(--co-fs-base); font-weight: 700; color: var(--co-text); font-variant-numeric: tabular-nums; }
.hd-fx-chg { font-size: var(--co-fs-xs); font-weight: 600; font-variant-numeric: tabular-nums; }
/* [AI-2026-10-08] .hd-hold-title 已随持仓表迁到共享组件 RtHoldingTable 内（其 .rt-hold-head） */
.hd-note { font-size: var(--co-fs-xs); color: var(--co-text-3); margin-top: 8px; }
.hd-error-foot {
  display: flex; align-items: center; gap: 12px;
  margin-top: 8px; flex-wrap: wrap;
}
/* 表格数值单元格：n-data-table render 函数元素带本组件 scopeId，scoped 可命中 */
.num-cell {
  font-family: "Inter", "Fira Code", Consolas, sans-serif;
  font-size: 13px;
  font-variant-numeric: tabular-nums;
  color: #1f2937;
}
</style>

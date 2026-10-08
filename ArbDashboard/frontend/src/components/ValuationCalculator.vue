<template>
  <div>
    <!-- =============================================================
         统一参数区: T-2 基准日 + T-1 估值日 + 实时数据
         ============================================================= -->
    <n-card
      v-if="!calc.isCashManagement.value"
      :bordered="false"
      class="shadow-soft"
      style="margin-bottom: 8px; background: var(--co-surface); border: 1px solid var(--co-border); padding: 0"
      :content-style="{ padding: '8px 16px' }"
    >
      <div style="display: flex; flex-direction: column; gap: 2px">
        <!-- === 第1行: T-2 基准日 === -->
        <div
          class="base-info-row"
          style="display: flex; gap: 12px; font-size: 13px; color: var(--co-text-2); align-items: center; font-weight: 500"
        >
          <div style="width: 160px">
            <strong>【T-2 基准日】</strong>
            {{ calc.meta.value?.base_data?.date ? calc.meta.value.base_data.date.substring(5) : '-' }}
          </div>
          <n-divider vertical style="margin: 0" />
          <div style="width: 140px">
            💰 <strong>净值</strong>
            <span style="color: var(--co-accent-text); font-weight: bold; font-family: monospace">{{
              Number(calc.meta.value?.base_data?.nav || 0).toFixed(4)
            }}</span>
          </div>
          <n-divider vertical style="margin: 0" />
          <div style="width: 110px">
            💱 <strong>汇率</strong>
            <span style="font-family: monospace">{{
              Number(calc.meta.value?.base_data?.exchange_rate || 0).toFixed(4)
            }}</span>
          </div>
          <n-divider vertical style="margin: 0" />
          <template v-if="calc.meta.value?.fund_config?.trade_future">
            <n-divider vertical style="margin: 0" />
            <div style="width: 160px">
              📊 <strong>{{ calc.meta.value?.fund_config?.trade_future }}校准因子</strong>
              <span style="font-family: monospace; color: var(--co-text-2)">{{
                calc.meta.value?.base_data?.calibration
                  ? Number(calc.meta.value.base_data.calibration).toFixed(3)
                  : '-'
              }}</span>
            </div>
          </template>
        </div>

        <!-- === 第2行: T-1 估值日 === -->
        <div
          v-if="calc.meta.value?.t1_data?.date"
          class="base-info-row"
          style="display: flex; gap: 12px; font-size: 13px; color: var(--co-text); align-items: center; font-weight: 500; border-top: 1px dashed var(--co-border-soft); padding-top: 4px; margin-top: 4px"
        >
          <div style="width: 160px">
            <strong>【T-1 估值日】</strong>
            {{ calc.meta.value.t1_data.date.substring(5) }}
          </div>
          <n-divider vertical style="margin: 0" />
          <div style="width: 140px">
            💰 <strong>估值</strong>
            <span style="color: var(--co-accent-text); font-weight: bold; font-family: monospace">{{
              Number(calc.meta.value?.t1_data?.static_val || 0).toFixed(4)
            }}</span>
          </div>
          <n-divider vertical style="margin: 0" />
          <div style="width: 110px">
            💱 <strong>汇率</strong>
            <span style="font-family: monospace">{{
              Number(calc.meta.value?.t1_data?.exchange_rate || 0).toFixed(4)
            }}</span>
          </div>
          <n-divider vertical style="margin: 0" />
          <template v-if="calc.meta.value?.fund_config?.trade_future">
            <n-divider vertical style="margin: 0" />
            <div style="width: 160px">
              📊 <strong>{{ calc.meta.value?.fund_config?.trade_future }}校准因子</strong>
              <span style="font-family: monospace; color: var(--co-text-2)">{{
                calc.meta.value?.t1_data?.calibration
                  ? Number(calc.meta.value.t1_data.calibration).toFixed(3)
                  : '-'
              }}</span>
            </div>
          </template>
        </div>

        <!-- === 第3行: 实时数据 === -->
        <div
          class="base-info-row"
          style="display: flex; gap: 12px; font-size: 13px; color: var(--co-text); align-items: center; font-weight: 500; border-top: 1px dashed var(--co-border-soft); padding-top: 4px; margin-top: 4px"
        >
          <div style="width: 160px">📍 <strong>【实时数据】</strong></div>
          <n-divider vertical style="margin: 0" />
          <div style="width: 140px; display: flex; align-items: center; gap: 0" data-role="lof-price">
            <strong style="color: var(--co-up); width: 60px; display: inline-block">LOF价</strong>
            <input
              type="number"
              :value="calc.simLofPrice.value"
              @input="onSimLofPriceInput"
              step="0.001"
              style="width: 65px; padding: 2px 4px; font-size: 13px; font-family: monospace; border: 1px solid var(--co-border); border-radius: 4px; color: var(--co-up); font-weight: bold; text-align: center"
            />
          </div>
          <n-divider vertical style="margin: 0" />
          <div style="width: 110px">
            💱 <strong style="color: var(--co-accent-text)">汇率</strong>
            <span style="font-size: 14px; font-weight: bold; color: var(--co-accent-text); font-family: monospace">{{
              Number(calc.latestExchangeRateInput.value).toFixed(4)
            }}</span>
          </div>
          <n-divider vertical style="margin: 0" />
          <div style="flex: 1; display: flex; align-items: center; gap: 8px; min-width: 0">
            <strong style="color: var(--co-text-3)">外盘源</strong>
            <span style="font-size: 12px; color: var(--co-text-3); font-family: monospace">{{ calc.foreignSource.value }}</span>
            <span style="font-size: 11px; color: var(--co-text-3); margin-left: auto; white-space: nowrap">本地深度源: {{ calc.localDepthSource.value }}</span>
          </div>
        </div>

        <!-- === 持仓实时价列表（2026-10-07 东哥拍板）===
             取代原 T-2/T-1「ETF收盘价」+「标的实时价」三段内联文本。
             数据源：valuation_meta 的 valuation_portfolio + base_data + t1_data.etfs_info + realtime_quotes，零后端改动。 -->
        <div
          v-if="calc.holdingRows.value.length"
          style="border-top: 1px dashed var(--co-border-soft); padding-top: 6px; margin-top: 6px"
        >
          <div style="font-size: 12px; font-weight: bold; color: var(--co-text-3); margin-bottom: 4px; display: flex; align-items: baseline; gap: 10px">
            <span>📋 持仓实时价</span>
            <span style="font-weight: normal; font-size: 11px; color: var(--co-text-3)"
              >盘中涨跌 = 实时价 ÷ T-1收盘 − 1 · 估值影响 = 占比 × 盘中涨跌</span
            >
          </div>
          <table class="rt-hold-table">
            <thead>
              <tr>
                <th>序号</th>
                <th>标的</th>
                <th>市场</th>
                <th class="num">占比</th>
                <th class="num">实时价 (T)</th>
                <th class="num">盘中涨跌</th>
                <th class="num">T-1收盘</th>
                <th class="num">T-1涨跌</th>
                <th class="num">T-2收盘</th>
                <th class="num">估值影响</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(r, i) in calc.holdingRows.value" :key="r.symbol">
                <td class="idx">{{ i + 1 }}</td>
                <td class="sym">
                  {{ r.symbol }}<span v-if="r.name" class="sym-cn">{{ r.name }}</span>
                </td>
                <td class="mkt"><span class="mkt-chip" :class="r.mktClass">{{ r.market }}</span></td>
                <td class="num">{{ r.weight.toFixed(2) }}%</td>
                <td class="num rt">{{ fmtNum(r.rt) }}</td>
                <td class="num" :style="{ color: chgColor(r.rtChg), fontWeight: 'bold' }">{{ fmtSigned(r.rtChg) }}</td>
                <td class="num">{{ fmtNum(r.t1) }}</td>
                <td class="num" :style="{ color: chgColor(r.t1Chg) }">{{ fmtSigned(r.t1Chg) }}</td>
                <td class="num">{{ fmtNum(r.t2) }}</td>
                <td class="num" :style="{ color: chgColor(r.impact) }">{{ fmtSigned(r.impact, 3) }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </n-card>

    <!-- =============================================================
         现金管理债券ETF专属面板
         ============================================================= -->
    <div
      v-if="calc.isCashManagement.value && cashFundInfo"
      style="display: flex; flex-direction: column; gap: 8px; width: 100%; margin-bottom: 8px"
    >
      <!-- 基本信息卡片 -->
      <div
        style="background: linear-gradient(135deg, var(--co-surface-2) 0%, var(--co-border) 100%); padding: 12px 16px; border-radius: 8px; border: 1px solid #6ee7b7; box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05)"
      >
        <div style="display: flex; align-items: center; gap: 16px; flex-wrap: wrap">
          <div style="display: flex; align-items: center; gap: 8px">
            <span style="font-size: 15px; font-weight: bold; color: var(--co-accent-text)">{{ cashFundInfo.name }}</span>
            <n-tag type="success" size="small" round>{{ cashFundInfo.type }}</n-tag>
            <n-tag type="warning" size="small" round>风险: {{ cashFundInfo.riskLevel }}</n-tag>
          </div>
          <n-divider vertical style="margin: 0" />
          <div style="font-size: 13px; color: #374151">
            <strong>赎回门槛:</strong> {{ cashFundInfo.redemptionMin }}
          </div>
          <n-divider vertical style="margin: 0" />
          <div style="font-size: 13px; color: #374151">
            <strong>到账:</strong> {{ cashFundInfo.redemptionDays }}
          </div>
          <n-divider vertical style="margin: 0" />
          <div style="font-size: 13px; color: #374151">
            <strong>节假日:</strong> {{ cashFundInfo.holidayRule }}
          </div>
        </div>
      </div>

      <!-- 估值计算器面板 -->
      <div
        style="background: linear-gradient(135deg, var(--co-surface-2) 0%, var(--co-border) 100%); padding: 12px 16px; border-radius: 8px; border: 1px solid #93c5fd; box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05)"
      >
        <div style="display: flex; align-items: center; gap: 12px; flex-wrap: wrap">
          <div style="flex: 1; display: flex; align-items: center; gap: 8px; flex-wrap: wrap">
            <span style="font-size: 14px; font-weight: bold; color: var(--co-accent-text)">估值参数</span>
            <span style="font-size: 12px; color: var(--co-text-3)">最新净值:</span>
            <span style="font-family: monospace; font-weight: bold; color: var(--co-accent-text); font-size: 14px">{{
              calc.meta.value?.latest_nav
                ? Number(calc.meta.value.latest_nav).toFixed(4)
                : calc.meta.value?.base_data?.nav
                  ? Number(calc.meta.value.base_data.nav).toFixed(4)
                  : '-'
            }}</span>
            <span style="font-size: 12px; color: var(--co-text-3)">日均增长:</span>
            <span style="font-family: monospace; font-weight: bold; color: var(--co-down); font-size: 14px">{{
              calc.meta.value?.avg_daily_growth
                ? (calc.meta.value.avg_daily_growth * 10000).toFixed(1) + '万'
                : '-'
            }}</span>
          </div>
          <n-divider vertical style="margin: 0" />
          <div style="display: flex; align-items: center; gap: 8px">
            <span style="color: var(--co-text-2); font-size: 14px; font-weight: bold">预估净值:</span>
            <span style="font-size: 15px; font-weight: bold; color: var(--co-accent-text); font-family: monospace">{{
              calc.meta.value?.estimated_nav && calc.meta.value.estimated_nav > 0
                ? calc.meta.value.estimated_nav.toFixed(4)
                : '-'
            }}</span>
            <span style="color: var(--co-text-2); font-size: 14px; font-weight: bold">折价率:</span>
            <span
              :style="{
                fontSize: '16px',
                fontWeight: 'bold',
                color:
                  calc.simLofPrice.value > 0 && getEstNav() > 0
                    ? calc.simLofPrice.value / getEstNav() - 1 < 0
                      ? 'var(--co-up)'
                      : 'var(--co-down)'
                    : 'var(--co-neutral)',
                fontFamily: 'monospace',
                width: '70px',
                textAlign: 'left',
              }"
            >
              {{
                calc.simLofPrice.value > 0 && getEstNav() > 0
                  ? ((calc.simLofPrice.value / getEstNav() - 1) * 100).toFixed(2) + '%'
                  : '-'
              }}
            </span>
          </div>
        </div>
        <!-- 测试价说明 -->
        <div
          v-if="waterLinePrice"
          style="margin-top: 6px; font-size: 11px; color: var(--co-text-3); display: flex; gap: 16px; flex-wrap: wrap"
        >
          <span>测试价 = 预估赎回净值 - 逆回购成本（{{ waterLinePrice.redeemDays }}天）</span>
          <span>折价买入线: 场内价格 &lt; 测试价时可考虑赎回套利</span>
          <span>经验阈值: 折价 &gt; 万5 大概率盈利</span>
        </div>
      </div>

      <!-- 套利策略提示 -->
      <div
        style="background: linear-gradient(135deg, var(--co-surface-2) 0%, var(--co-border) 100%); padding: 10px 16px; border-radius: 8px; border: 1px solid #fde047"
      >
        <div style="font-size: 13px; color: var(--co-text-2); font-weight: bold; margin-bottom: 6px">套利策略参考</div>
        <div style="display: flex; gap: 24px; flex-wrap: wrap; font-size: 12px; color: var(--co-text-2)">
          <div><strong>折价赎回套利:</strong> 场内折价买入 → 赎回 → 按净值结算现金</div>
          <div><strong>日内价差:</strong> 早盘折价买入 → 收盘溢价卖出 → 赚差价 + 逆回购</div>
          <div><strong>节假日套利:</strong> 节假日前持有 → 节前卖出 → 赚取假期利息</div>
        </div>
      </div>

      <!-- 511520 BP手动输入面板 -->
      <div
        v-if="fundCode === '511520'"
        style="background: linear-gradient(135deg, var(--co-surface-2) 0%, var(--co-border) 100%); padding: 10px 16px; border-radius: 8px; border: 1px solid #7dd3fc"
      >
        <div style="font-size: 13px; color: var(--co-accent-text); font-weight: bold; margin-bottom: 6px">Choice BP手动输入</div>
        <div style="display: flex; align-items: center; gap: 8px; flex-wrap: wrap">
          <span style="font-size: 12px; color: var(--co-text-3)">7年期:</span>
          <input
            type="number"
            v-model.number="manualBp7y"
            step="0.5"
            style="width: 55px; padding: 2px 4px; font-size: 13px; font-family: monospace; border: 1px solid var(--co-border); border-radius: 4px; text-align: center"
          />
          <span style="font-size: 11px; color: var(--co-text-3)">bp</span>
          <span style="font-size: 12px; color: var(--co-text-3)">10年期:</span>
          <input
            type="number"
            v-model.number="manualBp10y"
            step="0.5"
            style="width: 55px; padding: 2px 4px; font-size: 13px; font-family: monospace; border: 1px solid var(--co-border); border-radius: 4px; text-align: center"
          />
          <span style="font-size: 11px; color: var(--co-text-3)">bp</span>
          <n-button size="tiny" type="primary" @click="submitBpOverride">应用</n-button>
          <n-button size="tiny" quaternary @click="clearBpOverride">清除</n-button>
          <span
            v-if="calc.meta.value?.bp_source === 'manual'"
            style="font-size: 11px; color: var(--co-text-2); font-weight: bold"
            >已应用 (今日有效)</span
          >
        </div>
      </div>
    </div>

    <!-- =============================================================
         估值与对冲数量推演区（非现金管理）
         ============================================================= -->
    <div
      v-if="!calc.isCashManagement.value"
      style="display: flex; flex-direction: column; gap: 8px; width: 100%; margin-bottom: 8px"
    >
      <!-- Panel 1: ETF实时估值 + 对冲数量 -->
      <div
        style="background: var(--co-surface-2); padding: 8px 14px; border-radius: 8px; border: 1px solid var(--co-border); box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05)"
      >
        <div style="display: flex; flex-direction: column; gap: 6px; width: 100%">
          <!-- Row 1 -->
          <div style="display: flex; align-items: center; gap: 12px; width: 100%">
            <div style="width: 160px; flex-shrink: 0">
              <span style="font-size: 14px; font-weight: bold; color: var(--co-accent-text)">ETF实时估值</span>
            </div>
            <div style="flex-shrink: 0; display: flex; align-items: center; gap: 2px">
              <div
                v-for="(item, index) in calc.uniqueValuationSymbols.value"
                :key="item.symbol"
                style="display: flex; align-items: center; white-space: nowrap"
              >
                <span
                  v-if="index === 0"
                  style="color: var(--co-accent-text); font-size: 14px; font-weight: bold; width: 60px; text-align: right; padding-right: 6px; flex-shrink: 0"
                  >{{ item.symbol }}价</span
                >
                <span
                  v-else
                  style="color: var(--co-accent-text); font-size: 14px; font-weight: bold; padding-right: 6px; padding-left: 12px; flex-shrink: 0"
                  >{{ item.symbol }}价</span
                >
                <input
                  type="number"
                  :value="calc.testEtfPrices[item.symbol]"
                  @input="onEtfPriceInput(item.symbol, $event)"
                  step="0.01"
                  style="width: 65px; padding: 2px 4px; font-size: 13px; font-family: monospace; border: 1px solid var(--co-border); border-radius: 4px; color: var(--co-accent-text); font-weight: bold; text-align: center; flex-shrink: 0"
                  :data-sym="item.symbol"
                />
              </div>
            </div>
            <div
              v-if="calc.isComplexCategory.value"
              style="flex: 1; min-width: 0; display: flex; align-items: center; gap: 4px; flex-wrap: nowrap"
            >
              <span style="font-size: 13px; color: var(--co-text-2); white-space: nowrap">买LOF</span>
              <input
                type="number"
                :value="calc.targetLofQty.value"
                @input="onTargetLofQtyInput"
                step="100"
                style="width: 65px; padding: 2px 4px; font-size: 13px; font-family: monospace; border: 1px solid var(--co-border); border-radius: 4px; font-weight: bold; text-align: center; color: var(--co-text-2)"
              />
              <span style="font-size: 13px; color: var(--co-text-2); white-space: nowrap">股，投入</span>
              <span
                style="font-size: 14px; color: var(--co-text-2); font-weight: bold; font-family: monospace; white-space: nowrap"
                >{{ calc.syncedCapital.value || '-' }}</span
              >
              <span style="font-size: 13px; color: var(--co-text-2); white-space: nowrap"
                >元，做空 {{ calc.meta.value?.fund_config?.trade_etf }}</span
              >
              <span
                style="font-size: 14px; color: var(--co-accent-text); font-weight: bold; font-family: monospace; white-space: nowrap"
                >{{ calc.lofQtyEtf.value ? calc.lofQtyEtf.value.etfQty : '-' }}</span
              >
              <span style="font-size: 13px; color: var(--co-text-2); white-space: nowrap">股</span>
            </div>
          </div>
          <!-- Row 2 -->
          <div style="display: flex; align-items: center; gap: 12px; width: 100%">
            <div style="width: 160px; flex-shrink: 0; display: flex; align-items: center; gap: 6px">
              <span style="color: var(--co-text-2); font-size: 14px; font-weight: bold; white-space: nowrap">估值:</span>
              <span
                style="font-size: 15px; font-weight: bold; color: var(--co-accent-text); font-family: monospace"
                >{{ calc.etfVal.value > 0 ? calc.etfVal.value.toFixed(4) : '-' }}</span
              >
            </div>
            <div
              style="width: 140px; flex-shrink: 0; display: flex; align-items: center; gap: 6px"
            >
              <span
                style="color: var(--co-text-2); font-size: 14px; font-weight: bold; white-space: nowrap; padding-left: 16px"
                >溢价:</span
              >
              <span
                :style="{
                  fontSize: '16px',
                  fontWeight: 'bold',
                  color: calc.derivedEtfPremium.value > 0 ? 'var(--co-up)' : 'var(--co-down)',
                  fontFamily: 'monospace',
                }"
              >
                {{
                  calc.etfVal.value > 0 && calc.simLofPrice.value > 0
                    ? (calc.derivedEtfPremium.value > 0 ? '+' : '') +
                      calc.derivedEtfPremium.value.toFixed(3) +
                      '%'
                    : '-'
                }}
              </span>
            </div>
            <div
              v-if="calc.isComplexCategory.value"
              style="flex: 1; min-width: 0; display: flex; align-items: center; gap: 12px; flex-wrap: nowrap"
            >
              <span style="font-size: 11px; color: var(--co-text-3); white-space: nowrap"
                >对冲值:
                <span style="color: var(--co-accent-text); font-family: monospace">{{
                  (calc.meta.value?.base_data?.hedge || 0).toFixed(4)
                }}</span></span
              >
              <span style="font-size: 11px; color: var(--co-text-3); white-space: nowrap"
                >敞口:
                <span style="color: var(--co-accent-text); font-family: monospace">{{
                  calc.lofQtyEtf.value ? calc.lofQtyEtf.value.exposure.toFixed(2) : '-'
                }}元</span></span
              >
              <span
                v-if="
                  calc.lofQtyEtf.value &&
                  calc.lofQtyEtf.value.breakdown &&
                  calc.lofQtyEtf.value.breakdown.length > 0
                "
                style="font-size: 11px; color: var(--co-accent-text); white-space: nowrap"
              >
                一篮子拆解:
                <span
                  v-for="(item, idx) in calc.lofQtyEtf.value.breakdown"
                  :key="item.symbol"
                  style="font-family: monospace; font-weight: bold"
                  >{{ item.symbol }}={{ item.qty }}股<span v-if="idx < calc.lofQtyEtf.value.breakdown.length - 1">, </span></span
                >
              </span>
            </div>
          </div>
        </div>
      </div>

      <!-- [AI-2026-08-28] 白银基金专用面板: AG0 盘口 + 参考估值 -->
      <div
        v-if="calc.isSilver.value"
        style="background: var(--co-surface-2); padding: 8px 14px; border-radius: 8px; border: 1px solid var(--co-border); box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05)"
      >
        <div style="display: flex; flex-direction: column; gap: 6px; width: 100%">
          <!-- Row 1: AG0 盘口数据 -->
          <div style="display: flex; align-items: center; gap: 12px; width: 100%">
            <div style="width: 160px; flex-shrink: 0">
              <span style="font-size: 14px; font-weight: bold; color: var(--co-down)">AG0 沪银盘口</span>
            </div>
            <div style="display: flex; align-items: center; gap: 12px; flex-wrap: wrap">
              <span style="font-size: 12px; color: var(--co-text-3)">实时价:</span>
              <span style="font-family:monospace;font-weight:bold;color:var(--co-down);font-size:14px;">
                {{ calc.meta.value?.future_quote?.price ? Number(calc.meta.value.future_quote.price).toFixed(0) : '-' }}
              </span>
              <span style="font-size: 12px; color: var(--co-text-3)">昨结算:</span>
              <span style="font-family:monospace;font-weight:bold;color:var(--co-down);font-size:14px;">
                {{ calc.meta.value?.future_quote?.settlement ? Number(calc.meta.value.future_quote.settlement).toFixed(0) : '-' }}
              </span>
              <span style="font-size: 12px; color: var(--co-text-3)">VWAP:</span>
              <span style="font-family:monospace;font-weight:bold;color:var(--co-down);font-size:14px;">
                {{ calc.meta.value?.future_quote?.vwap ? Number(calc.meta.value.future_quote.vwap).toFixed(0) : '-' }}
              </span>
              <span style="font-size: 11px; color: var(--co-text-3); margin-left: 4px">
                ({{ calc.meta.value?.future_quote?.source || '等待数据' }})
              </span>
            </div>
          </div>
          <!-- Row 2: LOF价 + 参考估值 + 溢价 -->
          <div style="display: flex; align-items: center; gap: 12px; width: 100%">
            <div style="width: 160px; flex-shrink: 0"></div>
            <div style="display: flex; align-items: center; gap: 6px">
              <span style="color: var(--co-text-2); font-size: 14px; font-weight: bold">LOF价:</span>
              <input
                type="number"
                :value="calc.simLofPrice.value"
                @input="onSimLofPriceInput"
                step="0.001"
                style="width: 70px; padding: 2px 4px; font-size: 13px; font-family: monospace; border: 1px solid var(--co-border); border-radius: 4px; color: var(--co-up); font-weight: bold; text-align: center"
              />
            </div>
            <div style="width: 140px; flex-shrink: 0; display: flex; align-items: center; gap: 6px">
              <span style="color: var(--co-text-2); font-size: 14px; font-weight: bold">参考估值:</span>
              <span style="font-size: 15px; font-weight: bold; color: var(--co-accent-text); font-family: monospace">
                {{ calc.ag0Val.value > 0 ? calc.ag0Val.value.toFixed(4) : '-' }}
              </span>
            </div>
            <div style="width: 140px; flex-shrink: 0; display: flex; align-items: center; gap: 6px">
              <span style="color: var(--co-text-2); font-size: 14px; font-weight: bold">溢价:</span>
              <span :style="{ fontSize: '16px', fontWeight: 'bold', fontFamily: 'monospace', color: calc.ag0Premium.value > 0 ? 'var(--co-up)' : 'var(--co-down)' }">
                {{ calc.ag0Premium.value > 0 ? (calc.ag0Premium.value > 0 ? '+' : '') + calc.ag0Premium.value.toFixed(3) + '%' : '-' }}
              </span>
            </div>
            <div style="flex: 1; font-size: 11px; color: var(--co-text-3)">
              公式: NAV × (AG0实时价 / AG0昨结算)
            </div>
          </div>
        </div>
      </div>

      <!-- Panel 2: 期货校准估值 + 对冲数量 -->
      <div
        v-if="calc.showFutCalib.value && calc.isComplexCategory.value"
        style="background: var(--co-surface-2); padding: 8px 14px; border-radius: 8px; border: 1px solid var(--co-border); box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05)"
      >
        <div
          style="display: flex; align-items: center; justify-content: flex-start; gap: 12px; width: 100%"
        >
          <!-- 左侧 -->
          <div style="flex: 1; display: flex; align-items: center; gap: 6px; flex-wrap: wrap">
            <span style="font-size: 14px; font-weight: bold; color: var(--co-accent-text); width: 95px"
              >期货校准估值</span
            >
            <span style="color: var(--co-accent-text); font-size: 14px; font-weight: bold"
              >{{ calc.meta.value?.fund_config?.trade_future }}:</span
            >
            <input
              type="number"
              :value="calc.testFutPrice.value"
              @input="onFutPriceInput"
              step="0.01"
              style="width: 75px; padding: 3px; font-size: 14px; font-family: monospace; border: 1px solid var(--co-border); border-radius: 4px; color: var(--co-accent-text); font-weight: bold; text-align: center"
            />
            <span style="color: var(--co-text-2); font-size: 13px; font-weight: bold; margin-left: 4px"
              >校准:</span
            >
            <input
              type="number"
              :value="calc.testFutCalib.value"
              @input="onFutCalibInput"
              step="0.0001"
              style="width: 60px; padding: 3px; font-size: 13px; font-family: monospace; border: 1px solid var(--co-border); border-radius: 4px; text-align: center"
            />
          </div>

          <n-divider vertical style="margin: 0" />

          <!-- 中间 估值 & 溢价 -->
          <div
            style="width: 220px; display: flex; align-items: center; gap: 6px; justify-content: center"
          >
            <span style="color: var(--co-text-2); font-size: 14px; font-weight: bold">估值:</span>
            <span
              style="font-size: 15px; font-weight: bold; color: var(--co-accent-text); font-family: monospace; width: 65px; text-align: left"
              >{{ calc.futCalibVal.value > 0 ? calc.futCalibVal.value.toFixed(4) : '-' }}</span
            >
            <span style="color: var(--co-text-2); font-size: 14px; font-weight: bold">溢价:</span>
            <span
              :style="{
                fontSize: '16px',
                fontWeight: 'bold',
                color: calc.derivedFutPremium.value > 0 ? 'var(--co-up)' : 'var(--co-down)',
                fontFamily: 'monospace',
                width: '60px',
                textAlign: 'left',
              }"
            >
              {{
                calc.futCalibVal.value > 0 && calc.simLofPrice.value > 0
                  ? (calc.derivedFutPremium.value > 0 ? '+' : '') +
                    calc.derivedFutPremium.value.toFixed(3) +
                      '%'
                  : '-'
              }}
            </span>
          </div>

          <n-divider vertical style="margin: 0" />

          <!-- 右侧 交易 -->
          <div style="flex: 1.2; display: flex; align-items: center; gap: 4px; flex-wrap: wrap">
            <span style="font-size: 13px; color: var(--co-text-2)">交易</span>
            <input
              type="number"
              :value="calc.targetLotsFuture.value"
              @input="onTargetLotsFutureInput"
              step="1"
              style="width: 65px; padding: 2px 4px; font-size: 13px; font-family: monospace; border: 1px solid var(--co-border); border-radius: 4px; font-weight: bold; text-align: center; color: var(--co-text-2)"
            />
            <span style="font-size: 13px; color: var(--co-text-2)">手期货 → 对应LOF</span>
            <span
              style="font-size: 14px; color: var(--co-up); font-weight: bold; font-family: monospace"
              >{{ calc.lofQtyFuture.value ? calc.lofQtyFuture.value.lofQty : '-' }}</span
            >
            <span style="font-size: 13px; color: var(--co-text-2)">股</span>
          </div>
        </div>
        <div
          style="display: flex; justify-content: center; gap: 24px; font-size: 11px; color: var(--co-text-3); margin-top: 3px"
        >
          <span
            >对冲值:
            <span style="color: var(--co-accent-text); font-family: monospace">{{
              calc.lofQtyFuture.value ? calc.lofQtyFuture.value.hedgeValue.toFixed(4) : '-'
            }}</span></span
          >
          <span
            >敞口:
            <span style="color: var(--co-accent-text); font-family: monospace">{{
              calc.lofQtyFuture.value ? calc.lofQtyFuture.value.exposure.toFixed(2) : '-'
            }}元</span></span
          >
          <span
            >校准ETF:
            <span style="color: var(--co-accent-text); font-family: monospace">{{
              calc.equivEtfPrice.value > 0 ? calc.equivEtfPrice.value.toFixed(3) : '-'
            }}</span></span
          >
        </div>
      </div>

      <!-- Panel 3: 纯期货估值 + 对冲数量 -->
      <div
        v-if="calc.showPureFut.value && calc.isComplexCategory.value"
        style="background: var(--co-surface-2); padding: 8px 14px; border-radius: 8px; border: 1px solid var(--co-border); box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05)"
      >
        <div
          style="display: flex; align-items: center; justify-content: flex-start; gap: 12px; width: 100%"
        >
          <div style="flex: 1; display: flex; align-items: center; gap: 6px; flex-wrap: wrap">
            <span style="font-size: 14px; font-weight: bold; color: var(--co-accent-text); width: 95px"
              >纯期货估值</span
            >
            <span style="color: var(--co-accent-text); font-size: 14px; font-weight: bold"
              >{{ calc.meta.value?.fund_config?.trade_future }}:</span
            >
            <input
              type="number"
              :value="calc.testFutPrice.value"
              @input="onFutPriceInput"
              step="0.01"
              style="width: 75px; padding: 3px; font-size: 14px; font-family: monospace; border: 1px solid var(--co-border); border-radius: 4px; color: var(--co-accent-text); font-weight: bold; text-align: center"
            />
          </div>

          <n-divider vertical style="margin: 0" />

          <div
            style="width: 220px; display: flex; align-items: center; gap: 6px; justify-content: center"
          >
            <span style="color: var(--co-text-2); font-size: 14px; font-weight: bold">估值:</span>
            <span
              style="font-size: 15px; font-weight: bold; color: var(--co-text); font-family: monospace; width: 65px; text-align: left"
              >{{
                calc.pureFutVal.value > 0 ? calc.pureFutVal.value.toFixed(4) : '-'
              }}</span
            >
            <span style="color: var(--co-text-2); font-size: 14px; font-weight: bold">溢价:</span>
            <span
              :style="{
                fontSize: '16px',
                fontWeight: 'bold',
                color: calc.derivedPureFutPremium.value > 0 ? 'var(--co-up)' : 'var(--co-down)',
                fontFamily: 'monospace',
                width: '60px',
                textAlign: 'left',
              }"
            >
              {{
                calc.pureFutVal.value > 0 && calc.simLofPrice.value > 0
                  ? (calc.derivedPureFutPremium.value > 0 ? '+' : '') +
                    calc.derivedPureFutPremium.value.toFixed(3) +
                      '%'
                  : '-'
              }}
            </span>
          </div>

          <n-divider vertical style="margin: 0" />

          <div style="flex: 1.2; display: flex; align-items: center; gap: 4px; flex-wrap: wrap">
            <span style="font-size: 13px; color: var(--co-text-2)">交易</span>
            <input
              type="number"
              :value="calc.targetLotsPureFuture.value"
              @input="onTargetLotsPureFutureInput"
              step="1"
              style="width: 65px; padding: 2px 4px; font-size: 13px; font-family: monospace; border: 1px solid var(--co-border); border-radius: 4px; font-weight: bold; text-align: center; color: var(--co-text-2)"
            />
            <span style="font-size: 13px; color: var(--co-text-2)">手期货 → 对应LOF</span>
            <span
              style="font-size: 14px; color: var(--co-up); font-weight: bold; font-family: monospace"
              >{{
                calc.lofQtyPureFuture.value ? calc.lofQtyPureFuture.value.lofQty : '-'
              }}</span
            >
            <span style="font-size: 13px; color: var(--co-text-2)">股</span>
          </div>
        </div>
        <div
          style="display: flex; justify-content: center; gap: 24px; font-size: 11px; color: var(--co-text-3); margin-top: 3px"
        >
          <span
            >对冲值:
            <span style="color: var(--co-accent-text); font-family: monospace">{{
              calc.lofQtyPureFuture.value
                ? calc.lofQtyPureFuture.value.hedgeValue.toFixed(4)
                : '-'
            }}</span></span
          >
          <span
            >敞口:
            <span style="color: var(--co-accent-text); font-family: monospace">{{
              calc.lofQtyPureFuture.value
                ? calc.lofQtyPureFuture.value.exposure.toFixed(2)
                : '-'
            }}元</span></span
          >
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
/**
 * ValuationCalculator - 共享实时估值计算器组件
 *
 * 封装 T-2/T-1/实时数据行 + ETF估值 + 期货校准 + 纯期货面板。
 * 通过 defineExpose 暴露内部 composable 状态，供父页面通过模板 ref 访问。
 *
 * 用法：
 *   <ValuationCalculator ref="vcRef" :fund-code="fundCode" />
 *   父页面通过 vcRef?.simLofPrice, vcRef?.etfVal 等访问状态
 */
import { ref, computed, watch } from 'vue'
import { NCard, NDivider, NTag, NButton } from 'naive-ui'
import { useValuationCalculator } from '../composables/useValuationCalculator'
import { postBpOverride, clearBpOverride as clearBpOverrideApi } from '../api'

const props = defineProps<{
  fundCode: string
}>()

const calc = useValuationCalculator()

// 同步 fundCode 到 composable
watch(
  () => props.fundCode,
  (code) => {
    if (code && code !== calc.fundCode.value) {
      calc.fundCode.value = code
      calc.resetInitialized()
    }
  },
  { immediate: true },
)

// ============================================================
// 持仓实时价表格式化辅助
// ============================================================
const fmtNum = (v: number | null, digits = 2) => (v === null ? '-' : v.toFixed(digits))
const fmtSigned = (v: number | null, digits = 2) =>
  v === null ? '-' : (v > 0 ? '+' : '') + v.toFixed(digits) + '%'
// A股惯例：涨红跌绿；0/无数据显示中性灰
const chgColor = (v: number | null) =>
  v === null || v === 0 ? 'var(--co-neutral)' : v > 0 ? 'var(--co-up)' : 'var(--co-down)'

// ============================================================
// 输入事件转发（使父页面可通过 defineExpose 访问最新值）
// ============================================================
const onSimLofPriceInput = (e: Event) => {
  const val = parseFloat((e.target as HTMLInputElement).value)
  if (!isNaN(val)) calc.simLofPrice.value = val
}

const onEtfPriceInput = (sym: string, e: Event) => {
  const val = parseFloat((e.target as HTMLInputElement).value)
  if (!isNaN(val)) calc.testEtfPrices[sym] = val
}

const onFutPriceInput = (e: Event) => {
  const val = parseFloat((e.target as HTMLInputElement).value)
  if (!isNaN(val)) calc.testFutPrice.value = val
}

const onFutCalibInput = (e: Event) => {
  const val = parseFloat((e.target as HTMLInputElement).value)
  if (!isNaN(val)) calc.testFutCalib.value = val
}

const onTargetLofQtyInput = (e: Event) => {
  const val = parseInt((e.target as HTMLInputElement).value, 10)
  if (!isNaN(val) && val >= 0) calc.targetLofQty.value = val
}

const onTargetLotsFutureInput = (e: Event) => {
  const val = parseInt((e.target as HTMLInputElement).value, 10)
  if (!isNaN(val) && val >= 0) calc.targetLotsFuture.value = val
}

const onTargetLotsPureFutureInput = (e: Event) => {
  const val = parseInt((e.target as HTMLInputElement).value, 10)
  if (!isNaN(val) && val >= 0) calc.targetLotsPureFuture.value = val
}

// ============================================================
// 现金管理（与 LazyMode 保持一致的完整版本）
// ============================================================
const manualBp7y = ref(0)
const manualBp10y = ref(0)

const cashFundInfo = computed(() => {
  const code = props.fundCode
  if (!code) return null
  const infoMap: Record<
    string,
    { name: string; type: string; redemptionMin: string; redemptionDays: string; holidayRule: string; riskLevel: string }
  > = {
    '511880': {
      name: '银华日利ETF',
      type: '货币基金',
      redemptionMin: '1份',
      redemptionDays: 'T+1盘中到账（银河）',
      holidayRule: '节前最后一天结算假期收益',
      riskLevel: '极低',
    },
    '511360': {
      name: '短融ETF',
      type: '短期融资券ETF',
      redemptionMin: '2000份（约22万）',
      redemptionDays: 'T+2盘中到账（银河）',
      holidayRule: '节后第一天更新净值',
      riskLevel: '中低',
    },
    '511520': {
      name: '政金债ETF',
      type: '中长期政金债ETF',
      redemptionMin: '10000份（约1.14万）',
      redemptionDays: 'T+2 14:30后到账',
      holidayRule: '-',
      riskLevel: '中',
    },
  }
  return infoMap[code] || null
})

const getEstNav = () => {
  if (calc.isCashManagement.value) {
    return calc.meta.value?.estimated_nav || 0
  }
  return calc.meta.value?.rt_val || 0
}

const waterLinePrice = computed(() => {
  if (!calc.meta.value?.base_data) return null
  const bd = calc.meta.value.base_data
  const nav = parseFloat(bd.nav) || 0
  const avgGrowth = calc.meta.value?.avg_daily_growth || 0

  if (nav <= 0 || avgGrowth === 0) return null

  const estimatedRedeemNav = nav + avgGrowth
  const repoCost = 0.02
  const dailyRepoCost = repoCost / 252
  const redeemDays = props.fundCode === '511880' ? 1 : 2
  const totalRepoCost = dailyRepoCost * redeemDays
  const waterLine = estimatedRedeemNav * (1 - totalRepoCost)

  return {
    estimatedRedeemNav: Math.round(estimatedRedeemNav * 10000) / 10000,
    waterLine: Math.round(waterLine * 10000) / 10000,
    avgDailyGrowth: avgGrowth,
    treasuryPct: calc.meta.value?.treasury_index_pct || 0,
    repoCost: (totalRepoCost * 10000).toFixed(1) + '万',
    redeemDays,
  }
})

const submitBpOverride = async () => {
  try {
    await postBpOverride(props.fundCode, manualBp7y.value, manualBp10y.value)
    calc.fetchValuationMeta()
  } catch {
    /* ignore */
  }
}

const clearBpOverride = async () => {
  try {
    await clearBpOverrideApi(props.fundCode)
    manualBp7y.value = 0
    manualBp10y.value = 0
    calc.fetchValuationMeta()
  } catch {
    /* ignore */
  }
}

// ============================================================
// 暴露内部状态给父页面（通过模板 ref 访问）
// ============================================================
defineExpose({
  // 直接透传 composable 的所有导出
  ...calc,
  // 额外的现金管理状态
  cashFundInfo,
  getEstNav,
  waterLinePrice,
  manualBp7y,
  manualBp10y,
})
</script>

<style scoped>
/* 持仓实时价表：紧凑 12px，数字右对齐等宽，与静态弹窗「估值明细」同视觉语言 */
.rt-hold-table {
  width: 100%;
  border-collapse: collapse;
  font-size: var(--co-fs-sm);
  font-family: var(--co-font-sans);
}
.rt-hold-table th,
.rt-hold-table td {
  padding: 3px 8px;
  border-bottom: 1px solid var(--co-border-soft);
  white-space: nowrap;
}
.rt-hold-table thead th {
  background: var(--co-accent-soft);
  color: var(--co-accent-text);
  font-weight: 600;
  text-align: right;
}
.rt-hold-table thead th:first-child,
.rt-hold-table thead th:nth-child(2),
.rt-hold-table thead th:nth-child(3) {
  text-align: left;
}
.rt-hold-table td.num {
  font-family: var(--co-font-mono);
  text-align: right;
  color: var(--co-text);
}
.rt-hold-table td.sym {
  font-weight: 700;
  color: var(--co-text);
  text-align: left;
}
.rt-hold-table td.idx {
  color: var(--co-text-3);
  font-family: var(--co-font-mono);
  text-align: left;
  width: 34px;
}
.rt-hold-table td.sym .sym-cn {
  font-weight: normal;
  font-size: 11px;
  color: var(--co-text-3);
  margin-left: 5px;
}
.rt-hold-table td.mkt {
  text-align: left;
}
.rt-hold-table .mkt-chip {
  display: inline-block;
  font-size: 10px;
  line-height: 16px;
  padding: 0 7px;
  border-radius: 8px;
}
/* 市场分类配色（淡雅同色系：浅底 + 深字，纯自起色板，互不撞色） */
.rt-hold-table .mkt-chip.mkt-us    { background: #e8f0fb; color: #2f5d9e; }  /* 美股 · 蓝 */
.rt-hold-table .mkt-chip.mkt-hk    { background: #e3f4f1; color: #1f7a6e; }  /* 香港 · 青 */
.rt-hold-table .mkt-chip.mkt-jp    { background: #fbe9ee; color: #9e3b59; }  /* 日本 · 玫 */
.rt-hold-table .mkt-chip.mkt-eu    { background: #eee9f8; color: #5b3f9e; }  /* 欧洲 · 紫 */
.rt-hold-table .mkt-chip.mkt-cn    { background: #f6f0e0; color: #8a6d1f; }  /* A股 · 琥 */
.rt-hold-table .mkt-chip.mkt-ft    { background: #eceff2; color: #4a5560; }  /* 期货 · 灰 */
.rt-hold-table .mkt-chip.mkt-other { background: var(--co-accent-soft); color: var(--co-accent-text); }
.rt-hold-table td.rt {
  font-weight: bold;
  color: var(--co-accent-text);
}
.rt-hold-table tbody tr:hover {
  background: var(--co-surface-2);
}
</style>

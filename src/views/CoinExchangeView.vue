<template lang="pug">
  .coins-container
    .ui.container
      h2.page-title {{ sify('兌幣練習') }}

      .split-zones
        //- 左側放置區
        .zone-wrapper
          .zone-label {{ sify('左側') }}
          .drop-zone(
            @dragover.prevent
            @drop="onDrop($event, 'left')"
            @dragenter.prevent
            @dragleave.prevent
            ref="dropZoneLeft"
          )
            .placed-coins
              template(v-for="(coin, index) in leftCoins" :key="coin.id + '-L-' + index")
                .placed-coin(
                  :draggable="true"
                  @dragstart="onPlacedCoinDragStart($event, index, 'left')"
                  @touchstart="onTouchStart($event, coin, 'placed', index, 'left')"
                  :class="{ 'is-rect': coin.shape === 'rect' }"
                  :style="coinStyle(coin)"
                )
                  span.coin-value ${{ coin.value }}
            .drop-zone-text(v-if="leftCoins.length === 0")
              p {{ sify('將硬幣拖曳到此處') }}
          .zone-total {{ sify('總額') }}：${{ leftTotal }}

        .zone-divider +

        //- 右側放置區（自動兌幣）
        .zone-wrapper
          .zone-label {{ sify('右側（自動兌幣）') }}
          .drop-zone(
            @dragover.prevent
            @drop="onDrop($event, 'right')"
            @dragenter.prevent
            @dragleave.prevent
            ref="dropZoneRight"
          )
            .placed-coins
              template(v-for="(coin, index) in rightCoins" :key="coin.id + '-R-' + index")
                .placed-coin(
                  :draggable="!coin.merging"
                  @dragstart="onPlacedCoinDragStart($event, index, 'right')"
                  @touchstart="onTouchStart($event, coin, 'placed', index, 'right')"
                  :class="{ 'is-rect': coin.shape === 'rect', 'is-merging': coin.merging, 'is-appearing': coin.appearing }"
                  :style="coinStyle(coin)"
                )
                  span.coin-value ${{ coin.value }}
            .drop-zone-text(v-if="rightCoins.length === 0")
              p {{ sify('將硬幣拖曳到此處') }}
          .zone-total {{ sify('總額') }}：${{ rightTotal }}

      //- 兌換紀錄
      .exchange-log(v-if="exchangeLog.length > 0")
        transition-group(name="log-fade" tag="div")
          .log-item(v-for="(entry, i) in exchangeLog" :key="entry.id")
            span.log-arrow ⇒
            | {{ entry.text }}

      .equation-box(v-if="showEquation")
        span.eq-text ${{ leftTotal }} + ${{ rightTotal }} = ${{ leftTotal + rightTotal }}

      .ui.buttons
        button.ui.button.teal(@click="toggleEquation")
          span(v-if="!showEquation") {{ sify('合計') }}
          span(v-else) {{ sify('隱藏總合') }}
        button.ui.button.primary(@click="clearAll") {{ sify('清除全部') }}

      .coins-toolbar(
        @dragover.prevent
        @drop="onToolbarDrop($event)"
      )
        .coin-item(
          v-for="coin in availableCoins"
          :key="coin.id"
          :draggable="true"
          @dragstart="onDragStart($event, coin)"
          @touchstart="onTouchStart($event, coin, 'toolbar')"
          :class="{ 'is-rect': coin.shape === 'rect' }"
          :style="toolbarCoinStyle(coin)"
        )
          span.coin-value ${{ coin.value }}

      //- 浮動硬幣（觸控拖曳時顯示）
      .placed-coin(
        v-if="touchDragging && touchDragCoin && touchDragCoin.x !== null && touchDragCoin.y !== null"
        :class="{ 'is-rect': touchDragCoin && touchDragCoin.shape === 'rect' }"
        :style="touchFloatStyle"
      )
        span.coin-value ${{ touchDragCoin && touchDragCoin.value }}
</template>

<script>
import { sify } from 'chinese-conv'

// 兌換規則：from（幾元）× count 個 → to（幾元）1個
// 由大到小排列，讓高額優先兌換（cascade 用）
const EXCHANGE_RULES = [
  { from: 500, count: 2, to: 1000 },
  { from: 100, count: 5, to: 500 },
  { from: 50,  count: 2, to: 100 },
  { from: 10,  count: 5, to: 50  },
  { from: 5,   count: 2, to: 10  },
  { from: 1,   count: 5, to: 5   },
]

// 各面額的外觀定義
const COIN_DEFS = {
  1:    { value: 1,    color: '#D0D0D0', size: 40,  shape: 'circle' },
  5:    { value: 5,    color: '#C0C0C0', size: 45,  shape: 'circle' },
  10:   { value: 10,   color: '#B8B8B8', size: 50,  shape: 'circle' },
  50:   { value: 50,   color: '#CD7F32', size: 55,  shape: 'circle' },
  100:  { value: 100,  color: '#85C1E9', width: 80,  height: 44, shape: 'rect' },
  500:  { value: 500,  color: '#82E0AA', width: 90,  height: 50, shape: 'rect' },
  1000: { value: 1000, color: '#F9E79F', width: 100, height: 55, shape: 'rect' },
}

export default {
  name: 'CoinExchangeView',
  props: ['si'],
  data() {
    return {
      availableCoins: [
        { id: 'coin-1',    ...COIN_DEFS[1]    },
        { id: 'coin-5',    ...COIN_DEFS[5]    },
        { id: 'coin-10',   ...COIN_DEFS[10]   },
        { id: 'coin-50',   ...COIN_DEFS[50]   },
        { id: 'bill-100',  ...COIN_DEFS[100]  },
        { id: 'bill-500',  ...COIN_DEFS[500]  },
        { id: 'bill-1000', ...COIN_DEFS[1000] },
      ],
      leftCoins: [],
      rightCoins: [],
      draggedCoin: null,
      draggedIndex: null,
      draggedSide: null,
      showEquation: false,
      exchangeLog: [],        // 兌換紀錄列表
      isExchanging: false,    // 防止動畫期間重複觸發
      logIdCounter: 0,
      // 觸控專用
      touchDragging: false,
      touchDragCoin: null,
      touchDragIndex: null,
      touchDragSide: null,
      touchOffset: { x: 0, y: 0 },
      originalCoin: null,
    }
  },
  computed: {
    leftTotal() {
      return this.leftCoins.reduce((sum, c) => sum + c.value, 0)
    },
    rightTotal() {
      return this.rightCoins.reduce((sum, c) => sum + c.value, 0)
    },
    touchFloatStyle() {
      if (!this.touchDragCoin) return {}
      const coin = this.touchDragCoin
      const base = {
        left: coin.x + 'px',
        top: coin.y + 'px',
        backgroundColor: coin.color,
        position: 'fixed',
        zIndex: 9999,
        pointerEvents: 'none',
      }
      if (coin.shape === 'rect') {
        return { ...base, width: (coin.width || 80) + 'px', height: (coin.height || 44) + 'px' }
      }
      return { ...base, width: (coin.size || 50) + 'px', height: (coin.size || 50) + 'px' }
    }
  },
  methods: {
    sify(t) {
      return this.si ? sify(t) : t
    },

    // --- 樣式輔助 ---
    coinStyle(coin) {
      const base = { left: coin.x + 'px', top: coin.y + 'px', backgroundColor: coin.color }
      if (coin.shape === 'rect') {
        return { ...base, width: (coin.width || 80) + 'px', height: (coin.height || 44) + 'px' }
      }
      return { ...base, width: (coin.size || 50) + 'px', height: (coin.size || 50) + 'px' }
    },
    toolbarCoinStyle(coin) {
      const base = { backgroundColor: coin.color }
      if (coin.shape === 'rect') {
        return { ...base, width: (coin.width || 80) + 'px', height: (coin.height || 44) + 'px' }
      }
      return { ...base, width: (coin.size || 50) + 'px', height: (coin.size || 50) + 'px' }
    },
    coinW(coin) { return coin.shape === 'rect' ? (coin.width  || 80) : (coin.size || 50) },
    coinH(coin) { return coin.shape === 'rect' ? (coin.height || 44) : (coin.size || 50) },

    toggleEquation() {
      this.showEquation = !this.showEquation
    },

    // --- 滑鼠拖曳：從工具列 ---
    onDragStart(event, coin) {
      this.draggedCoin = { ...coin, id: coin.id + '-' + Date.now() }
      this.draggedIndex = null
      this.draggedSide = null
      event.dataTransfer.effectAllowed = 'copy'
      event.dataTransfer.setData('text/plain', coin.id)
      event.stopPropagation()
    },

    // --- 滑鼠拖曳：從排列區 ---
    onPlacedCoinDragStart(event, index, side) {
      this.draggedIndex = index
      this.draggedSide = side
      const arr = side === 'left' ? this.leftCoins : this.rightCoins
      this.draggedCoin = { ...arr[index] }
      event.dataTransfer.effectAllowed = 'move'
      event.dataTransfer.setData('text/plain', 'placed-' + side + '-' + index)
      event.stopPropagation()
    },

    onDrop(event, targetSide) {
      event.preventDefault()
      event.stopPropagation()
      if (!this.draggedCoin) return

      const refKey = targetSide === 'left' ? 'dropZoneLeft' : 'dropZoneRight'
      const rect = this.$refs[refKey].getBoundingClientRect()
      const w = this.coinW(this.draggedCoin)
      const h = this.coinH(this.draggedCoin)
      const x = event.clientX - rect.left - w / 2
      const y = event.clientY - rect.top  - h / 2

      // 若從排列區拖來，先移除原位置
      if (this.draggedIndex !== null && this.draggedSide !== null) {
        const srcArr = this.draggedSide === 'left' ? this.leftCoins : this.rightCoins
        srcArr.splice(this.draggedIndex, 1)
        this.draggedIndex = null
        this.draggedSide = null
      }

      const targetArr = targetSide === 'left' ? this.leftCoins : this.rightCoins
      targetArr.push({
        ...this.draggedCoin,
        x: Math.max(0, Math.min(x, rect.width  - w)),
        y: Math.max(0, Math.min(y, rect.height - h)),
      })
      this.draggedCoin = null

      // 右側才觸發兌幣
      if (targetSide === 'right') {
        this.$nextTick(() => this.tryExchange())
      }
    },

    onToolbarDrop(event) {
      event.preventDefault()
      event.stopPropagation()
      if (this.draggedIndex !== null && this.draggedSide !== null) {
        const arr = this.draggedSide === 'left' ? this.leftCoins : this.rightCoins
        arr.splice(this.draggedIndex, 1)
        this.$forceUpdate()
      }
      this.draggedCoin = null
      this.draggedIndex = null
      this.draggedSide = null
    },

    // --- 兌幣核心邏輯（含動畫） ---
    async tryExchange() {
      if (this.isExchanging) return
      this.isExchanging = true

      let exchanged = true
      while (exchanged) {
        exchanged = false
        for (const rule of EXCHANGE_RULES) {
          const candidates = this.rightCoins.filter(c => c.value === rule.from && !c.merging && !c.appearing)
          if (candidates.length >= rule.count) {
            exchanged = true
            const toMerge = candidates.slice(0, rule.count)

            // 1. 標記為合併中 → 觸發浮起動畫
            toMerge.forEach(c => { c.merging = true })
            this.$forceUpdate()
            await this._wait(420)

            // 2. 移除被合併的硬幣
            toMerge.forEach(c => {
              const idx = this.rightCoins.indexOf(c)
              if (idx !== -1) this.rightCoins.splice(idx, 1)
            })

            // 3. 放入新硬幣（appearing 動畫）
            const def = COIN_DEFS[rule.to]
            const anchor = toMerge[0]  // 新幣出現在第一枚位置附近
            const newCoin = {
              ...def,
              id: `coin-${rule.to}-${Date.now()}`,
              x: anchor.x,
              y: anchor.y,
              appearing: true,
              merging: false,
            }
            this.rightCoins.push(newCoin)
            this.$forceUpdate()
            await this._wait(380)
            newCoin.appearing = false
            this.$forceUpdate()

            // 4. 記錄兌換紀錄
            this.addLog(`${rule.count} × $${rule.from} → 1 × $${rule.to}`)
            break  // 重新從頭檢查（cascade）
          }
        }
      }

      this.isExchanging = false
    },

    _wait(ms) {
      return new Promise(resolve => setTimeout(resolve, ms))
    },

    addLog(text) {
      this.logIdCounter++
      this.exchangeLog.unshift({ id: this.logIdCounter, text })
      // 只保留最近 5 筆
      if (this.exchangeLog.length > 5) this.exchangeLog.pop()
    },

    // --- 觸控拖曳 ---
    onTouchStart(e, coin, type, index, side) {
      if (e.touches.length !== 1) return
      e.preventDefault()
      this.touchDragging = true
      if (type === 'toolbar') {
        this.touchDragCoin = { ...coin, id: coin.id + '-' + Date.now() }
        this.touchDragIndex = null
        this.touchDragSide = null
      } else {
        this.touchDragCoin = { ...coin }
        this.touchDragIndex = index
        this.touchDragSide = side
      }
      const touch = e.touches[0]
      const rect = e.target.getBoundingClientRect()
      this.touchOffset = {
        x: touch.clientX - (rect.left + rect.width  / 2),
        y: touch.clientY - (rect.top  + rect.height / 2),
      }
      this.originalCoin = coin
      this.touchDragCoin.x = null
      this.touchDragCoin.y = null
      window.addEventListener('touchmove', this.onTouchMove, { passive: false })
      window.addEventListener('touchend',  this.onTouchEnd,  { passive: false })
    },

    onTouchMove(e) {
      if (!this.touchDragging || !this.touchDragCoin) return
      e.preventDefault()
      const touch = e.touches[0]
      const w = this.coinW(this.touchDragCoin)
      const h = this.coinH(this.touchDragCoin)
      this.touchDragCoin.x = touch.clientX - this.touchOffset.x - w / 2
      this.touchDragCoin.y = touch.clientY - this.touchOffset.y - h / 2
      this.$forceUpdate()
    },

    onTouchEnd(e) {
      if (!this.touchDragging || !this.touchDragCoin) return
      const touch = (e.changedTouches && e.changedTouches[0]) || (e.touches && e.touches[0])

      // 拖回工具列 → 刪除
      const toolbar = this.$el.querySelector('.coins-toolbar')
      const toolbarRect = toolbar.getBoundingClientRect()
      if (
        touch.clientX >= toolbarRect.left && touch.clientX <= toolbarRect.right &&
        touch.clientY >= toolbarRect.top  && touch.clientY <= toolbarRect.bottom
      ) {
        if (this.touchDragIndex !== null && this.touchDragSide !== null) {
          const arr = this.touchDragSide === 'left' ? this.leftCoins : this.rightCoins
          arr.splice(this.touchDragIndex, 1)
          this.$forceUpdate()
        }
      } else {
        // 判斷落在哪個 drop zone
        const zones = [
          { ref: this.$refs.dropZoneLeft,  side: 'left'  },
          { ref: this.$refs.dropZoneRight, side: 'right' },
        ]
        for (const zone of zones) {
          const rect = zone.ref.getBoundingClientRect()
          if (
            touch.clientX >= rect.left && touch.clientX <= rect.right &&
            touch.clientY >= rect.top  && touch.clientY <= rect.bottom
          ) {
            const coin = this.touchDragCoin
            const w = this.coinW(coin)
            const h = this.coinH(coin)
            const x = touch.clientX - rect.left - w / 2
            const y = touch.clientY - rect.top  - h / 2
            const finalCoin = {
              ...coin,
              x: Math.max(0, Math.min(x, rect.width  - w)),
              y: Math.max(0, Math.min(y, rect.height - h)),
            }

            if (this.touchDragIndex !== null && this.touchDragSide !== null) {
              const srcArr = this.touchDragSide === 'left' ? this.leftCoins : this.rightCoins
              srcArr.splice(this.touchDragIndex, 1)
            }
            const targetArr = zone.side === 'left' ? this.leftCoins : this.rightCoins
            targetArr.push(finalCoin)

            if (zone.side === 'right') {
              this.$nextTick(() => this.tryExchange())
            }
            break
          }
        }
      }

      this.touchDragging = false
      this.touchDragCoin = null
      this.touchDragIndex = null
      this.touchDragSide = null
      this.originalCoin = null
      window.removeEventListener('touchmove', this.onTouchMove)
      window.removeEventListener('touchend',  this.onTouchEnd)
      this.$forceUpdate()
    },

    clearAll() {
      this.leftCoins = []
      this.rightCoins = []
      this.showEquation = false
      this.exchangeLog = []
    },
  },

  mounted() {
    this.onTouchMove = this.onTouchMove.bind(this)
    this.onTouchEnd  = this.onTouchEnd.bind(this)
  },
}
</script>

<style scoped>
.coins-container {
  padding: 10px;
  max-width: 1200px;
  margin: 0 auto;
}

.page-title {
  text-align: center;
  font-size: 1.6rem;
  font-weight: bold;
  color: #1a237e;
  margin-bottom: 16px;
}

/* ===== 工具列 ===== */
.coins-toolbar {
  display: flex;
  justify-content: center;
  gap: 16px;
  margin-top: 30px;
  flex-wrap: wrap;
  align-items: center;
}

.coin-item {
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: grab;
  user-select: none;
  box-shadow: 0 4px 8px rgba(0,0,0,0.2);
  transition: transform 0.2s ease;
  border: 2px solid #555;
  -webkit-user-drag: element;
  pointer-events: auto;
}
.coin-item.is-rect { border-radius: 6px; }
.coin-item:hover  { transform: scale(1.1); }
.coin-item:active { cursor: grabbing; transform: scale(0.95); }

/* ===== 硬幣文字 ===== */
.coin-value {
  font-weight: bold;
  color: #333;
  font-size: 11px;
  pointer-events: none;
  user-select: none;
}

/* ===== 左右分割區 ===== */
.split-zones {
  display: flex;
  align-items: flex-start;
  gap: 0;
  margin: 20px 0;
}

.zone-wrapper {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: stretch;
}

.zone-label {
  text-align: center;
  font-size: 16px;
  font-weight: bold;
  color: #333;
  padding: 8px 4px;
  background: #f0f4ff;
  border-radius: 8px 8px 0 0;
  border: 2px solid #ccc;
  border-bottom: none;
}

.zone-divider {
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 2.5rem;
  font-weight: bold;
  color: #888;
  padding: 0 10px;
  margin-top: 50px;
}

.drop-zone {
  min-height: 260px;
  border: 3px dashed #ccc;
  position: relative;
  background-color: #f9f9f9;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: border-color 0.3s ease;
  overflow: hidden;
  pointer-events: auto;
}
.drop-zone:hover { border-color: #007bff; }

.drop-zone-text {
  text-align: center;
  color: #aaa;
  font-size: 14px;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  pointer-events: none;
}

.placed-coins {
  position: relative;
  width: 100%;
  height: 100%;
  min-height: 260px;
}

.zone-total {
  text-align: center;
  font-size: 1.3rem;
  font-weight: bold;
  color: #1a237e;
  background: #e8eaf6;
  border: 2px solid #ccc;
  border-top: none;
  border-radius: 0 0 8px 8px;
  padding: 10px 0;
}

/* ===== 放置後的硬幣 ===== */
.placed-coin {
  position: absolute;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: grab;
  user-select: none;
  box-shadow: 0 4px 8px rgba(0,0,0,0.3);
  border: 2px solid #555;
  z-index: 10;
  pointer-events: auto;
  transition: transform 0.2s ease;
}
.placed-coin.is-rect  { border-radius: 6px; }
.placed-coin:hover    { transform: scale(1.1); z-index: 20; }
.placed-coin:active   { cursor: grabbing; transform: scale(0.95); }

/* ===== 兌幣動畫 ===== */

/* 向上浮起並縮小消失 */
.placed-coin.is-merging {
  animation: mergeUp 0.4s ease-in forwards;
  pointer-events: none;
  z-index: 50;
}

@keyframes mergeUp {
  0%   { opacity: 1;   transform: translateY(0)    scale(1);    }
  60%  { opacity: 0.8; transform: translateY(-30px) scale(1.1); }
  100% { opacity: 0;   transform: translateY(-55px) scale(0.3); }
}

/* 從中心放大出現 */
.placed-coin.is-appearing {
  animation: coinAppear 0.35s cubic-bezier(0.175, 0.885, 0.32, 1.275) forwards;
  z-index: 50;
}

@keyframes coinAppear {
  0%   { opacity: 0; transform: scale(0.2); }
  70%  { opacity: 1; transform: scale(1.2); }
  100% { opacity: 1; transform: scale(1);   }
}

/* ===== 兌換紀錄 ===== */
.exchange-log {
  margin: 10px 0 4px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.log-item {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  background: #fff8e1;
  border: 1.5px solid #ffe082;
  border-radius: 20px;
  padding: 4px 14px;
  font-size: 1rem;
  font-weight: 600;
  color: #555;
  width: fit-content;
}

.log-arrow {
  font-size: 1.1rem;
  color: #e67e22;
}

.log-fade-enter-active { animation: logSlideIn 0.3s ease; }
.log-fade-leave-active { animation: logSlideIn 0.3s ease reverse; }

@keyframes logSlideIn {
  from { opacity: 0; transform: translateX(-12px); }
  to   { opacity: 1; transform: translateX(0); }
}

/* ===== 合計算式 ===== */
.equation-box {
  text-align: center;
  margin: 12px 0 8px;
  padding: 14px 24px;
  background: #fffbea;
  border: 2px solid #f5c518;
  border-radius: 10px;
  animation: fadeIn 0.25s ease;
}

.eq-text {
  font-size: 2rem;
  font-weight: 900;
  color: #333;
  letter-spacing: 2px;
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(-6px); }
  to   { opacity: 1; transform: translateY(0); }
}

/* ===== RWD ===== */
@media screen and (max-width: 768px) {
  .coins-container { padding: 0; }
  .coins-toolbar   { gap: 12px; }
  .coin-value      { font-size: 10px; }
  .drop-zone       { min-height: 240px; }
  .placed-coins    { min-height: 240px; }
  .zone-divider    { font-size: 1.8rem; padding: 0 6px; }
  .zone-total      { font-size: 1.1rem; }
  .eq-text         { font-size: 1.5rem; }
}

@media screen and (max-width: 480px) {
  .coins-toolbar { gap: 8px; }
  .coin-value    { font-size: 9px; }
  .drop-zone     { min-height: 200px; }
  .placed-coins  { min-height: 200px; }
  .zone-label    { font-size: 13px; }
  .zone-divider  { font-size: 1.4rem; margin-top: 42px; }
  .zone-total    { font-size: 1rem; }
}
</style>

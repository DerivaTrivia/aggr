<template>
  <div class="pane-chart">
    <pane-header :loading="loading" :paneId="paneId" :showTimeframe="true">
      <dropdown
        :options="{
          //clear: { label: 'Clear', click: clear },
          //trim: { label: 'Trim', click: refreshChart },
          render: { label: 'Render', click: renderChart },
          reset: { label: 'Reset', click: resetChart }
        }"
      >
        <template v-slot:option="{ value }">
          <div>
            {{ value.label }}
          </div>
        </template>
        <template v-slot:selection>
          <span>Debug</span>
        </template>
      </dropdown>
    </pane-header>
    <div class="chart__container" ref="chartContainer"></div>
    <div class="chart__indicators">
      <IndicatorControl v-for="(indicator, id) in indicators" :key="id" :indicatorId="id" :paneId="paneId" />

      <div class="column mt8">
        <a href="javascript:void(0);" @click="addIndicator" v-tippy="{ placement: 'bottom' }" title="Add" class="mr4 -text">
          <i class="icon-plus"></i>
        </a>
      </div>
    </div>
    <div class="chart__controls">
      <button class="chart__screenshot btn -text" @click="takeScreenshot"><i class="icon-photo"></i></button>
    </div>
    <IndicatorResize v-if="resizingIndicator" :indicatorId="resizingIndicator" :paneId="paneId"></IndicatorResize>
  </div>
</template>

<script lang="ts">
import { Component, Mixins } from 'vue-property-decorator'

import ChartController, { TimeRange } from './chartController'

import { formatPrice, formatAmount, formatTime, getHms, formatBytes, openBase64InNewTab } from '../../utils/helpers'
import { MAX_BARS_PER_CHUNKS } from '../../utils/constants'
import { getCustomColorsOptions } from './chartOptions'

import aggregatorService from '@/services/aggregatorService'
import historicalService from '@/services/historicalService'
import dialogService from '../../services/dialogService'

import PaneMixin from '@/mixins/paneMixin'
import PaneHeader from '../panes/PaneHeader.vue'
import IndicatorResize from './IndicatorResize.vue'
import IndicatorControl from './IndicatorControl.vue'
import CreateIndicatorDialog from './CreateIndicatorDialog.vue'
import { ChartPaneState } from '@/store/panesSettings/chart'
import { getColorLuminance, joinRgba, splitRgba } from '@/utils/colors'

@Component({
  name: 'Chart',
  components: {
    IndicatorResize,
    IndicatorControl,
    PaneHeader
  }
})
export default class extends Mixins(PaneMixin) {
  reachedEnd = false
  loading = false

  private _onStoreMutation: () => void
  private _keepAliveTimeout: number
  private _onPanTimeout: number
  private _chartController: ChartController
  private _legendElements: { [id: string]: HTMLElement }

  get markets() {
    return this.$store.state.panes.panes[this.paneId].markets
  }

  get timeframe() {
    return (this.$store.state[this.paneId] as ChartPaneState).timeframe
  }

  get indicators() {
    return (this.$store.state[this.paneId] as ChartPaneState).indicators
  }

  get resizingIndicator() {
    return (this.$store.state[this.paneId] as ChartPaneState).resizingIndicator
  }

  get showLegend() {
    return (this.$store.state[this.paneId] as ChartPaneState).showLegend
  }

  get timezoneOffset() {
    return this.$store.state.settings.timezoneOffset
  }

  $refs!: {
    chartContainer: HTMLElement
  }

  created() {
    this._chartController = new ChartController(this.paneId)
    this._legendElements = {}

    this._onStoreMutation = this.$store.subscribe(mutation => {
      switch (mutation.type) {
        case 'settings/SET_CHART_COLOR':
          if (mutation.payload) {
            this._chartController.chartInstance.applyOptions(getCustomColorsOptions(mutation.payload))
          }
          break
        case 'settings/SET_CHART_THEME':
          this._chartController.chartInstance.applyOptions(getCustomColorsOptions())
          break
        case 'settings/SET_TIMEZONE_OFFSET':
          this._chartController.setTimezoneOffset(this.timezoneOffset)
          this._chartController.clearChart()
          this._chartController.renderAll()
          break
        case 'app/EXCHANGE_UPDATED':
          this._chartController.renderAll()
          break
        case 'panes/SET_PANE_MARKETS':
          if (mutation.payload.id === this.paneId) {
            this._chartController.setMarkets(mutation.payload.markets)

            this.clear()
            this.fetch()
          }
          break
        case this.paneId + '/SET_TIMEFRAME':
          this.clear()
          this.fetch()
          break
        case this.paneId + '/SET_REFRESH_RATE':
          this._chartController.clearQueue()
          this._chartController.setupQueue()
          break
        case this.paneId + '/TOGGLE_LEGEND':
          if (this.showLegend) {
            this.bindLegend()
            this._chartController.chartInstance.subscribeCrosshairMove(this.onCrosshair)
          } else {
            this.unbindLegend()
            this._chartController.chartInstance.unsubscribeCrosshairMove(this.onCrosshair)
          }
          break
        case this.paneId + '/SET_GRIDLINES':
          this.updateGridlines(mutation.payload.type)
          break
        case this.paneId + '/SET_INDICATOR_OPTION':
          this._chartController.setIndicatorOption(mutation.payload.id, mutation.payload.key, mutation.payload.value)
          break
        case this.paneId + '/SET_INDICATOR_SCRIPT':
          this._chartController.rebuildIndicator(mutation.payload.id)
          break
        case this.paneId + '/SET_INDICATOR_SERIES':
          this.unbindLegend(mutation.payload.id)
          this.bindLegend(mutation.payload.id)
          break
        case this.paneId + '/ADD_INDICATOR':
          if (this._chartController.addIndicator(mutation.payload.id)) {
            this._chartController.redrawIndicator(mutation.payload.id)
            this.bindLegend(mutation.payload.id)
          }

          break
        case this.paneId + '/REMOVE_INDICATOR':
          this.unbindLegend(mutation.payload)
          this._chartController.removeIndicator(mutation.payload)
          break
        case 'app/SET_OPTIMAL_DECIMAL':
        case 'settings/SET_DECIMAL_PRECISION':
          for (const id in this.indicators) {
            const indicator = (this.$store.state[this.paneId] as ChartPaneState).indicators[id]

            if (!indicator.options) {
              continue
            }

            if (indicator.options.priceFormat && indicator.options.priceFormat.type === 'price') {
              this._chartController.setIndicatorOption(indicator.id, 'priceFormat.precision', mutation.payload)

              this._chartController.setIndicatorOption(indicator.id, 'priceFormat.minMove', 1 / Math.pow(10, mutation.payload))
            }
          }

          break
      }
    })
  }

  mounted() {
    this.createChart()

    this.keepAlive()
  }

  async createChart() {
    await this.$nextTick()

    this._chartController.setupQueue()
    this._chartController.createChart(this.$refs.chartContainer)

    await this.$nextTick()

    this.bindChartEvents()

    await this.fetch()

    this.positionControls()
  }

  destroyChart() {
    this.unbindChartEvents()

    this._chartController.destroy()

    clearTimeout(this._keepAliveTimeout)
  }

  beforeDestroy() {
    this.destroyChart()
    this._onStoreMutation()
  }

  /**
   * fetch whatever is missing based on visiblerange
   * @param {boolean} clear will clear the chart / initial fetch
   */
  fetch(rangeToFetch?: TimeRange) {
    if (this.loading) {
      return Promise.reject('already-fetching')
    }

    const historicalMarkets = historicalService.getHistoricalMarktets(this.markets)

    if (!historicalMarkets.length) {
      return Promise.reject('unsupported-markets')
    }

    const visibleRange = this._chartController.getVisibleRange() as TimeRange
    const timeframe = +(this.$store.state[this.paneId] as ChartPaneState).timeframe

    if (!rangeToFetch) {
      const barsCount = Math.ceil(window.innerWidth / 2)

      let leftTime

      if (this._chartController.chartCache.cacheRange && this._chartController.chartCache.cacheRange.from) {
        leftTime = this._chartController.chartCache.cacheRange.from
      } else if (visibleRange && visibleRange.from) {
        leftTime = visibleRange.from + this.timezoneOffset / 1000
      } else {
        leftTime = +new Date() / 1000
      }

      rangeToFetch = {
        from: leftTime - timeframe * barsCount,
        to: leftTime
      }

      const bytesPerBar = 206.54355723892712
      const estimatedSize = formatBytes(barsCount * historicalMarkets.length * bytesPerBar)

      this.$store.dispatch('app/showNotice', {
        id: 'fetching-' + this.paneId,
        timeout: 15000,
        title: `Fetching ${barsCount * historicalMarkets.length} × ${getHms(timeframe * 1000)} bars (~${estimatedSize})`,
        type: 'info'
      })
    }

    rangeToFetch.from = Math.floor(Math.round(rangeToFetch.from) / timeframe) * timeframe
    rangeToFetch.to = Math.ceil(Math.round(rangeToFetch.to) / timeframe) * timeframe - 1

    console.debug(`[chart/fetch] fetch rangeToFetch: FROM: ${formatTime(rangeToFetch.from)} | TO: ${formatTime(rangeToFetch.to)}`)

    this._chartController.lockRender()

    this.loading = true

    //rangeToFetch.from = +new Date(new Date(rangeToFetch.from * 1000).toISOString().replace('2021', '2019')) / 1000
    //rangeToFetch.to = +new Date(new Date(rangeToFetch.to * 1000).toISOString().replace('2021', '2019')) / 1000

    return historicalService
      .fetch(Math.round(rangeToFetch.from * 1000), Math.round(rangeToFetch.to * 1000 - 1), timeframe, historicalMarkets)
      .then(({ data, from, to, format }) => {
        /**
         * @type {Chunk}
         */
        let chunk

        switch (format) {
          case 'point':
            chunk = {
              from,
              to,
              bars: data
            }
            break
          default:
            throw new Error('unsupported-historical-data-format')
        }

        if (chunk && chunk.bars.length) {
          /**
           * @type {Chunk[]}
           */
          const chunks = [
            {
              from: chunk.from,
              to: chunk.from,
              bars: []
            }
          ]

          console.log(`[chart/fetch] success (${data.length} new ${format}s)`)

          if (chunk.bars.length > MAX_BARS_PER_CHUNKS) {
            console.debug(`[chart/fetch] response chunk is too large (> ${MAX_BARS_PER_CHUNKS} bars) -> start splitting`)
          }

          while (chunk.bars.length) {
            const bar = chunk.bars.shift()

            if (chunks[0].bars.length >= MAX_BARS_PER_CHUNKS && chunks[0].to < bar.timestamp) {
              chunks.unshift({
                from: bar.timestamp,
                to: bar.timestamp,
                bars: []
              })
            }

            chunks[0].bars.push(bar)
            chunks[0].to = bar.timestamp
          }

          if (chunks.length > 1) {
            console.debug(`[chart/fetch] splitted result into ${chunks.length} chunks`)
          }

          console.debug(`[chart/fetch] save ${chunks.length} new chunks`)
          console.debug(
            `\t-> [first] FROM: ${formatTime(chunks[0].from)} | TO: ${formatTime(chunks[0].to)} (${formatAmount(chunks[0].bars.length)} bars)`
          )
          console.debug(
            `\t-> [last] FROM: ${formatTime(chunks[chunks.length - 1].from)} | TO: ${formatTime(chunks[chunks.length - 1].to)} (${formatAmount(
              chunks[chunks.length - 1].bars.length
            )} bars)`
          )
          console.debug(
            `\t-> [current cacheRange] FROM: ${formatTime(this._chartController.chartCache.cacheRange.from)} | TO: ${formatTime(
              this._chartController.chartCache.cacheRange.to
            )}`
          )
          for (const chunk of chunks) {
            this._chartController.chartCache.saveChunk(chunk)
          }

          this._chartController.renderAll()
        }
      })
      .catch(err => {
        console.error(err)

        if (err === 'no-more-data' || err === 'unsupported-historical-data-format') {
          this.reachedEnd = true
        } else {
          this.$store.dispatch('app/showNotice', {
            title: err ? (typeof err === 'string' ? err : err.message) : `Historical API seems down at the moment`,
            type: 'error',
            icon: 'icon-warning',
            timeout: -1
          })
        }
      })
      .then(() => {
        this.$store.dispatch('app/hideNotice', 'fetching-' + this.paneId)
        this.loading = false
        this._chartController.unlockRender()
      })
  }

  /**
   * TV chart mousemove event
   * @param{TV.MouseEventParams} param tv mousemove param
   */
  onCrosshair(param) {
    let showLegend = true

    if (
      param === undefined ||
      param.time === undefined ||
      param.point.x < 0 ||
      param.point.x > this.$refs.chartContainer.clientWidth ||
      param.point.y < 0 ||
      param.point.y > this.$refs.chartContainer.clientHeight
    ) {
      showLegend = false
    }

    for (let i = 0; i < this._chartController.loadedIndicators.length; i++) {
      const indicator = this._chartController.loadedIndicators[i]

      for (let j = 0; j < this._chartController.loadedIndicators[i].apis.length; j++) {
        const api = this._chartController.loadedIndicators[i].apis[j]
        const id = indicator.id + api.id

        if (!this._legendElements[id]) {
          continue
        }

        if (!showLegend) {
          this._legendElements[id].innerText = ''
        } else {
          const data = param.seriesPrices.get(api)

          if (!data) {
            this._legendElements[id].innerText = ''
            continue
          }

          const formatFunction = indicator.options.priceFormat && indicator.options.priceFormat.type === 'volume' ? formatAmount : formatPrice

          if (typeof data === 'number') {
            this._legendElements[id].innerText = formatFunction(data)
          } else if (data.close) {
            this._legendElements[id].innerText = `O: ${formatFunction(data.open)} H: ${formatFunction(data.high)} L: ${formatFunction(
              data.low
            )} C: ${formatFunction(data.close)}`
          }
        }
      }
    }
  }

  /**
   * @param{Trade[]} trades trades to process
   */
  onTrades(trades) {
    this._chartController.queueTrades(trades)
  }

  refreshChartDimensions() {
    if (!this._chartController || !this._chartController.chartInstance) {
      return
    }

    this._chartController.chartInstance.resize(this.$el.clientWidth, this.$el.clientHeight)
  }

  onPan(visibleLogicalRange) {
    if (!visibleLogicalRange || this._chartController.panPrevented) {
      return
    }

    if (this._onPanTimeout) {
      clearTimeout(this._onPanTimeout)
      this._onPanTimeout = null
    }

    this._onPanTimeout = setTimeout(() => {
      if (this._chartController.chartCache.cacheRange.from === null) {
        return
      }

      this.fetchOrRecover(visibleLogicalRange)
    }, 200)
  }

  bindChartEvents() {
    aggregatorService.on('trades', this.onTrades)

    if (this.showLegend) {
      this._chartController.chartInstance.subscribeCrosshairMove(this.onCrosshair)
    }

    this._chartController.chartInstance.timeScale().subscribeVisibleLogicalRangeChange(this.onPan)
  }

  unbindChartEvents() {
    aggregatorService.off('trades', this.onTrades)

    if (this.showLegend) {
      this._chartController.chartInstance.unsubscribeCrosshairMove(this.onCrosshair)
    }

    this._chartController.chartInstance.timeScale().unsubscribeVisibleLogicalRangeChange(this.onPan)
  }

  keepAlive() {
    if (this._keepAliveTimeout) {
      this._chartController.chartCache.trim()

      this._chartController.renderAll()
    }

    this._keepAliveTimeout = setTimeout(this.keepAlive.bind(this), 1000 * 60 * 30)
  }

  refreshChart() {
    this._chartController.chartCache.trim()

    this.renderChart()
  }

  renderChart() {
    this._chartController.renderAll()
  }

  resetChart() {
    this.destroyChart()
    this.$nextTick(() => {
      this.createChart()
    })
  }

  addIndicator() {
    dialogService.open(CreateIndicatorDialog, { paneId: this.paneId })
  }

  fetchOrRecover(visibleLogicalRange) {
    if (!visibleLogicalRange || visibleLogicalRange.from > 0) {
      return
    }

    const barsToLoad = Math.abs(visibleLogicalRange.from)
    const rangeToFetch = {
      from: this._chartController.chartCache.cacheRange.from - barsToLoad * this.timeframe,
      to: this._chartController.chartCache.cacheRange.from
    }

    console.debug(`[chart/pan] timeout fired`)
    console.debug(`\t-> barsToLoad: ${barsToLoad}`)
    console.debug(`\t-> rangeToFetch: FROM: ${formatTime(rangeToFetch.from)} | TO: ${formatTime(rangeToFetch.to)}`)
    console.debug(
      `\t-> current cacheRange: FROM: ${formatTime(this._chartController.chartCache.cacheRange.from)} | TO: ${formatTime(
        this._chartController.chartCache.cacheRange.to
      )}`
    )

    if (
      !this.reachedEnd &&
      (!this._chartController.chartCache.cacheRange.from || rangeToFetch.to <= this._chartController.chartCache.cacheRange.from)
    ) {
      this.fetch(rangeToFetch)
    } else {
      console.warn(
        `[chart/pan] wont fetch this range\n\t-> rangeToFetch.to (${formatTime(rangeToFetch.to)}) > chart.chartCache.cacheRange.from (${formatTime(
          this._chartController.chartCache.cacheRange.from
        )})`
      )
      console.warn('(might trigger redraw with more cached chunks here...)')

      this._chartController.renderAll()
    }
  }

  onResize() {
    this.refreshChartDimensions()
  }

  clear() {
    this._chartController.clear()

    this.reachedEnd = false
  }

  async bindLegend(indicatorId?: string) {
    if (!indicatorId) {
      for (const id in this.indicators) {
        this.bindLegend(id)
      }
      return
    }

    const series = this.indicators[indicatorId].series

    await this.$nextTick()

    for (let i = 0; i < series.length; i++) {
      const id = indicatorId + series[i]

      if (this._legendElements[id]) {
        continue
      }

      const el = document.getElementById(id)

      if (el) {
        this._legendElements[id] = el
      }
    }
  }

  unbindLegend(indicatorId?: string) {
    if (!indicatorId) {
      for (const id in this.indicators) {
        this.unbindLegend(id)
      }
      return
    }

    for (const legendId in this._legendElements) {
      if (legendId.indexOf(indicatorId) === 0) {
        delete this._legendElements[legendId]
      }
    }
  }

  updateGridlines(type: 'vertical' | 'horizontal') {
    const chartOptions = this.$store.state[this.paneId] as ChartPaneState
    let show: boolean
    let color: string

    if (type === 'vertical') {
      show = chartOptions.showVerticalGridlines
      color = chartOptions.verticalGridlinesColor
    } else {
      show = chartOptions.showHorizontalGridlines
      color = chartOptions.horizontalGridlinesColor
    }

    this._chartController.chartInstance.applyOptions({
      grid: {
        [type === 'vertical' ? 'vertLines' : 'horzLines']: {
          color: color,
          visible: show
        }
      }
    })
  }

  positionControls() {
    const priceAxisCanvas = this.$refs.chartContainer.querySelector('td:last-child canvas:nth-child(2)') as HTMLElement
    const chartControls = this.$el.querySelector('.chart__controls') as HTMLElement
    chartControls.style.marginRight = priceAxisCanvas.clientWidth + 'px'
  }

  takeScreenshot() {
    const chartCanvas = this._chartController.chartInstance.takeScreenshot()
    const canvas = document.createElement('canvas')
    const ctx = canvas.getContext('2d')

    const text = this.markets.join(', ') + ' | ' + getHms(this.timeframe * 1000).toUpperCase()

    const isHRS = window.devicePixelRatio > 1

    const textPadding = isHRS ? 14 : 8
    const textFontsize = isHRS ? 16 : 10
    canvas.width = chartCanvas.width
    ctx.font = `${textFontsize}px Share Tech Mono`
    ctx.textAlign = 'left'
    ctx.textBaseline = 'top'

    const words = text.split(' ')
    const lines = []
    const date = new Date().toISOString()
    const dateWidth = ctx.measureText(date).width

    let currentLine = ' |'

    for (let i = 0; i < words.length; i++) {
      const word = words[i]
      const width = (!lines.length ? dateWidth : 0) + ctx.measureText(currentLine + ' ' + word).width
      if (width < chartCanvas.width - textPadding * 2) {
        currentLine += ' ' + word
      } else {
        lines.push(currentLine)
        currentLine = word
      }
    }

    lines.push(currentLine)

    const lineHeight = Math.round(textPadding + textPadding * 0.25)
    const headerHeight = Math.round(textPadding * 2 + lines.length * lineHeight)
    canvas.height = chartCanvas.height + headerHeight

    const backgroundColor = this.$store.state.settings.backgroundColor

    ctx.fillStyle = 'white'
    ctx.fillRect(0, 0, canvas.width, headerHeight)
    ctx.fillStyle = backgroundColor
    ctx.fillRect(0, headerHeight, canvas.width, canvas.height - headerHeight)
    ctx.drawImage(chartCanvas, 0, headerHeight)
    ctx.fillStyle = 'rgba(0,0,0,.2)'
    ctx.fillRect(0, headerHeight, canvas.width, canvas.height - headerHeight)

    ctx.fillStyle = 'black'
    ctx.font = `${textFontsize}px Share Tech Mono`
    ctx.textAlign = 'left'
    ctx.textBaseline = 'top'

    for (let i = 0; i < lines.length; i++) {
      let offset = 0

      if (i === 0) {
        ctx.fillStyle = '#2196f3'
        ctx.fillText(date, textPadding, textPadding)
        offset = dateWidth
        ctx.fillStyle = 'black'
      }

      ctx.fillText(lines[i], offset + textPadding, textPadding + lineHeight * i)
    }

    const luminance = getColorLuminance(splitRgba(backgroundColor))
    const textColor = luminance < 180 ? 'white' : 'black'

    Object.values(this.indicators).forEach((indicator, index) => {
      const options = indicator.options as any

      if (options.visible === false) {
        return
      }

      let color = options.color || options.upColor || textColor

      try {
        color = splitRgba(color)

        if (typeof color[3] !== 'undefined') {
          color[3] = 1
        }

        color = joinRgba(color)
      } catch (error) {
        // not rgb(a)
      }

      ctx.fillStyle = color
      ctx.fillText(indicator.name.toUpperCase(), textPadding, headerHeight + textPadding + index * lineHeight)
    })

    const dataURL = canvas.toDataURL('image/png')
    const startIndex = dataURL.indexOf('base64,') + 7
    const b64 = dataURL.substr(startIndex)

    openBase64InNewTab(b64, 'image/png')
  }
}
</script>

<style lang="scss" scoped>
.pane-chart {
  &:hover .chart__indicators,
  &:hover .chart__controls {
    display: block;
  }

  &.-loading {
    cursor: wait;
  }
}

.chart__container {
  position: relative;
  width: 100%;
  height: 100%;

  -webkit-touch-callout: none;
  -webkit-user-select: none;
  -khtml-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
}

.chart__indicators {
  position: absolute;
  top: 3em;
  left: 1em;
  font-family: 'Barlow Semi Condensed';
  z-index: 3;
  display: none;

  @media (-webkit-min-device-pixel-ratio: 2) {
    font-size: 12px;
  }
}

.chart__layout {
  position: absolute;
  top: 0;
  bottom: 0;
  left: 0;
  right: 0;
  z-index: 3;
}

.chart__controls {
  position: absolute;
  top: 2.5rem;
  right: 1rem;
  z-index: 2;
  display: none;

  @media screen and (max-width: 767px) {
    display: none;
  }
}
.btn.-text {
  &.-text {
    opacity: 0.5;

    &:hover {
      opacity: 1;
    }
  }
}
</style>

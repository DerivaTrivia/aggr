<template>
  <Dialog @clickOutside="hide" medium class="-sticky-footer">
    <template v-slot:header>
      <div class="title" v-if="paneId">
        {{ paneName }}'s MARKETS
        <div class="subtitle">Choose which markets <u>this pane</u> should listen to</div>
      </div>
      <div class="title" v-else>
        ALL PANES MARKETS
        <div class="subtitle">Change <u>all</u> panes markets regardless of their current configuration</div>
      </div>
      <div class="column -center"></div>
    </template>
    <div class="d-flex d-flex mobile-dir-col-desktop-dir-row">
      <div class="form-group search">
        <label>Available ({{ flattenedProducts.length }})</label>
        <div class="mb8 d-flex">
          <input ref="input" type="text" class="form-control" placeholder="search (eg: BITMEX:XBTUSD)" v-model="query" />
          <dropdown :options="filters" placeholder="Filter" title="Filters" v-tippy="{ placement: 'top' }">
            <template v-slot:selection>
              <div class="btn" :class="{ '-text': !hasFilters, ml8: hasFilters }">
                filter <i class="ml4" :class="hasFilters ? 'icon-check' : 'icon-plus'"></i>
              </div>
            </template>
            <template v-slot:option="{ index, value }">
              <div @click="toggleFilter(index)">
                <i class="icon-check" v-if="value"></i>

                {{ index }}
              </div>
            </template>
          </dropdown>
        </div>
        <table v-if="results.length">
          <tbody>
            <tr v-for="market of results" :key="market" @click="selectMarket(market)" class="-action">
              <td class="icon search__exchange" :class="'icon-' + market.split(':')[0]"></td>
              <td v-text="market"></td>
              <td>
                <i v-if="historicalMarkets.indexOf(market) !== -1" class="icon-candlestick"></i>
              </td>
            </tr>
          </tbody>
        </table>
        <div class="text-danger" v-else>
          <p>No results</p>
        </div>
      </div>
      <hr class="-horizontal" />
      <hr class="-vertical mb8" />
      <div class="form-group selection">
        <label class="text-nowrap">Selection <code v-if="paneId" class="-filled" v-text="paneId"></code></label>
        <button
          v-for="market of selection"
          :key="market"
          class="btn -small mr4 mb4 -green"
          :class="{ '-connected': activeMarkets.indexOf(market) !== -1 }"
          title="Click to remove"
          @click="deselectMarket(market)"
        >
          {{ market }}

          <i class="icon-check ml8" title="Connected" v-tippy></i>
        </button>
      </div>
    </div>

    <footer>
      <a href="javascript:void(0);" class="btn -text" @click="hide">Close</a>
      <button v-if="toConnect || toDisconnect" class="btn -large ml8" @click="apply" v-text="submitLabel"></button>
    </footer>
  </Dialog>
</template>

<script>
import Dialog from '@/components/framework/Dialog.vue'
import DialogMixin from '@/mixins/dialogMixin'
import { getBucketId } from '@/utils/helpers'
import dialogService from '@/services/dialogService'

export default {
  mixins: [DialogMixin],
  components: {
    Dialog
  },
  props: {
    query: {
      default: ''
    },
    paneId: {
      required: false
    }
  },
  data: () => ({
    markets: [],
    selection: [],
    originalSelection: [],
    filters: {
      historical: false
    }
  }),
  computed: {
    paneName() {
      if (this.paneId) {
        return this.$store.state.panes.panes[this.paneId].name
      } else {
        return null
      }
    },
    activeMarkets() {
      return this.$store.state.app.activeMarkets.map(m => m.exchange + ':' + m.pair)
    },
    toConnect() {
      return this.selection.filter(a => this.activeMarkets.indexOf(a) === -1).length
    },
    toDisconnect() {
      return this.originalSelection.filter(a => this.selection.indexOf(a) === -1).length
    },
    submitLabel() {
      const toConnect = +this.toConnect
      const toDisconnect = +this.toDisconnect
      let label = ''

      if (toConnect) {
        label += `connect ${toConnect}`
      }

      if (toDisconnect) {
        label += `${toConnect ? ' and ' : ''}disconnect ${toDisconnect}`
      }

      return label
    },
    historicalMarkets() {
      return this.$store.state.app.historicalMarkets
    },
    indexedProducts() {
      return this.$store.state.app.indexedProducts
    },
    flattenedProducts() {
      return Array.prototype.concat(...Object.values(this.indexedProducts))
    },
    queryFilter: function() {
      return new RegExp(this.query.replace(/[ ,]/g, '|'), 'i')
    },
    filteredProducts() {
      return this.flattenedProducts.filter(a => {
        if (this.filters.historical) {
          return this.historicalMarkets.indexOf(a) !== -1
        }

        return true
      })
    },
    results: function() {
      return this.filteredProducts.filter(a => this.selection.indexOf(a) === -1 && this.queryFilter.test(a)).slice(0, 100)
    },
    hasFilters() {
      return this.filters.historical
    }
  },
  watch: {
    '$store.state.app.showSearch': function(value) {
      if (!value) {
        this.close(false)
      }
    }
  },
  created() {
    if (this.paneId) {
      this.selection = this.$store.state.panes.panes[this.paneId].markets.slice()
    } else {
      this.selection = this.activeMarkets.slice()
    }

    this.originalSelection = this.selection.slice()
  },
  mounted() {
    this.$nextTick(() => {
      this.$refs.input.focus()
    })
  },
  methods: {
    containMultipleMarketsConfigurations() {
      return (
        Object.keys(this.$store.state.panes.panes)
          .map(id => getBucketId(this.$store.state.panes.panes[id].markets))
          .filter((v, i, a) => a.indexOf(v) === i).length > 1
      )
    },
    selectMarket(market) {
      this.selection.push(market)
    },
    deselectMarket(market) {
      const index = this.selection.indexOf(market)

      if (index !== -1) {
        this.selection.splice(index, 1)
      }
    },
    async apply() {
      if (!this.paneId) {
        if (
          this.containMultipleMarketsConfigurations() &&
          !(await dialogService.confirm('Are you sure ? Some of the panes are watching specific markets.'))
        ) {
          return
        }

        this.$store.dispatch('panes/setMarketsForAll', this.selection)
      } else {
        this.$store.dispatch('panes/setMarketsForPane', {
          id: this.paneId,
          markets: this.selection
        })
      }

      this.hide()
    },
    hide() {
      this.$store.dispatch('app/hideSearch')
    },
    toggleFilter(key) {
      this.$set(this.filters, key, !this.filters[key])
    }
  }
}
</script>
<style lang="scss" scoped>
.selection {
  flex-basis: 0;
  margin-left: 1rem;

  text-align: right;

  .btn {
    text-transform: none;
    opacity: 0.75;

    i {
      display: none;
    }

    &.-added {
      &:after {
        content: '';
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        opacity: 0;
        background-color: white;
        animation: 1s $ease-out-expo highlight;
        pointer-events: none;
      }
    }

    &.-connected {
      opacity: 1;

      i {
        display: block;
      }
    }
  }
}
.search {
  flex-grow: 1;

  table {
    border: 0;
    border-collapse: collapse;
    width: 100%;
  }

  td {
    padding: 0.25rem 0.33rem;
  }

  tr:hover {
    background-color: rgba(white, 0.1);
    cursor: pointer;
  }

  &__exchange {
    background-position: right;
  }
}
</style>

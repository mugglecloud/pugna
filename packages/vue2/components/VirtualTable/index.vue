<template>
  <div ref="container" class="virtual-table">
    <table class="table">
      <thead class="header">
        <tr v-if="viewWidth" :style="headerStyle">
          <th
            v-for="h in columns"
            :key="h.label"
            :style="{
              ...computeCellWidth(h),
              ...headerCellStyle,
              textAlign: h.align,
            }"
            :title="h.label"
          >
            {{ h.label }}
          </th>
        </tr>
      </thead>
      <tbody ref="body" class="body stripe">
        <div
          class="before-content"
          :style="{ height: `${this.beforeContentHeight}px` }"
        ></div>
        <tr
          v-for="(r, i) in computedData"
          :key="r[key]"
          :height="calcRowHeight(r)"
        >
          <td
            v-for="c in columns"
            :key="[r[key], c.label].join('_')"
            :style="computeCellWidth(c)"
          >
            <div class="cell-wrapper" :title="r[c.key]">
              <slot :row="r" :column="c" :index="i + start"></slot>
              <!-- <resize-observer @notify="(...args) => handleResize(r, ...args)" /> -->
            </div>
          </td>
        </tr>
        <div
          class="after-content"
          :style="{ height: `${this.afterContentHeight}px` }"
        ></div>
        <div class="loading-data-tips">
          {{ noMoreData ? "没有更多数据了" : "正在加载..." }}
        </div>
      </tbody>
    </table>
    <!-- <p class="no-data" v-if="!mounting && !loading && !data.length">暂无数据</p> -->
  </div>
</template>

<script>
import memoize from 'lodash.memoize';

function getScrollbarWidth () {
  // Creating invisible container
  const outer = document.createElement('div');
  outer.style.visibility = 'hidden';
  outer.style.overflow = 'scroll'; // forcing scrollbar to appear
  outer.style.msOverflowStyle = 'scrollbar'; // needed for WinJS apps
  document.body.appendChild(outer);

  // Creating inner element and placing it in the container
  const inner = document.createElement('div');
  outer.appendChild(inner);

  // Calculating difference between container's full width and the child width
  const scrollbarWidth = (outer.offsetWidth - inner.offsetWidth);

  // Removing temporary elements from the DOM
  outer.parentNode.removeChild(outer);

  return scrollbarWidth;
}

function findIndexLesser (orderedlist, threshold, a, b) {
  let i = parseInt((a + b) / 2)
  if (i == a) return i

  let v = orderedlist.get(i)
  if (v < threshold) {
    return findIndexLesser(orderedlist, threshold, i, b)
  } else {
    return findIndexLesser(orderedlist, threshold, a, i)
  }
}

function findIndexGreater (orderedlist, threshold, a, b) {
  let i = findIndexLesser(orderedlist, threshold, a, b)
  let v = orderedlist.get(i + 1)
  return Math.min(b, v == threshold ? i + 2 : i + 1)
}

const noop = () => { }

export default {
  name: 'VirtualTable',
  components: { ResizeObserver },
  props: {
    data: { type: Array },
    headerHeight: { type: String | Number, default: () => 40 },
    rowHeight: { type: String | Number | Function, default: () => 40 },
    columns: { type: Array },
    rowKey: { type: String, default: () => 'id' },
    distance: { type: String | Number, default: () => 10 },
    headerCellStyle: { type: Object }
  },
  data () {
    return {
      boundingWidth: 0,
      boudingHeight: 0,

      scrollbarWidth: 0,
      xscroll: 0,
      yscroll: 0,

      start: 0,
      end: 0,
      computedData: [],

      totalRows: this.data.length,
      totalHeight: 0,

      beforeContentHeight: 0,
      afterContentHeight: 0,

      loading: false,
      mounting: true,
      noMoreData: false
    }
  },
  created () {
    this.__heightCache = new Map()
    this.totalHeight = this.calcHeight(this.data, this.__heightCache)
  },
  watch: {
    data (val) {
      this.__isLoadingMore = false
      const noMoreData = this.totalRows === val.length
      this.totalRows = val ? val.length : 0

      this.__heightCache = new Map()
      this.totalHeight = this.calcHeight(val, this.__heightCache)

      this.renderData()

      if (noMoreData) {
        this.noMoreData = true
      } else {
        if (this.totalRows <= this.count) {
          this.emitLoadMore()
        }
      }
    },
    yscroll (val) {
      this.renderData()

      if (this.noMoreData) return

      const overthreshold = this.totalHeight - this.yscroll - this.viewHeight - this.distance

      if (overthreshold <= 0) {
        this.emitLoadMore()
      }
    },
  },
  computed: {
    key () {
      return this.rowKey || 'id'
    },
    headerStyle () {
      return { width: `calc(100% - ${this.scrollbarWidth}px)`, height: `${this.headerHeight}px`, transform: `translateX(-${this.xscroll}px)` }
    },
    spanUnit () {
      return (this.viewWidth - this.scrollbarWidth) / 24
    },
    viewWidth () {
      return this.boundingWidth
    },
    viewHeight () {
      return this.boudingHeight - this.headerHeight
    },
    count () {
      return this.end - this.start
    }
  },
  methods: {
    calcHeight (data, cache = null) {
      return data.reduce((acc, cur, i) => {
        const r = acc + this.calcRowHeight(cur)
        if (cache) cache.set(i, r)
        return r
      }, 0)
    },
    calcRowHeight (r) {
      let height = 40;
      if (typeof this.rowHeight === 'function') height = this.rowHeight(r)
      else height = this.rowHeight
      return Number(height)
    },
    emitLoadMore () {
      if (this.__isLoadingMore) return
      this.__isLoadingMore = true
      this.$emit('load-more')

      console.log('emit load more')
    },
    computeCellWidth (h) {
      return this._doComputeCellWidth && this._doComputeCellWidth(h.span || 2)
    },
    computeSpanWitdh (span) {
      return span * this.spanUnit
    },
    renderData () {
      const start = findIndexLesser(this.__heightCache, this.yscroll, 0, this.__heightCache.size)
      const height = this.calcHeight(this.data.slice(start, start + 2))
      const end = findIndexGreater(this.__heightCache, this.yscroll + this.viewHeight + height, 0, this.__heightCache.size)

      this.start = start
      this.end = end + 1
      this.computedData = this.data.slice(start, this.end)

      this.beforeContentHeight = start > 0 ? this.__heightCache.get(start - 1) : 0
      const visibleContentHeight = this.calcHeight(this.computedData)
      this.afterContentHeight = this.totalHeight - this.beforeContentHeight - visibleContentHeight

      // console.log('render data ...', start, end, this.__heightCache.size)
    }
  },
  mounted () {
    this._doComputeCellWidth = memoize((span) => {
      const width = `${this.computeSpanWitdh(span)}px`;
      return {
        minWidth: width
      }
    })

    this.$nextTick(() => {
      const rect = this.$refs.container.getBoundingClientRect()
      this.boundingWidth = rect.width
      this.boudingHeight = rect.height
      this.scrollbarWidth = getScrollbarWidth()

      this.$refs.body.addEventListener('scroll', (e) => {
        // const scrollWidth = e.target.scrollWidth
        // const scrollHeight = e.target.scrollHeight
        this.xscroll = e.target.scrollLeft
        this.yscroll = Math.min(this.totalHeight - this.viewHeight, e.target.scrollTop)

        // console.log('scroll event ...', this.yscroll, this.viewHeight, this.totalHeight - this.yscroll - this.viewHeight)
      })
    })
  },
  beforeDestroy () {
    if (this.$refs.body) {
      this.$refs.body.removeEventListener('scroll', null)
    }
  }
}

</script>

<style lang='scss' scoped>
.virtual-table {
  display: flex;
  flex-direction: column;
  width: 100%;
  height: 100%;
  overflow: hidden;
  font-size: 13px;

  .table {
    display: flex;
    flex-direction: column;
    width: 100%;
    height: 100%;

    td,
    th {
      padding: 10px;
      box-sizing: border-box;
      text-align: left;
      width: 100%;
      height: 100%;
      border-right: 1px solid #ebeef5;
    }

    tr {
      display: flex;
      flex-direction: row;
      align-items: center;
      width: 100%;

      > td:first-child {
        border-left: 1px solid #ebeef5;
      }
    }

    td {
      display: inline-flex;
      align-items: center;
      flex-grow: 1;
      border-bottom: 1px solid #ebeef5;
      overflow: hidden;

      .cell-wrapper {
        position: relative;
        width: 100%;
      }
    }

    .header {
      display: block;
      width: 100%;
      background: lightgrey;

      th {
        user-select: none;
        font-size: 14px;
        font-weight: bold;
        flex-grow: 1;
      }
    }

    .stripe {
      tr:nth-child(odd) {
        background: #eeeeee;
      }
      tr:nth-child(even) {
        background: #fafafa;
      }
    }

    .body {
      overflow: auto;

      & > tr:focus,
      & > tr:hover {
        background: aliceblue;
      }
    }
  }
}

.before-content,
.after-content {
  width: 100%;
  height: auto;
  visibility: hidden;
}

.loading-data-tips {
  padding: 10px;
  text-align: center;
  height: 40px;
}

.no-data {
  flex-grow: 2;
  width: 100%;
  height: 100%;
  min-height: calc(100% - 80px);
  display: flex;
  align-items: center;
  justify-content: center;
}
</style>

<template>
  <div :id="chartId"></div>
</template>
<script>
import DataSet from '@antv/data-set'
import G2 from '@antv/g2'
import elementResizeDetectorMaker from 'element-resize-detector'
import Filter from '@/store/model/filter'
import {uniqueCount} from '../utils/unique'
import vizConfig from '../config/index'
let cnt = 0
function getChartId () {
  return 'trick-interval-' + cnt++
}
const MEASURE_NAME = 'MEASURE_NAME'
const MEASURE_VALUE = 'MEASURE_VALUE'
const EXPAND_DIM = 'EXPAND_DIM'
export default {
  name: 'trick-interval',
  props: {
    dimensions: { type: Array },
    measures: { type: Array },
    dataSource: { type: Array },
    isLabel: { type: Array },
    operations: { type: Array },
    color: { type: String },
    shape: { type: String },
    size: { type: String },
    opacity: { type: String },
    filters: { type: Array },
    coord: { type: String },
    transpose: { type: Boolean },
    constScale: { type: Boolean },
    event: {
      type: Boolean,
      default: true
    }
  },
  data () {
    return {
      chartId: getChartId(),
      chart: undefined,
      renderCondition: {
        dimensions: [0, Infinity],
        measures: [1, Infinity]
      }
    }
  },
  mounted () {
    this.chart = new G2.Chart({
      container: this.chartId,
      padding: [60, 40],
      forceFit: true
    })
    this.renderChart()
    this.chart.on('interval:click', ev => {
      let filters = [
        new Filter({
          name: this.position[0],
          type: 'string',
          filterType: 'equal',
          values: [ev.data._origin[this.position[0]]]
        })
      ]
      this.$emit('geomClick', { filters })
    })
    let self = this
    this.erd = elementResizeDetectorMaker()
    this.erd.listenTo(this.$el, (ele) => {
      self.chart.changeSize(ele.offsetWidth, ele.offsetHeight)
    })
  },
  beforeDestroy () {
    this.chart.off('interval:click')
    this.erd.removeAllListeners(this.$el)
    this.erd = null
  },
  watch: {
    dimensions () {
      this.renderChart()
    },
    measures () {
      this.renderChart()
    },
    dataSource () {
      this.renderChart()
    },
    isLabel () {
      this.renderChart()
    },
    operations () {
      this.renderChart()
    },
    color () {
      this.renderChart()
    },
    opacity () {
      this.renderChart()
    },
    shape () {
      this.renderChart()
    },
    size () {
      this.renderChart()
    },
    filters () {
      this.renderChart()
    },
    coord () {
      this.renderChart()
    },
    transpose () {
      this.renderChart()
    },
    constScale () {
      this.renderChart()
    }
  },
  computed: {
    dimCode () {
      return this.$props.dimensions.slice(0, this.renderCondition.dimensions[1])
    },
    meaCode () {
      return this.$props.measures.slice(0, this.renderCondition.measures[1])
    },
    mapMeaCode () {
      return this.meaCode.map(item => {
        return item + '(aggregate)'
      })
    },
    scale () {
      const {dataSource, constScale, coord} = this.$props
      let ans = {}
      let fields = [...this.meaCode, ...this.dimCode, MEASURE_NAME, MEASURE_VALUE]
      fields.forEach(item => {
        ans[item] = {
          sync: constScale
        }
      })
      let dim = this.dimCode.slice(-1)[0]
      let dimAxis = dataSource.map(item => {
        return item[dim]
      })
      if (uniqueCount(dimAxis) > 20) {
        this.dimCode.forEach(item => {
          ans[item] = {
            sync: constScale,
            tickCount: parseInt(vizConfig.tickCount[coord] / this.meaCode.length)
          }
        })
      }
      return ans
    },
    algebra () {
      const {filters, dataSource} = this.$props
      let filterData = dataSource.filter(row => {
        return filters.every(f => {
          if (f.filterType === 'range') {
            return row[f.name] >= f.value[0] && row[f.name] <= f.value[1]
          } else {
            return f.value.indexOf(row[f.name]) !== -1
          }
        })
      })
      let expandData = filterData.map(row => {
        let expand = '_'
        if (this.dimCode.length > 1) {
          // fuck node slice -2
          // it will get the index of -1 if the dimcode.length == 1
          this.dimCode.slice(0, -1).forEach(dim => {
            expand += `${dim}=${row[dim]}\n`
          })
        }
        return {
          ...row,
          [EXPAND_DIM]: expand
        }
      })
      return expandData
    },
    data () {
      let dv = new DataSet().createView()
      dv.source(this.algebra)
      dv.transform({
        type: 'aggregate',
        fields: this.meaCode,
        operations: this.$props.operations,
        as: this.mapMeaCode,
        groupBy: [EXPAND_DIM].concat(this.dimCode)
      })
      dv.transform({
        type: 'fold',
        fields: this.mapMeaCode,
        key: MEASURE_NAME,
        value: MEASURE_VALUE
      })
      return dv
    },
    position () {
      if (this.dimCode.length === 0) {
        return [MEASURE_NAME, MEASURE_VALUE]
      }
      return this.dimCode.slice(-1).concat(MEASURE_VALUE)
    },
    facetFields () {
      return this.dimCode.slice(0, -1)
    },
    allowRender () {
      if (this.meaCode.length < this.renderCondition.measures[0]) {
        return false
      }
      if (this.dimCode.length < this.renderCondition.dimensions[0]) {
        return false
      }
      return true
    }
  },
  methods: {
    renderChart () {
      if (this.allowRender) {
        const {color, shape, opacity, size, coord, transpose, event} = this.$props
        let self = this
        this.chart.clear()
        this.chart.source(this.data)
        this.chart.scale(this.scale)
        this.chart.tooltip(event)
        let c = this.chart.coord(coord)
        if (transpose) { c.transpose() }
        this.chart.facet('rect', {
          fields: [EXPAND_DIM, MEASURE_NAME],
          eachView (view) {
            let geom = view.interval()
            geom.position(self.position)
            if (typeof color !== 'undefined') {
              geom.color(color)
            }
            if (typeof opacity !== 'undefined') {
              geom.opacity(opacity)
            }
            if (typeof size !== 'undefined') {
              geom.size(size)
            }
            if (typeof shape !== 'undefined') {
              geom.shape(shape)
            }
            geom.active(event)
          }
        })
        this.chart.render()
      }
    }
  }
}
</script>

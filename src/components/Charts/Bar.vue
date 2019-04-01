<template>
  <svg
    :viewBox="viewBox"
    preserveAspectRatio="xMinYMin meet"
    class="bar-chart"
  >
    <g :transform="stageTransform">
      <text
        :x="namePosition.x"
        :y="namePosition.y"
        text-anchor="middle"
      >{{ name }}</text>
      <g ref="bandSeries" />
      <g
        ref="bandsAxis"
        :transform="bandsAxisTransform"
      />
      <g ref="valueAxis" />
      <g
        ref="legend"
        :transform="legendTransform"
      />
      <g
        v-show="tooltip.show"
        :transform="tooltipTransform"
        class="bar-tooltip"
      >
        <rect
          :width="layout.tooltip.width"
          :height="layout.tooltip.height"
          class="frame"
        />
        <text
          :dx="layout.tooltip.text.x"
          :dy="layout.tooltip.text.y"
          class="name-text"
        >{{ tooltipName }}</text>
        <text
          :transform="tooltipValueItemTransform"
          class="value-item"
        >
          <tspan>{{ names[tooltip.data.index] }}: </tspan>
          <tspan class="value">{{ tooltipValue }}</tspan>
        </text>
      </g>
    </g>
  </svg>
</template>

<script>
    import * as d3 from 'd3'

    export default {
        props: {
            layout: {
                type: Object,
                default: () => ({
                    width: 500,
                    height: 280,
                    margin: {
                        left: 36,
                        right: 36,
                        top: 12,
                        bottom: 52
                    },
                    name: {
                        x: 5,
                        y: 10
                    },
                    tooltip: {
                        width: 156,
                        height: 30,
                        text: {
                            x: 6,
                            y: 12
                        },
                        valueItemY: 24
                    },
                    legend: {
                        x: 302,
                        y: 20,
                        width: 174,
                        itemSize: 18,
                        rect: {
                            size: 11,
                            padding: 5
                        },
                        textPadding: 9
                    }
                })
            },
            data: {
                type: Array,
                default: () => [],
            },
            columns: {
                type: Array,
                default: () => []
            },
            colors: {
                type: Array,
                default: () => ['#80b081', '#b08097']
            },
            names: {
                type: Array,
                default: () => ['Chart 1', 'Chart 2']
            },
            parse: {
                type: Object,
                default: () => ({
                    band: d3.timeParse('%Y-%m-%d'),
                    value: d => d
                })
            },
            band: {
                type: Object,
                default: () => ({
                    ticks: 10,
                    format: d3.timeFormat("%b %d, %Y")
                })
            },
            options: {
                type: Object,
                default: () => ({
                    scaleLog: false,
                    legend: true,
                    tooltip: true,
                    axisCount: 1
                })
            },
            tooltipFormat: {
                type: Object,
                default: () => ({
                    band: d => d,
                    value: [d => d, d => d]
                })
            },
            axisLegend: {
                type: Object,
                default: () => ({
                    x: 'Date',
                    y: ['Value 1', 'Value 2']
                })
            },
            name: {
                type: String,
                default: ''
            }
        },
        data() {
            return {
                scale: {
                    band: this.getBandScale(),
                    bandItem: this.getBandItemScale(),
                    value: this.getValueScale(),
                    colors: this.getColorsScale()
                },
                tooltip: {
                    show: false,
                    pos: {x: 0, y: 0},
                    data: {
                        index: null,
                        column: null,
                        name: null,
                        value: null
                    }
                }
            }
        },
        computed: {
            padded() {
                const width = this.layout.width + this.layout.margin.left + this.layout.margin.right
                const height = this.layout.height + this.layout.margin.top + this.layout.margin.bottom

                return {width, height}
            },
            viewBox() {
                return `0 0 ${this.padded.width} ${this.padded.height}`
            },
            stageTransform() {
                return `translate(${this.layout.margin.left},${this.layout.margin.top})`
            },
            namePosition() {
                const x = this.layout.width / 2 + this.layout.name.x
                const y = this.layout.name.y

                return {x, y}
            },
            bandsAxisTransform() {
                return `translate(0,${this.layout.height})`
            },
            tooltipTransform() {
                let x = this.tooltip.pos.x

                if (x + this.layout.tooltip.width > this.layout.width) {
                    x -= (this.layout.tooltip.width + 40)
                }

                return `translate(${x},${this.tooltip.pos.y})`
            },
            tooltipValueItemTransform() {
                return `translate(${this.layout.tooltip.text.x},${this.layout.tooltip.valueItemY})`
            },
            tooltipName() {
                return (this.tooltip.data.name !== null) ? this.tooltipFormat.band(this.parse.band(this.tooltip.data.name)) : ''
            },
            tooltipValue() {
                return (this.tooltip.data.value !== null) ? this.tooltipFormat.value[this.tooltip.data.index](this.parse.value(this.tooltip.data.value)) : ''
            },
            legendTransform() {
                return `translate(-${this.layout.legend.width * 0.2},0)`
            }
        },
        created() {
            this.updateScales()
        },
        mounted() {
            this.drawBandsAxis()
            this.drawValueAxis()
            this.drawBands()

            if (this.options.legend) {
                this.drawLegend()
            }
        },
        methods: {
            getBandScale() {
                return d3.scaleBand().range([0, this.layout.width]).paddingInner(0.01)
            },
            getBandItemScale() {
                return d3.scaleBand().padding(0.05)
            },
            valueExtents() {
                return this.columns.map(column => d3.extent(this.data, d => this.parse.value(d[column])))
            },
            initValueScale() {
                return this.options.scaleLog ? d3.scaleLog() : d3.scaleLinear()
            },
            getValueScale() {
                return this.valueExtents().map(ext => this.initValueScale().range([this.layout.height - 10, 10]).domain(ext))
            },
            getColorsScale() {
                return d3.scaleOrdinal().range(this.colors)
            },
            updateScales() {
                this.scale.band.domain(this.data.map(d => this.parse.band(d.name)))
                this.scale.bandItem.domain(this.columns).rangeRound([0, this.scale.band.bandwidth()])

                if (this.options.axisCount === 1) {
                    const extents = [d3.min(this.valueExtents(), ext => ext[0]), d3.max(this.valueExtents(), ext => ext[1])]

                    this.scale.value.map(scl => scl.domain(extents))
                }
                else if (this.options.axisCount === 2) {
                    const extents = [
                        this.valueExtents().filter((d, i) => !(i % 2)),
                        this.valueExtents().filter((d, i) => (i % 2))
                    ]

                    const domains = extents.map(ext => [d3.min(ext, e => e[0]), d3.max(ext, e => e[1])])

                    for (let index in this.scale.y) {
                        this.scale.value[index].domain(domains[!(index % 2) ? 0 : 1])
                    }
                }
            },
            drawBandsAxis() {
                const $axisBands = d3.select(this.$refs.bandsAxis)
                const ticks = Math.max(Math.floor(this.data.length / this.band.ticks), 1)
                const axisBands = d3.axisBottom(this.scale.band).tickValues(this.scale.band.domain().filter((d, i) => (this.band.ticks === null) || !(i % ticks))).tickFormat(this.band.format)
                const $$axisBands = $axisBands.call(axisBands)

                $axisBands.selectAll('text')
                    .style('text-anchor', 'start')
                    .attr('dx', '1.3em')
                    .attr('dy', '0.8em')
                    .attr('transform', 'rotate(65)')

                $$axisBands.append('text')
                    .attr('dx', this.layout.width - 5)
                    .attr('dy', '-0.5em')
                    .style('text-anchor', 'end')
                    .text(this.axisLegend.x)
                    .attr('class', 'axis-legend-text')
            },
            drawValueAxis() {
                const $axisValue = d3.select(this.$refs.valueAxis)

                if (this.options.axisCount === 1) {
                    const axisValue = d3.axisLeft(this.scale.value[0]).ticks(null, 's')
                    const $$axisValue = $axisValue.call(axisValue)

                    $$axisValue.append('text')
                        .attr('transform', 'rotate(-90)')
                        .attr('y', 6)
                        .attr('dy', '.71em')
                        .style('text-anchor', 'end')
                        .text(this.axisLegend.y.join(', '))
                        .attr('class', 'axis-legend-text')
                }
                else if (this.options.axisCount === 2) {
                    const axisValue = [
                        d3.axisLeft(this.scale.value[0]).ticks(null, 's'),
                        d3.axisRight(this.scale.value[1]).ticks(null, 's')
                    ]

                    // Draw first axis
                    const $firstAxisValue = $axisValue.append('g')
                        .call(axisValue[0])

                    $firstAxisValue.append('text')
                        .attr('transform', 'rotate(-90)')
                        .attr('y', 6)
                        .attr('dy', '.71em')
                        .style('text-anchor', 'end')
                        .text(this.axisLegend.y[0])
                        .attr('class', 'axis-legend-text')

                    //  Draw second axis
                    const $secondAxisValue = $axisValue.append('g')
                        .attr('transform', `translate(${this.layout.width},0)`)
                        .call(axisValue[1])

                    $secondAxisValue.append('text')
                        .attr('transform', 'rotate(-90)')
                        .attr('y', -10)
                        .attr('dy', '.91em')
                        .style('text-anchor', 'end')
                        .text(this.axisLegend.y[1])
                        .attr('class', 'axis-legend-text')
                }
            },
            drawBands() {
                const $bandSeries = d3.select(this.$refs.bandSeries)
                const $band = $bandSeries.selectAll('g').data(this.data)
                    .enter()
                    .append('g')
                    .attr('transform', d => `translate(${this.scale.band(this.parse.band(d.name))},0)`)
                    .selectAll('rect').data(d => this.columns.map((column, i) => ({index: i, 'column': column, name: d.name, value: d[column]})))
                    .enter()
                    .append('rect')
                    .attr('class', 'bar')
                    .attr('x', d => this.scale.bandItem(d.column))
                    .attr('y', (d, i) => this.scale.value[i](this.parse.value(d.value)))
                    .attr('width', this.scale.bandItem.bandwidth())
                    .attr('height', (d, i) => this.layout.height - this.scale.value[i](d.value))
                    .attr('fill', d => this.scale.colors(d.column))

                if (this.options.tooltip) {
                    $band.on('mouseover', this.mouseover)
                        .on('mouseout', this.mouseover)
                        .on('mousemove', this.mousemove)
                }
            },
            drawLegend() {
                const $legend = d3.select(this.$refs.legend)

                $legend.append('rect')
                    .attr('x', this.layout.legend.x)
                    .attr('y', this.layout.legend.y)
                    .attr('width', this.layout.legend.width)
                    .attr('height', this.layout.legend.itemSize * this.columns.length)
                    .attr('class', 'legend-frame')

                const $legendItem = $legend.selectAll('g').data(this.names)
                    .enter()
                    .append('g')
                    .attr('transform', (d, i) => `translate(0, ${this.layout.legend.y + i * this.layout.legend.itemSize})`)

                const rectSize = this.layout.legend.rect.size
                const rectPadding = this.layout.legend.rect.padding

                $legendItem.append('rect')
                    .attr('x', (this.layout.legend.x + this.layout.legend.width) - rectSize - rectPadding)
                    .attr('y', rectPadding)
                    .attr('width', rectSize)
                    .attr('height', rectSize)
                    .attr('fill',this.scale.colors)

                const textPadding = this.layout.legend.textPadding

                $legendItem.append('text')
                    .attr('x', this.layout.width - this.layout.legend.width + textPadding)
                    .attr('y', rectPadding + textPadding)
                    .style('font-size', '0.42em')
                    .text(d => d)

            },
            mouseover() {
                this.tooltip.show = false
            },
            mousemove(d) {
                const mousePos = d3.mouse(this.$el)

                this.tooltip.pos.x = mousePos[0] - 20
                this.tooltip.pos.y = mousePos[1] - 15
                this.tooltip.data = d

                this.tooltip.show = true
            }
        },
    }
</script>

<style lang="scss">
  .bar-chart {
    width: 100%;

    .tick {
      text {
        font-size: 0.6em;
        font-weight: 400;
      }
    }

    .axis-legend-text {
      fill: black;
      font-size: 0.72em;
    }

    .legend-frame {
      fill: #ececec;
      opacity: 0.4;
    }

    .bar-tooltip {
      .frame {
        fill: #cccccc;
        opacity: 0.8
      }

      .name-text {
        font-size: 0.48em;
      }

      tspan {
        font-size: 0.36em;
      }

      tspan.value {
        font-weight: bold;
      }
    }

    .bar {
      &:hover {
        fill: brown !important;
      }
    }
  }
</style>
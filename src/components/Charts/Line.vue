<template>
  <svg
    :viewBox="viewBox"
    preserveAspectRatio="xMinYMin meet"
    class="line-chart"
  >
    <g :transform="stageTransform">
      <text
        :x="namePosition.x"
        :y="namePosition.y"
        text-anchor="middle"
      >{{ name }}</text>
      <g
        ref="axisX"
        :transform="axisXTransform"
      />
      <g ref="axisY" />
      <g ref="lines" />
      <g ref="legend" />
      <line
        v-show="tooltip.show"
        :x1="tooltip.pos.x"
        :x2="tooltip.pos.x"
        :y2="layout.height"
        y1="0"
        class="hover-line"
      />
      <g
        v-show="tooltip.show"
        ref="tooltip"
        :transform="tooltipTransform"
        class="line-tooltip"
      >
        <rect
          :width="layout.tooltip.width"
          :height="tooltipHeight"
          :rx="layout.tooltip.rx"
          :ry="layout.tooltip.ry"
          class="frame"
        />
        <text
          :x="layout.tooltip.text.x"
          :y="layout.tooltip.text.yGrid"
          class="x-text"
        >{{ getTooltipTextX(tooltip.index) }}</text>
      </g>
      <rect
        ref="overlay"
        :width="layout.width"
        :height="layout.height"
        class="overlay"
      />
    </g>
  </svg>
</template>

<script>
    import * as d3 from 'd3'

    const circleSymbol = '●'

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
                        bottom: 36
                    },
                    name: {
                        x: 5,
                        y: 5
                    },
                    tooltip: {
                        width: 180,
                        heightGrid: 30,
                        rx: 3,
                        ry: 3,
                        text: {
                            x: 6,
                            yGrid: 12
                        }
                    },
                    legend: {
                        x: 302,
                        y: 30,
                        width: 180,
                        itemSize: 18,
                        circle: {
                            size: 4,
                            padding: 4
                        },
                        textPadding: 3
                    }
                }),
            },
            data: {
                type: Array,
                default: () => [],
            },
            lines: {
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
                    x: d3.timeParse('%Y-%m-%d'),
                    y: d => d
                })
            },
            scaleTypeY: {
                type: String,
                default: 'linear'
            },
            ticks: {
                type: Object,
                default: () => ({
                    x: {count: 10, format: d3.timeFormat("%b %d, %Y")},
                    y: {count: 30, format: d => d}
                })
            },
            options: {
                type: Object,
                default: () => ({
                    area: true,
                    legend: true,
                    tooltip: true,
                    axisCount: 1
                })
            },
            tooltipFormat: {
                type: Object,
                default: () => ({
                    x: d => d,
                    y: [d => d, d => d]
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
                    x: this.getScaleX(),
                    y: this.getScaleY(),
                    colors: this.getColorsScale()
                },
                tooltip: {
                    show: false,
                    pos: {x: 0, y: 0},
                    index: null
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
            axisXTransform() {
                return `translate(0,${this.layout.height})`
            },
            tooltipTransform() {
                const xOffset = 10
                let [x, y] = [this.tooltip.pos.x + xOffset, this.tooltip.pos.y]

                if (x + this.layout.tooltip.width > this.layout.width) {
                    x -= (this.layout.tooltip.width + 2*xOffset)
                }

                if (y + this.tooltipHeight > this.layout.height) {
                    y = this.layout.height - this.tooltipHeight - xOffset
                }

                return `translate(${x},${y})`
            },
            tooltipHeight() {
                return (this.lines.length + 2) * this.layout.tooltip.text.yGrid
            }
        },
        watch: {
            'tooltip.index': function(val) {
                const $tooltip = d3.select(this.$refs.tooltip)

                $tooltip.selectAll('.item').select('.value').text((d, i) => this.tooltipFormat.y[i](this.parse.y(this.data[val][d])))

                const $lines = d3.select(this.$refs.lines)

                $lines.selectAll('.tooltip-point')
                    .attr('cx', this.scale.x(this.parse.x(this.data[val].x)))
                    .attr('cy', (d, i) => this.scale.y[i](this.parse.y(this.data[val][d])))
            }
        },
        created() {
            this.updateScaleY()
        },
        mounted() {
            this.drawAxisX()
            this.drawAxisY()

            this.drawLines()

            if (this.options.legend) {
                this.drawLegend()
            }

            if (this.options.tooltip) {
                this.drawTooltip()
            }
        },
        methods: {
            initScaleY() {
                switch (this.scaleTypeY) {
                    case 'linear':
                        return d3.scaleLinear()
                    case 'log':
                        return d3.scaleLog()
                }
            },
            extentsY() {
                return this.lines.map(line => d3.extent(this.data, d => this.parse.y(d[line])))
            },
            getScaleX() {
                return d3.scalePoint().range([0, this.layout.width]).domain(this.data.map(d => this.parse.x(d.x)))
            },
            getScaleY() {
                return this.extentsY().map(ext => this.initScaleY().range([this.layout.height, 10]).domain(ext))
            },
            updateScaleY() {
                if (this.options.axisCount === 1) {
                    const extents = [d3.min(this.extentsY(), ext => ext[0]), d3.max(this.extentsY(), ext => ext[1])]

                    this.scale.y.map(scl => scl.domain(extents))
                }
                else if(this.options.axisCount === 2) {
                    const extents = [
                        this.extentsY().filter((d, i) => !(i % 2)),
                        this.extentsY().filter((d, i) => (i % 2))
                    ]

                    const domains = extents.map(ext => [d3.min(ext, e => e[0]), d3.max(ext, e => e[1])])

                    for (let index in this.scale.y) {
                        this.scale.y[index].domain(domains[!(index % 2) ? 0 : 1])
                    }
                }
            },
            getColorsScale() {
                return d3.scaleOrdinal().range(this.colors)
            },
            drawAxisX() {
                const $axisX = d3.select(this.$refs.axisX)
                const ticks = Math.max(Math.ceil(this.data.length / this.ticks.x.count), 1)
                const axisX = d3.axisBottom(this.scale.x).tickValues(this.scale.x.domain().filter((d, i) => (this.ticks.x.count === null) || !( i % ticks))).tickFormat(this.ticks.x.format)
                const $$axisX = $axisX.call(axisX)

                if (this.ticks.x.count === null) {
                    $axisX.selectAll('text')
                        .style('text-anchor', 'start')
                        .attr('dy', '1.5em')
                        .attr('transform', 'rotate(65)')
                }

                $$axisX.append('text')
                    .attr('dx', this.layout.width - 5)
                    .attr('dy', '-0.5em')
                    .style('text-anchor', 'end')
                    .text(this.axisLegend.x)
                    .attr('class', 'axis-legend-text')
            },
            drawAxisY() {
                const $axisY = d3.select(this.$refs.axisY)

                if (this.options.axisCount === 1) {
                    const axisY = d3.axisLeft(this.scale.y[0]).ticks(this.ticks.y.count).tickFormat(this.ticks.y.format)
                    const $$axisY = $axisY.call(axisY)

                    $$axisY.append('text')
                        .attr('transform', 'rotate(-90)')
                        .attr('y', 6)
                        .attr('dy', '.71em')
                        .style('text-anchor', 'end')
                        .text(this.axisLegend.y.join(', '))
                        .attr('class', 'axis-legend-text')
                }
                else if (this.options.axisCount === 2) {
                    const axisY = [
                        d3.axisLeft(this.scale.y[0]).ticks(this.ticks.y.count).tickFormat(this.ticks.y.format),
                        d3.axisRight(this.scale.y[1]).ticks(this.ticks.y.count).tickFormat(this.ticks.y.format)
                    ]

                    // Draw first axis
                    const $firstAxisY = $axisY.append('g')
                        .call(axisY[0])

                    $firstAxisY.append('text')
                        .attr('transform', 'rotate(-90)')
                        .attr('y', 6)
                        .attr('dy', '.71em')
                        .style('text-anchor', 'end')
                        .text(this.axisLegend.y[0])
                        .attr('class', 'axis-legend-text')

                    //  Draw second axis
                    const $secondAxisY = $axisY.append('g')
                        .attr('transform', `translate(${this.layout.width},0)`)
                        .call(axisY[1])

                    $secondAxisY.append('text')
                        .attr('transform', 'rotate(-90)')
                        .attr('y', -15)
                        .attr('dy', '.71em')
                        .style('text-anchor', 'end')
                        .text(this.axisLegend.y[1])
                        .attr('class', 'axis-legend-text')
                }
                //  Perhaps in the future it will be possible to add a third axis
            },
            getPoints(key) {
                return this.data.map(value => ({x: value.x, y: value[key]}))
            },
            drawLines() {
                const $lines = d3.select(this.$refs.lines)
                //  Drawing series group
                const $series = $lines.selectAll('.series')
                    .data(this.lines)
                    .enter()
                    .append('g')
                    .attr('class', 'series')

                //  Drawing line
                const line = this.scale.y.map(scl => d3.line().x(d => this.scale.x(this.parse.x(d.x))).y(d => scl(this.parse.y(d.y)))/*.curve(d3.curveCardinal)*/)

                $series.append('path')
                    .attr('class', 'line')
                    .attr('d', (d, i) => line[i](this.getPoints(d)))
                    .style('fill', 'none')
                    .style('stroke', (d, i) => this.scale.colors(i))
                    .style('stroke-width', 0.6)

                //  Drawing area
                if (this.options.area) {
                    const area = this.scale.y.map(scl => d3.area().x(d => this.scale.x(this.parse.x(d.x))).y0(this.layout.height).y1(d => scl(this.parse.y(d.y)))/*.curve(d3.curveCardinal)*/)

                    $series.append('path')
                        .attr('class', 'area')
                        .attr('d', (d, i) => area[i](this.getPoints(d)))
                        .style('fill', (d, i) => this.scale.colors(i))
                        .style('fill-opacity', 0.6)
                        .style('stroke', 'none')
                }

                //  Drawing tooltip points
                $series.append('circle')
                    .attr('class', 'tooltip-point')
                    .attr('r', 4)
                    .attr('cx', (d, i) => this.scale.x(this.parse.x(this.data[i].x)))
                    .attr('cy', (d, i) => this.scale.y[i](this.parse.y(this.data[i][d])))
                    .style('fill', '#fff')
                    .style('stroke', (d, i) => this.scale.colors(i))
                    .style('stroke-width', 3)
                    .style('stroke-opacity', 0.2)
                    .style('display', 'none')
            },
            drawLegend() {
                const $legend = d3.select(this.$refs.legend)

                $legend.append('rect')
                    .attr('x', this.layout.legend.x)
                    .attr('y', this.layout.legend.y)
                    .attr('width', this.layout.legend.width)
                    .attr('height', this.layout.legend.itemSize * this.names.length)
                    .attr('class', 'legend-frame')

                const $legendItem = $legend.selectAll('g').data(this.names)
                    .enter()
                    .append('g')
                    .attr('transform', (d, i) => `translate(${this.layout.legend.x}, ${this.layout.legend.y + i * this.layout.legend.itemSize})`)

                const circleSize = this.layout.legend.circle.size
                const circlePadding = this.layout.legend.circle.padding

                $legendItem.append('circle')
                    .attr('cx', 2*circlePadding)
                    .attr('cy', 2*circlePadding)
                    .attr('r', circleSize)
                    .attr('fill', (d, i) => this.scale.colors(i))

                const textPadding = this.layout.legend.textPadding

                $legendItem.append('text')
                    .attr('x', 2*circlePadding + 2.5*circleSize + textPadding)
                    .attr('y', 2*circlePadding + textPadding)
                    .style('font-size', '0.42em')
                    .text(d => d)
            },
            drawTooltip() {
                const $overlay = d3.select(this.$refs.overlay)

                $overlay.on('mouseover', this.mouseover)
                    .on('mouseout', this.mouseover)
                    .on('mousemove', this.mousemove)

                const $tooltipItem = d3.select(this.$refs.tooltip).selectAll('.item').data(this.lines)
                    .enter()
                    .append('text')
                    .attr('class', 'item')
                    .attr('transform', (d, i) => `translate(10,${(i + 2) * this.layout.tooltip.text.yGrid})`)

                $tooltipItem.append('tspan')
                    .attr('class', 'circle')
                    .attr('x', 0)
                    .style('fill', this.scale.colors)
                    .text(circleSymbol)

                $tooltipItem.append('tspan').attr('dx', 4).text((d, i) => `${this.names[i]}: `)

                $tooltipItem.append('tspan').attr('class', 'value').attr('dx', 0)
            },
            mouseover() {
                d3.selectAll('.tooltip-point').style('display', 'none')

                this.tooltip.show = false
            },
            mousemove() {
                const scales = this.data.map(d => this.scale.x(this.parse.x(d.x)))
                const index = d3.bisect(scales, d3.mouse(this.$refs.overlay)[0], 1) - 1
                const data = this.data[index]

                this.tooltip.pos.x = this.scale.x(this.parse.x(data.x))
                this.tooltip.pos.y = d3.mouse(this.$refs.overlay)[1]
                this.tooltip.index = index

                d3.selectAll('.tooltip-point').style('display', null)

                this.tooltip.show = true
            },
            getTooltipTextX(index) {
                return (index !== null) ? this.tooltipFormat.x(this.parse.x(this.data[index].x)) : ''
            }
        },
    }
</script>

<style lang="scss">
  .line-chart {
    width: 100%;

    .tick {
      text {
        font-size: 0.55em;
        font-weight: 400;
      }
    }

    .legend-frame {
      fill: #ececec;
      opacity: 0.4;
    }

    .axis-legend-text {
      fill: black;
      font-size: 0.6em;
    }

    .overlay {
      fill: none;
      opacity: 0;
      pointer-events: all;
    }

    .hover-line {
      stroke: #B74779;
      stroke-width: 1px;
      stroke-dasharray: 4, 4;
    }

    .line-tooltip {
      .frame {
        fill: #cccccc;
        stroke: #b3b3b3;
        stroke-width: 2px;
        opacity: 0.8
      }

      .x-text {
        font-size: 0.5em;
        color:rgba(23, 24, 27, 0.85);
        fill:rgba(23, 24, 27, 0.85);
      }

      tspan , tspan.circle {
        font-size: 0.4em;
      }

      tspan.value {
        font-weight: bold;
      }
    }
  }
</style>
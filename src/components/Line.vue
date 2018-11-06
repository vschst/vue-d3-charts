<template>
  <svg 
    :viewBox="viewBox" 
    preserveAspectRatio="xMinYMin meet" 
    class="line-chart">
    <g :transform="stageTransform">
      <g 
        ref="xAxis" 
        :transform="xAxisTransform"/>
      <g ref="yAxis"/>
      <g 
        ref="lines" 
        :transform="linesTransform"/>
      <g ref="legend"/>
      <line
        v-show="tooltip.show"
        :x1="tooltip.x"
        :x2="tooltip.x"
        :y2="layout.height"
        y1="0"
        class="hover-line"/>
      <g 
        v-show="tooltip.show"
        ref="tooltip"
        :transform="tooltipTransform"
        class="line-tooltip">
        <rect
          :width="layout.tooltip.width"
          :height="layout.tooltip.height"
          :rx="layout.tooltip.rx"
          :ry="layout.tooltip.ry"
          class="frame"/>
        <text 
          :x="layout.tooltip.text.x"
          :y="layout.tooltip.text.yGrid"
          class="x-text">{{ getTooltipTextX(tooltip.index) }}</text>
      </g>
      <rect 
        ref="overlay"
        :width="layout.width" 
        :height="layout.height" 
        class="overlay"/>
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
                    width: 900,
                    height: 500,
                    margin: {
                        left: 50,
                        right: 30,
                        top: 20,
                        bottom: 60
                    },
                    tooltip: {
                        width: 250,
                        height: 100,
                        rx: 5,
                        ry: 5,
                        text: {
                            x: 10,
                            yGrid: 20
                        }
                    },
                    legend: {
                        width: 180,
                        itemSize: 30,
                        circle: {
                            size: 7,
                            padding: 7
                        },
                        textPadding: 5
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
                default: () => []
            },
            parse: {
                type: Object,
                default: () => ({
                    x: d3.timeParse('%Y-%m-%d'),
                    y: d => d
                })
            },
            scaleType: {
                type: Object,
                default: () => ({
                    x: 'time',
                    y: 'linear'
                })
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
                    tooltip: true
                })
            },
            axisLegend: {
                type: Object,
                default: () => ({
                    x: 'Date',
                    y: 'Value'
                })
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
                    x: 0,
                    y: 0,
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
            xAxisTransform() {
                return `translate(0,${this.layout.height})`
            },
            linesTransform() {
                return `translate(0,-${this.layout.margin.top})`
            },
            tooltipTransform() {
                const xOffset = 10
                let [x, y] = [this.tooltip.x + xOffset, this.tooltip.y]

                if (x + this.layout.tooltip.width > this.layout.width) {
                    x -= (this.layout.tooltip.width + 2*xOffset)
                }

                if (y + this.layout.tooltip.height > this.layout.height) {
                    y = this.layout.height - this.layout.tooltip.height - xOffset
                }

                return `translate(${x},${y})`
            }
        },
        watch: {
            'tooltip.index': function(val) {
                const $tooltip = d3.select(this.$refs.tooltip)

                $tooltip.selectAll('.item').select('.value').text(d => this.parse.y(this.data[val][d]))

                const $lines = d3.select(this.$refs.lines)

                $lines.selectAll('.tooltip-point')
                    .attr('cx', this.scale.x(this.parse.x(this.data[val].x)))
                    .attr('cy', d => this.scale.y(this.parse.y(this.data[val][d])))
            }
        },
        mounted() {
            this.drawAxis('xAxis')
            this.drawAxis('yAxis')

            this.drawLines()

            if (this.options.legend) {
                this.drawLegend()
            }

            if (this.options.tooltip) {
                this.drawTooltip()
            }
        },
        methods: {
            initScale(axis) {
                switch (this.scaleType[axis]) {
                    case 'time':
                        return d3.scaleTime()
                    case 'linear':
                        return d3.scaleLinear()
                    case 'log':
                        return d3.scaleLog().base(Math.E)
                }
            },
            extentsY() {
                const min = d3.min(this.lines, line => d3.min(this.data, d => this.parse.y(d[line])))
                const max = d3.max(this.lines, line => d3.max(this.data, d => this.parse.y(d[line])))
                const range = max - min

                //  Set domain to be extent +- 7%
                return [min - (range * 0.07), max + (range * 0.07)]
            },
            getScaleX() {
                return this.initScale('x').range([0, this.layout.width]).domain(d3.extent(this.data, d => this.parse.x(d.x)))
            },
            getScaleY() {
                return this.initScale('y').range([this.layout.height, 0]).domain(this.extentsY())
            },
            getColorsScale() {
                return d3.scaleOrdinal().range(this.colors)
            },
            drawAxis(ref) {
                const $axis = d3.select(this.$refs[ref])
                const axisGenerator = {
                    xAxis: d3.axisBottom(this.scale.x).ticks(this.ticks.x.count).tickFormat(this.ticks.x.format),
                    yAxis: d3.axisLeft(this.scale.y).ticks(this.ticks.y.count).tickFormat(this.ticks.y.format)
                }

                const $$axis = $axis.call(axisGenerator[ref])

                if (ref === 'xAxis') {
                    $$axis.append('text')
                        .attr('dx', this.layout.width - 5)
                        .attr('dy', '-0.5em')
                        .style('text-anchor', 'end')
                        .text(this.axisLegend.x)
                        .attr('class', 'legend')
                }
                else if (ref === 'yAxis') {
                    $$axis.append('text')
                        .attr('transform', 'rotate(-90)')
                        .attr('y', 6)
                        .attr('dy', '.71em')
                        .style('text-anchor', 'end')
                        .text(this.axisLegend.y)
                        .attr('class', 'legend')
                }
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
                const line = d3.line().x(d => this.scale.x(this.parse.x(d.x))).y(d => this.scale.y(this.parse.y(d.y))).curve(d3.curveCardinal)

                $series.append('path')
                    .attr('class', 'line')
                    .attr('d', d => line(this.getPoints(d)))
                    .style('fill', 'none')
                    .style('stroke', (d, i) => this.scale.colors(i))
                    .style('stroke-width', 2)

                //  Drawing area
                if (this.options.area) {
                    const area = d3.area().x(d => this.scale.x(this.parse.x(d.x))).y0(this.layout.height + this.layout.margin.top).y1(d => this.scale.y(this.parse.y(d.y))).curve(d3.curveCardinal)

                    $series.append('path')
                        .attr('class', 'area')
                        .attr('d', d => area(this.getPoints(d)))
                        .style('fill', (d, i) => this.scale.colors(i))
                        .style('fill-opacity', 0.6)
                        .style('stroke', 'none')
                }

                //  Drawing tooltip points
                $series.append('circle')
                    .attr('class', 'tooltip-point')
                    .attr('r', 4)
                    .attr('cx', (d, i) => this.scale.x(this.parse.x(this.data[i].x)))
                    .attr('cy', (d, i) => this.scale.y(this.parse.y(this.data[i][d])))
                    .style('fill', '#fff')
                    .style('stroke', (d, i) => this.scale.colors(i))
                    .style('stroke-width', 4)
                    .style('stroke-opacity', 0.3)
                    .style('display', 'none')
            },
            drawLegend() {
                const $legend = d3.select(this.$refs.legend)
                const legendX = this.layout.width - this.layout.legend.width

                $legend.append('rect')
                    .attr('x', legendX)
                    .attr('width', this.layout.legend.width)
                    .attr('height', this.layout.legend.itemSize * this.names.length)
                    .attr('fill', '#ececec')

                const $legendItem = $legend.selectAll('g').data(this.names)
                    .enter()
                    .append('g')
                    .attr('transform', (d, i) => `translate(${legendX}, ${i * this.layout.legend.itemSize})`)

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

                $tooltipItem.append('tspan').attr('dx', 4).text(d => `${d}: `)

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

                this.tooltip.x = this.scale.x(this.parse.x(data.x))
                this.tooltip.y = d3.mouse(this.$refs.overlay)[1]

                this.tooltip.index = index

                d3.selectAll('.tooltip-point').style('display', null)

                this.tooltip.show = true
            },
            getTooltipTextX(index) {
                return (index !== null) ? this.data[index].x : ''
            }
        },
    }
</script>

<style lang="scss">
    .line-chart {
      width: 100%;

      .legend {
        fill: black;
        font-size: 1em;
      }

      .overlay {
        fill: none;
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
          font-size:13px;
          color:rgba(23, 24, 27, 0.85);
          fill:rgba(23, 24, 27, 0.85);
        }

        tspan {
          font-size: 0.8em;
        }

        tspan.circle {
          font-size: 0.6em;
        }

        tspan.value {
          font-weight: bold;
        }
      }
    }
</style>
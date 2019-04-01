<template>
  <svg
    :viewBox="viewBox"
    preserveAspectRatio="xMinYMin meet"
    class="scatter-chart"
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
      <g ref="dots" />
      <g
        v-show="tooltip.show"
        :transform="tooltipTransform"
        class="scatter-tooltip"
      >
        <rect
          :width="layout.tooltip.width"
          :height="layout.tooltip.height"
          class="frame"
        />
        <text :transform="tooltipArgumentTransform">
          <tspan>{{ axisLegend.x }}: </tspan>
          <tspan class="value">{{ tooltipArgumentText }}</tspan>
        </text>
        <text :transform="tooltipValueTransform">
          <tspan>{{ tooltipValueLegendText }}: </tspan>
          <tspan class="value">{{ tooltipValueText }}</tspan>
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
                        bottom: 36
                    },
                    name: {
                        x: 25,
                        y: 10
                    },
                    tooltip: {
                        width: 180,
                        height: 30,
                        text: {
                            argument: {
                                x: 3,
                                y: 9
                            },
                            value: {
                                x: 3,
                                y: 24
                            }
                        }
                    },
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
                default: () => ['#80b099']
            },
            parse: {
                type: Object,
                default: () => ({
                    x: d => d,
                    y: d => d
                })
            },
            ticks: {
                type: Object,
                default: () => ({
                    x: {count: 50, format: d => d},
                    y: {count: 30, format: d => d}
                })
            },
            tooltipFormat: {
                type: Object,
                default: () => ({
                    x: d => d,
                    y: [d => d]
                })
            },
            axisLegend: {
                type: Object,
                default: () => ({
                    x: 'Count',
                    y: ['Value 1']
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
                    data: {
                        index: null,
                        x: null,
                        y: null
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
            axisXTransform() {
                return `translate(0,${this.layout.height})`
            },
            tooltipTransform() {
                let x = this.tooltip.pos.x

                if (x + this.layout.tooltip.width > this.layout.width) {
                    x -= (this.layout.tooltip.width + 5)
                }

                return `translate(${x},${this.tooltip.pos.y})`
            },
            tooltipArgumentTransform() {
                return `translate(${this.layout.tooltip.text.argument.x},${this.layout.tooltip.text.argument.y})`
            },
            tooltipArgumentText() {
                return (this.tooltip.data.x !== null) ? this.tooltipFormat.x(this.parse.x(this.tooltip.data.x)) : ''
            },
            tooltipValueTransform() {
                return `translate(${this.layout.tooltip.text.value.x},${this.layout.tooltip.text.value.y})`
            },
            tooltipValueLegendText() {
                return (this.tooltip.data.index !== null) ? this.axisLegend.y[this.tooltip.data.index] : ''
            },
            tooltipValueText() {
                return (this.tooltip.data.y !== null) ? this.tooltipFormat.y[this.tooltip.data.index](this.parse.y(this.tooltip.data.y)) : ''
            }
        },
        mounted() {
            this.drawAxisX()
            this.drawAxisY()
            this.drawDots()
        },
        methods: {
            getScaleX() {
                return d3.scaleLinear().range([0, this.layout.width]).domain(d3.extent(this.data, d => this.parse.x(d.x)))
            },
            extentsY() {
                const extents = this.lines.map(line => d3.extent(this.data, d => this.parse.y(d[line])))

                return [d3.min(extents, ext => ext[0]), d3.max(extents, ext => ext[1])]
            },
            getScaleY() {
                return d3.scaleLinear().range([this.layout.height, 0]).domain(this.extentsY())
            },
            getColorsScale() {
                return d3.scaleOrdinal().range(this.colors)
            },
            drawAxisX() {
                const $axisX = d3.select(this.$refs.axisX)
                const axisX = d3.axisBottom(this.scale.x).ticks(this.ticks.x.count).tickFormat(this.ticks.x.format)
                const $$axisX = $axisX.call(axisX)

                $axisX.selectAll('text')
                    .style('text-anchor', 'start')
                    .attr('dx', '1em')
                    .attr('dy', '-0.5em')
                    .attr('transform', 'rotate(45)')

                $$axisX.append('text')
                    .attr('dx', this.layout.width - 5)
                    .attr('dy', '-0.5em')
                    .style('text-anchor', 'end')
                    .text(this.axisLegend.x)
                    .attr('class', 'axis-legend-text')
            },
            drawAxisY() {
                const $axisY = d3.select(this.$refs.axisY)
                const axisY = d3.axisLeft(this.scale.y).ticks(this.ticks.y.count).tickFormat(this.ticks.y.format)
                const $$axisY = $axisY.call(axisY)

                $$axisY.append('text')
                    .attr('transform', 'rotate(-90)')
                    .attr('y', 6)
                    .attr('dy', '.71em')
                    .style('text-anchor', 'end')
                    .text(this.axisLegend.y.join(', '))
                    .attr('class', 'axis-legend-text')
            },
            drawDots() {
                const $dots = d3.select(this.$refs.dots)

                //  Drawing series group
                const $series = $dots.selectAll('.series')
                    .data(this.lines.map((line, i) => ({name: line, dots: this.data.map(d => ({index: i, x: d.x, y: d[line]}))})))
                    .enter()
                    .append('g')
                    .attr('class', 'series')

                //  Drawing dots
                $series.selectAll('.dot')
                    .data(d => d.dots)
                    .enter().append('circle')
                    .attr('class', 'dot')
                    .attr('r', 2)
                    .attr('cx', d => this.scale.x(this.parse.x(d.x)))
                    .attr('cy', d => this.scale.y(this.parse.y(d.y)))
                    .style('fill', (d, i) => this.scale.colors(i))
                    .on('mouseover', this.mouseover)
                    .on('mouseout', this.mouseout)
            },
            mouseover(d) {
                const pos = {x: this.scale.x(this.parse.x(d.x)) + 5, y: this.scale.y(this.parse.y(d.y)) - 5}

                this.tooltip.pos.x = pos.x
                this.tooltip.pos.y = pos.y
                this.tooltip.data = d

                this.tooltip.show = true
            },
            mouseout() {
                this.tooltip.show = false
            }
        }
    }
</script>

<style lang="scss">
    .scatter-chart {
        width: 100%;

        .tick {
            text {
                font-size: 0.55em;
                font-weight: 400;
            }
        }

        .axis-legend-text {
            fill: black;
            font-size: 0.6em;
        }

        .dot {
            stroke: #000;

            &:hover {
                fill: brown !important;
            }
        }

        .scatter-tooltip {
            .frame {
                fill: #cccccc;
                opacity: 0.8
            }

            text {
                font-size: 0.5em;

                .value {
                    font-weight: bold;
                }
            }
        }
    }
</style>
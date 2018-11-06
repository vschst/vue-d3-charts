<template>
  <svg 
    :viewBox="viewBox" 
    preserveAspectRatio="xMinYMin meet">
    <g :transform="stageTransform">
      <g ref="bandSeries"/>
      <g 
        ref="bandsAxis" 
        :transform="bandsAxisTransform"/>
      <g ref="valueAxis"/>
      <g ref="legend"/>
      <g 
        v-show="tooltip.show" 
        :transform="tooltipTransform"
        class="bar-tooltip">
        <rect 
          :width="layout.tooltip.width" 
          :height="layout.tooltip.height" 
          class="frame"/>
        <text 
          :dx="layout.tooltip.text.dx" 
          :dy="layout.tooltip.text.dy">{{ tooltip.value }}</text>
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
                    width: 900,
                    height: 500,
                    margin: {
                        left: 40,
                        right: 30,
                        top: 20,
                        bottom: 80
                    },
                    tooltip: {
                        width: 150,
                        height: 20,
                        text: {
                            dx: 10,
                            dy: 15
                        }
                    },
                    legend: {
                        width: 200,
                        itemSize: 30,
                        rect: {
                            size: 19,
                            padding: 5
                        },
                        textPadding: 15
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
                default: () => []
            },
            options: {
                type: Object,
                default: () => ({
                    legend: true,
                    tooltip: true
                })
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
                    value: ''
                }
            }
        },
        computed: {
            //  Twice
            padded() {
                const width = this.layout.width + this.layout.margin.left + this.layout.margin.right
                const height = this.layout.height + this.layout.margin.top + this.layout.margin.bottom

                return {width, height}
            },
            //  Twice
            viewBox() {
                return `0 0 ${this.padded.width} ${this.padded.height}`
            },
            //  Twice
            stageTransform() {
                return `translate(${this.layout.margin.left},${this.layout.margin.top})`
            },
            getBandNames() {
                return this.data.map(d => d.name)
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
            }
        },
        created() {
            this.updateScales()
        },
        mounted() {
            this.drawAxis('bandsAxis')
            this.drawAxis('valueAxis')
            this.drawBands()

            if (this.options.legend) {
                this.drawLegend()
            }
        },
        methods: {
            getBandScale() {
                return d3.scaleBand().rangeRound([0, this.layout.width]).paddingInner(0.1)
            },
            getBandItemScale() {
                return d3.scaleBand().padding(0.05)
            },
            getValueScale() {
                return d3.scaleLinear().rangeRound([this.layout.height, 0]).domain([0, 1.1 * d3.max(this.data, d => d3.max(this.columns, key => d[key]))])
            },
            //  Double
            getColorsScale() {
                return d3.scaleOrdinal().range(this.colors)
            },
            updateScales() {
                this.scale.band.domain(this.getBandNames)
                this.scale.bandItem.domain(this.columns).rangeRound([0, this.scale.band.bandwidth()])
            },
            drawAxis(ref) {
                const $axis = d3.select(this.$refs[ref])
                const axisGenerator = {
                    bandsAxis: d3.axisBottom(this.scale.band),
                    valueAxis: d3.axisLeft(this.scale.value).ticks(null, 's')
                }

                $axis.call(axisGenerator[ref])

                if (ref === 'bandsAxis') {
                    $axis.selectAll('text')
                        .style('text-anchor', 'start')
                        .attr('dy', '1.55em')
                        .attr('transform', 'rotate(65)')
                }
            },
            drawBands() {
                const $bandSeries = d3.select(this.$refs.bandSeries)
                const $band = $bandSeries.selectAll('g').data(this.data)
                    .enter()
                    .append('g')
                    .attr('transform', d => `translate(${this.scale.band(d.name)},0)`)
                    .selectAll('rect').data(d => this.columns.map(key => ({'key': key, value: d[key]})))
                    .enter()
                    .append('rect')
                    .attr('class', 'bar')
                    .attr('x', d => this.scale.bandItem(d.key))
                    .attr('y', d => this.scale.value(d.value))
                    .attr('width', this.scale.bandItem.bandwidth())
                    .attr('height', d => this.layout.height - this.scale.value(d.value))
                    .attr('fill', d => this.scale.colors(d.key))

                if (this.options.tooltip) {
                    $band.on('mouseover', this.mouseover)
                        .on('mouseout', this.mouseover)
                        .on('mousemove', this.mousemove)
                }
            },
            drawLegend() {
                const $legend = d3.select(this.$refs.legend)

                $legend.append('rect')
                    .attr('x', this.layout.width - this.layout.legend.width)
                    .attr('width', this.layout.legend.width)
                    .attr('height', this.layout.legend.itemSize * this.columns.length)
                    .attr('fill', '#ececec')

                const $legendItem = $legend.selectAll('g').data(this.columns)
                    .enter()
                    .append('g')
                    .attr('transform', (d, i) => `translate(0, ${i * this.layout.legend.itemSize})`)

                const rectSize = this.layout.legend.rect.size
                const rectPadding = this.layout.legend.rect.padding

               $legendItem.append('rect')
                    .attr('x', this.layout.width - rectSize - rectPadding)
                    .attr('y', rectPadding)
                    .attr('width', rectSize)
                    .attr('height', rectSize)
                    .attr('fill',this.scale.colors)

                const textPadding = this.layout.legend.textPadding

                $legendItem.append('text')
                    .attr('x', this.layout.width - this.layout.legend.width + textPadding)
                    .attr('y', rectPadding + textPadding)
                    .text(d => d)

            },
            mouseover() {
                this.tooltip.show = false
            },
            mousemove(d) {
                const mousePos = d3.mouse(this.$el)

                this.tooltip.pos.x = mousePos[0] - 20
                this.tooltip.pos.y = mousePos[1] - 15
                this.tooltip.value = d.value
                this.tooltip.show = true
            }
        },
    }
</script>

<style lang="scss">
    .bar-tooltip {
      .frame {
        fill: #cccccc;
        opacity: 0.8
      }

      text {
        font-size: 0.8em;
        font-weight: bold;
      }
    }
</style>
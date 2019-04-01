<template>
  <svg
    :viewBox="viewBox"
    preserveAspectRatio="xMinYMin meet"
    class="pie-chart"
  >
    <text
      :x="namePosition.x"
      :y="namePosition.y"
      text-anchor="middle"
    >{{ name }}</text>
    <g
      ref="stage"
      :transform="stageTransform"
    >
      <g ref="slices" />
      <g ref="labels" />
      <g ref="lines" />
      <g
        v-show="tooltip.show"
        :transform="tooltipTransform"
        class="pie-tooltip"
      >
        <rect
          :width="layout.tooltip.width"
          :height="layout.tooltip.height"
          class="frame"
        />
        <text
          :x="layout.tooltip.text.x"
          :y="layout.tooltip.text.y"
          class="label-text"
        >{{ tooltip.data.label }}</text>
        <text
          :transform="tooltipValueItemTransform"
          class="value-item"
        >
          <tspan>{{ argument }}: </tspan>
          <tspan class="value">{{ tooltip.data.value }}</tspan>
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
                    name: {
                        x: 0,
                        y: 15
                    },
                    tooltip: {
                        width: 180,
                        height: 30,
                        text: {
                            x: 6,
                            y: 12
                        },
                        valueItemY: 24
                    }
                })
            },
            data: {
                type: Array,
                default: () => [],
            },
            colors: {
                type: Array,
                default: () => ['#20d6e9', '#20e998', '#ff0078', '#ff0800', '#8dff00', '#fff200', '#a42eff', '#3b2eff']
            },
            argument: {
                type: String,
                default: ''
            },
            name: {
                type: String,
                default: ''
            }
        },
        data() {
            return {
                colorScale: this.getColorScale(),
                tooltip: {
                    show: false,
                    pos: {x: 0, y: 0},
                    data: {
                        label: null,
                        value: null
                    }
                }
            }
        },
        computed: {
            viewBox() {
                return `0 0 ${this.layout.width} ${this.layout.height}`
            },
            stageTransform() {
                return `translate(${this.layout.width / 2},${this.layout.height / 2})`
            },
            namePosition() {
                const x = this.layout.width / 2 + this.layout.name.x
                const y = this.layout.name.y

                return {x, y}
            },
            radius() {
                return Math.min(this.layout.width, this.layout.height) / 2
            },
            sumValue() {
                return this.data.reduce((sum, d) => sum + d.value, 0)
            },
            tooltipTransform() {
                return `translate(${this.tooltip.pos.x},${this.tooltip.pos.y})`
            },
            tooltipValueItemTransform() {
                return `translate(${this.layout.tooltip.text.x},${this.layout.tooltip.valueItemY})`
            }
        },
        mounted() {
            this.drawSlices()
            this.drawLabels()
            this.drawLines()
        },
        methods: {
            pie: d3.pie().sort(null).value(d => d.value),
            arc() {
                return d3.arc().outerRadius(this.radius * 0.8).innerRadius(this.radius * 0.4)
            },
            outerArc() {
                return d3.arc().innerRadius(this.radius * 0.9).outerRadius(this.radius * 0.9)
            },
            getColorScale() {
                const colors = this.data.map((d, i) => {
                    const index = i % this.colors.length

                    return this.colors[index]
                })

                if (colors.pop() === colors[0] && colors.length > 2) {
                    colors[colors.length - 1] = colors[1]
                }

                return d3.scaleOrdinal().domain(Object.keys(this.data)).range(colors)
            },
            drawSlices() {
                const $slices = d3.select(this.$refs.slices)
                const $slice = $slices.selectAll('.slice').data(this.pie(this.data))
                    .enter()
                    .append('path')
                    .attr('d', this.arc())
                    .style('fill', (d, i) => this.colorScale(i))
                    .style('opacity', 0.8)
                    .attr('class', 'slice')
                    .on('mouseover', this.mouseover)
                    .on('mouseout', this.mouseover)
                    .on('mousemove', this.mousemove)

                $slice.exit().remove()
            },
            midAngle(d) {
                return d.startAngle + (d.endAngle - d.startAngle) / 2
            },
            itemVisibility(d) {
                return ((d.value / this.sumValue) > 0.02) ? 'visible' : 'hidden'
            },
            percentValue(value) {
                return value / this.sumValue
            },
            drawLabels() {
                const $labels = d3.select(this.$refs.labels)
                const $text = $labels.selectAll('text').data(this.pie(this.data))

                $text.enter()
                    .append('text')
                    .attr('dy', '.35em')
                    .text(d =>  {
                        const label = d.data.label
                        const percentValue = d3.format('.0%')(this.percentValue(d.data.value))

                        return `${label} - ${percentValue}`
                    })
                    .attr('transform', d => {
                        const pos = this.outerArc().centroid(d)

                        pos[0] = this.radius * ((this.midAngle(d) < Math.PI) ? 1 : -1)

                        return `translate(${pos})`
                    })
                    .attr('class', 'label-text')
                    .style('text-anchor', d => {
                        return (this.midAngle(d) < Math.PI) ? 'start' : 'end'
                    })
                    .style('visibility', this.itemVisibility)

                $text.exit().remove()
            },
            drawLines() {
                const $lines = d3.select(this.$refs.lines)
                const $polyline = $lines.selectAll('polyline').data(this.pie(this.data))

                $polyline.enter()
                    .append('polyline')
                    .attr('points', d => {
                        const pos = this.outerArc().centroid(d)

                        pos[0] = this.radius * 0.95 * ((this.midAngle(d) < Math.PI) ? 1 : -1)

                        return [this.arc().centroid(d), this.outerArc().centroid(d), pos]
                    })
                    .style('visibility', this.itemVisibility)

                $polyline.exit().remove()
            },
            mouseover() {
                this.tooltip.show = false
            },
            mousemove(d) {
                const mousePos = [d3.mouse(this.$el)[0] - this.layout.width / 2 + 20, d3.mouse(this.$el)[1] - this.layout.height / 2 + 20]

                this.tooltip.pos.x = mousePos[0]
                this.tooltip.pos.y = mousePos[1]
                this.tooltip.data = d.data

                this.tooltip.show = true
            }
        }
    }
</script>

<style lang="scss">
  .pie-chart {
    width: 100%;

    path.slice {
      stroke-width: 5px;

      &:hover {
        fill: red !important;
        opacity: .8;
      }
    }

    .label-text {
      font-size: 0.6em;
    }

    polyline {
      opacity: .3;
      stroke: black;
      stroke-width: 2px;
      fill: none;
    }

    .overlay {
      fill: none;
      pointer-events: all;
    }

    .pie-tooltip {
      .frame {
        fill: #cccccc;
        stroke: #b3b3b3;
        stroke-width: 2px;
        opacity: 0.8
      }

      .label-text {
        font-size: 0.48em;
      }

      tspan {
        font-size: 0.4em;
      }

      tspan.value {
        font-weight: bold;
      }
    }
  }
</style>
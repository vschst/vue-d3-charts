<template>
  <svg
    :viewBox="viewBox"
    preserveAspectRatio="xMinYMin meet"
    class="simple-pie-chart"
  >
    <g :transform="stageTransform">
      <g ref="slices" />
      <g ref="labels" />
      <g ref="lines" />
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
                    width: 335,
                    height: 130
                })
            },
            data: {
                type: Array,
                default: () => [],
            },
            colors: {
                type: Array,
                default: () => ['#20d6e9', '#ff0078', '#fff200', '#a42eff']
            }
        },
        data() {
            return {
                colorScale: this.getColorScale()
            }
        },
        computed: {
            viewBox() {
                return `0 0 ${this.layout.width} ${this.layout.height}`
            },
            stageTransform() {
                return `translate(${this.layout.width / 2},${this.layout.height / 2})`
            },
            radius() {
                return Math.min(this.layout.width, this.layout.height) / 2
            },
            sumValue() {
                return this.data.reduce((sum, d) => sum + d.value, 0)
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
                return d3.arc().outerRadius(this.radius * 0.7).innerRadius(this.radius * 0.35)
            },
            outerArc() {
                return d3.arc().innerRadius(this.radius * 0.77).outerRadius(this.radius * 0.77)
            },
            getColorScale() {
                return d3.scaleOrdinal().domain(Object.keys(this.data)).range(this.colors)
            },
            drawSlices() {
                const $slices = d3.select(this.$refs.slices)
                const $slice = $slices.selectAll('.slice').data(this.pie(this.data))
                    .enter()
                    .append('path')
                    .attr('d', this.arc())
                    .style('fill', (d, i) => this.colorScale(i))
                    .attr('class', 'slice')

                $slice.exit().remove()
            },
            midAngle(d) {
                return d.startAngle + (d.endAngle - d.startAngle) / 2
            },
            itemVisibility(d) {
                return ((d.value / this.sumValue) > 0.05) ? 'visible' : 'hidden'
            },
            drawLabels() {
                const $labels = d3.select(this.$refs.labels)
                const $text = $labels.selectAll('text').data(this.pie(this.data))

                $text.enter()
                    .append('text')
                    .attr('dy', '.35em')
                    .text(d => d.data.label)
                    .attr('transform', d => {
                        const pos = this.outerArc().centroid(d)

                        pos[0] = this.radius * ((this.midAngle(d) < Math.PI) ? 1 : -1)

                        return `translate(${pos})`
                    })
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
            }
        }
    }
</script>

<style lang="scss">
    .simple-pie-chart {
        width: 100%;

        path.slice {
            stroke-width: 2px;
        }

        text {
            font-size: 50%;
        }

        polyline {
            opacity: .3;
            stroke: black;
            stroke-width: 1px;
            fill: none;
        }
    }
</style>
<template>
  <svg
    :viewBox="viewBox" 
    preserveAspectRatio="xMinYMin meet" 
    class="simple-line-chart">
    <defs ref="defs"/>
    <g :transform="stageTransform">
      <g 
        ref="dateAxis"
        :transform="dateAxisTransform" 
        class="date-axis axis"/>
      <g 
        ref="valueAxis" 
        class="value-axis axis"/>
      <g 
        ref="line"/>
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
                    height: 130,
                    margin: {
                        left: 43,
                        right: 0,
                        top: 20,
                        bottom: 30
                    }
                }),
            },
            data: {
                type: Array,
                default: () => [],
            },
            colors: {
                type: Array,
                default: () => ['#C7EB3A', '#C7EB3A', '#90E475', '#58DDAF', '#20D6E9', '#41ACEF', '#6282F4', '#8358FA', '#A42EFF']
            },
            gradientId: {
                type: String,
                default: 'simple-line-gradient'
            },
            valueTick: {
                type: Object,
                default: () => ({count: 5, format: d => d})
            },
            area: {
                type: Boolean,
                default: false
            }
        },
        data() {
            return {
                scale: {
                    date: this.getDateScale(),
                    value: this.getValueScale(),
                    colors: this.getColorsScale()
                }
            }
        },
        computed: {
            padded() {
                const width = this.layout.width - this.layout.margin.left - this.layout.margin.right
                const height = this.layout.height - this.layout.margin.top - this.layout.margin.bottom

                return {width, height}
            },
            viewBox() {
                return `0 0 ${this.layout.width} ${this.layout.height}`
            },
            stageTransform() {
                return `translate(${this.layout.margin.left},${this.layout.margin.top})`
            },
            dateAxisTransform() {
                return `translate(0,${this.padded.height})`
            }
        },
        created() {
            this.updateScale()
        },
        mounted() {
            this.drawAxis('dateAxis')
            this.drawAxis('valueAxis')

            this.drawGradient()

            const $line = d3.select(this.$refs.line)

            if (this.area) {
                this.drawArea($line)
            }
            else {
                this.drawLine($line)
            }
        },
        methods: {
            parseTime: d3.timeParse('%Y-%m-%d'),
            getDateScale() {
                return d3.scaleTime().domain(d3.extent(this.data, d => this.parseTime(d.date)))
            },
            getValueScale() {
                return d3.scaleLinear().domain(d3.extent(this.data, d => d.value))
            },
            updateScale() {
                this.scale.date.range([0, this.padded.width])
                this.scale.value.range([this.padded.height, 0])
            },
            getColorsScale() {
                return d3.scaleBand().range([0, 100]).domain(this.colors)
            },
            drawGradient() {
                const $defs = d3.select(this.$refs.defs)
                const $linearGradient = $defs.append('linearGradient')
                    .attr('id', this.gradientId)
                    .attr('x1', '0%').attr('y1', '0%')
                    .attr('x2', '0%').attr('y2', '100%')
                    .selectAll('stop')
                    .data(this.colors)

                $linearGradient.enter()
                    .append('stop')
                    .attr('offset', d => `${this.scale.colors(d)}%`)
                    .attr('stop-color', d => d)
            },
            drawAxis(ref) {
                const $axis = d3.select(this.$refs[ref])
                const axisGenerator = {
                    dateAxis: d3.axisBottom(this.scale.date).ticks(d3.timeMonth).tickFormat(d3.timeFormat('%b')),
                    valueAxis: d3.axisLeft(this.scale.value).ticks(this.valueTick.count).tickFormat(this.valueTick.format)
                }

                $axis.call(axisGenerator[ref])
            },
            drawArea($line) {
                const area = d3.area().x(d => this.scale.date(this.parseTime(d.date))).y0(this.padded.height).y1(d => this.scale.value(d.value)).curve(d3.curveCardinal)

                $line.append('path')
                    .datum(this.data)
                    .attr('class', 'area')
                    .attr('d', area)
                    .attr('fill', `url(#${this.gradientId})`)
            },
            drawLine($line) {
                const line = d3.line().x(d => this.scale.date(this.parseTime(d.date))).y(d => this.scale.value(d.value)).curve(d3.curveCardinal)

                $line.append('path')
                    .datum(this.data)
                    .attr('class', 'line')
                    .attr('d', line)
                    .attr('stroke', `url(#${this.gradientId})`)
            }
        }
    }
</script>

<style lang="scss">
    .simple-line-chart {
        width: 100%;

        .axis {
          text {
            font-size: 0.6em;
            font-weight: 400;
          }

          path, line {
            fill: none;
            stroke: #000;
            shape-rendering: crispEdges;
          }
        }

        .date-axis.axis {
          path {
            display: none;
          }
        }

        .value-axis.axis {
          path {
            display: none;
          }
        }

        .line {
            fill: none;
            stroke-width: 2px;
        }

        .area {
            stroke-width: 5px;
        }
    }
</style>
<template>
  <svg 
    :viewBox="viewBox" 
    preserveAspectRatio="xMinYMin meet">
    <g :transform="stageTransform">
      <g 
        ref="dateAxis" 
        :transform="dateAxisTransform"
        class="date-axis axis"/>
      <g 
        ref="valueAxis"
        class="value-axis axis"/>
      <g ref="band"/>
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
                        left: 30,
                        right: 0,
                        top: 20,
                        bottom: 20
                    }
                })
            },
            data: {
                type: Array,
                default: () => [],
            },
            colors: {
                type: Array,
                default: () => ['#FFF200', '#C7EB3A', '#90E475', '#58DDAF', '#20D6E9', '#41ACEF', '#6282F4', '#8358FA', '#A42EFF']
            },
            valueTick: {
                type: Object,
                default: () => ({
                    count: 5, format: d => d
                })
            },
            titleFormat: {
                type: Function,
                default: d => d.value
            }
        },
        data() {
            return {
                scale: {
                    date: this.getDateScale(),
                    band: this.getBandScale(),
                    value: this.getValueScale(),
                    color: this.getColorScale()
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
            },
        },
        created() {
            this.updateScale()
        },
        mounted() {
            this.drawAxis('dateAxis')
            this.drawAxis('valueAxis')
            this.drawBand()
        },
        methods: {
            parseTime: d3.timeParse('%Y-%m-%d'),
            getDateScale() {
                return d3.scaleTime().domain(d3.extent(this.data, d => this.parseTime(d.date)))
            },
            getBandScale() {
                return d3.scaleBand().domain(this.data.map(d => d.date)).padding(0.3).round(true)
            },
            getValueScale() {
                return  d3.scaleLinear().domain([0, d3.max(this.data, d => d.value)])
            },
            getColorScale() {
                return d3.scaleQuantile().range(this.colors).domain(d3.extent(this.data, d => d.value))
            },
            updateScale() {
                this.scale.date.range([0, this.padded.width])
                this.scale.band.rangeRound([0, this.padded.width])
                this.scale.value.range([this.padded.height, 0])
            },
            drawAxis(ref) {
                const $axis = d3.select(this.$refs[ref])
                const axisGenerator = {
                    dateAxis: d3.axisBottom(this.scale.date).ticks(d3.timeMonth).tickFormat(d3.timeFormat('%b')),
                    valueAxis: d3.axisLeft(this.scale.value).ticks(this.valueTick.count).tickFormat(this.valueTick.format)
                }

                $axis.call(axisGenerator[ref])
            },
            drawBand() {
                const $band = d3.select(this.$refs.band).selectAll('bar').data(this.data)
                    .enter()
                    .append('rect')
                    .attr('x', d => this.scale.band(d.date))
                    .attr('width', this.scale.band.bandwidth())
                    .attr('y', d => this.scale.value(d.value))
                    .attr('height', d => (this.padded.height - this.scale.value(d.value)))
                    .attr('class', 'bar')
                    .style('fill', this.colors[0])

                $band.append('title').text(this.titleFormat)

                $band.transition()
                    .duration(1500)
                    .style('fill', d => this.scale.color(d.value))
            }
        }
    }
</script>

<style lang="scss">
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

  .bar {
    &:hover {
      fill: brown !important;
    }
  }
</style>
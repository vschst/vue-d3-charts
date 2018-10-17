<template>
    <svg preserveAspectRatio="xMinYMin meet" :viewBox="viewBox">
        <g ref="stage" v-bind:style="stageStyle">
            <g ref="xAxis" v-bind:style="xAxisStyle"></g>
            <g ref="yAxis" v-bind:style="yAxisStyle"></g>
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
                        bottom: 60
                    }
                }),
            },
            data: {
                type: Array,
                default: () => [],
            },
            colors: {
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
            }
        },
        data() {
            return {
                scale: {
                    x: this.getScaleX(),
                    y: this.getScaleY()
                }
            }
        },
        mounted() {
            this.drawAxis('xAxis')
            this.drawAxis('yAxis')
            this.drawLines()
        },
        methods: {
            initScale(axis) {
                switch (this.scaleType[axis]) {
                    case 'time':
                        return d3.scaleTime()
                    case 'linear':
                        return d3.scaleLinear()
                }
            },
            extents(key) {
                const min = d3.min(this.data, data => d3.min(data, d => this.parse[key](d[key])))
                const max = d3.max(this.data, data => d3.max(data, d => this.parse[key](d[key])))

                return [min, max]
            },
            getScaleX() {
                return this.initScale('x').range([0, this.layout.width]).domain(this.extents('x'))
            },
            getScaleY() {
                return this.initScale('y').range([this.layout.height, 0]).domain(this.extents('y'))
            },
            drawAxis(ref) {
                const $axis = d3.select(this.$refs[ref])
                const axisGenerator = {
                    xAxis: d3.axisBottom(this.scale.x).ticks(this.ticks.x.count).tickFormat(this.ticks.x.format),
                    yAxis: d3.axisLeft(this.scale.y).ticks(this.ticks.y.count).tickFormat(this.ticks.y.format)
                }

                $axis.call(axisGenerator[ref])
            },
            drawLines() {
                const $stage = d3.select(this.$refs.stage)
                const line = d3.line().x(d => this.scale.x(this.parse.x(d.x))).y(d => this.scale.y(this.parse.y(d.y))).curve(d3.curveBasis)
                const $line = $stage.selectAll('path').data(this.data)

                $line.enter()
                    .append('path')
                    .attr('d', line)
                    .style('fill', 'none')
                    .style('stroke', (d, i) => this.colors[i])
                    .style('stroke-width', 2)
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
            stageStyle() {
                return {'transform': `translate(${this.layout.margin.left}px,${this.layout.margin.top}px)`}
            },
            xAxisStyle() {
                return {'transform': `translate(0px,${this.layout.height}px)`}
            },
            yAxisStyle() {
                return {'transform': `translate(0px,0px)`}
            }
        }
    }
</script>
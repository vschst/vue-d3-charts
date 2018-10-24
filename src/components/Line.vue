<template>
    <svg preserveAspectRatio="xMinYMin meet" :viewBox="viewBox">
        <g :transform="stageTransform">
            <g ref="xAxis" :transform="xAxisTransform"></g>
            <g ref="yAxis"></g>
            <g ref="lines" :transform="linesTransform"></g>
            <g ref="legend"></g>
            <g ref="tooltip" v-show="tooltip.show">
                <line x1="0" y1="0" x2="0" :y2="layout.height" class="hover-line"></line>
                <g :transform="tooltipFrameTransform(layout.height)" class="x-frame">
                    <rect :width="layout.tooltip.width" :height="layout.tooltip.height"></rect>
                    <text :dx="layout.tooltip.text.dy" :dy="layout.tooltip.text.dy">{{getTooltipFrameBottomText(tooltip.index)}}</text>
                </g>
            </g>
            <rect ref="overlay" :width="layout.width" :height="layout.height" class="overlay"></rect>
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
                    },
                    tooltip: {
                        width: 100,
                        height: 20,
                        padding: 2,
                        text: {
                            dx: 30,
                            dy: 15
                        }
                    },
                    legend: {
                        width: 150,
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
                default: () => []
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
                    show: true,
                    x: 0,
                    index: null
                }
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
        watch: {
            'tooltip.index': function(val) {
                const $tooltip = d3.select(this.$refs.tooltip)
                    .attr('transform', `translate(${this.tooltip.x},0)`)

                const $pointFrame = $tooltip.selectAll('.point-frame')
                    .data(this.lines)
                    .attr('transform', d => {
                        const offset = this.layout.margin.top + this.layout.tooltip.height

                        return this.tooltipFrameTransform(this.scale.y(this.parse.y(this.data[val][d])) - offset, -1)
                    })

                $pointFrame.select('text')
                    .text(d => this.data[val][d])
            }
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

                $axis.call(axisGenerator[ref])
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
                        .style('fill-opacity', 0.25)
                        .style('stroke', 'none')
                }

                //  Drawing points
                $series.style('fill', '#fff')
                    .style('stroke', (d, i) => this.scale.colors(i))
                    .selectAll('.point')
                    .data(d => this.getPoints(d))
                    .enter()
                    .append('circle')
                    .attr('class', 'point')
                    .attr('r', 3)
                    .attr('cx', d => this.scale.x((this.parse.x(d.x))))
                    .attr('cy', d => this.scale.y((this.parse.y(d.y))))
                    .style('stroke-width', 2)
            },
            drawLegend() {
                const $legend = d3.select(this.$refs.legend)
                const legendX = this.layout.width - this.layout.legend.width

                $legend.append('rect')
                    .attr('x', legendX)
                    .attr('width', this.layout.legend.width)
                    .attr('height', this.layout.legend.itemSize * this.names.length)
                    .attr('fill', 'white')

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
                    .attr('fill',this.scale.colors)

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

                const $pointFrame = d3.select(this.$refs.tooltip)
                    .selectAll('.point-frame')
                    .data(this.lines)
                    .enter()
                    .append('g')
                    .attr('class', 'point-frame')

                $pointFrame.append('rect')
                    .attr('width', this.layout.tooltip.width)
                    .attr('height', this.layout.tooltip.height)

                $pointFrame.append('text')
                    .attr('dx', this.layout.tooltip.text.dx)
                    .attr('dy', this.layout.tooltip.text.dy)
            },
            mouseover() {
                this.tooltip.show = false
            },
            mousemove() {
                const scales = this.data.map(d => this.scale.x(this.parse.x(d.x)))
                const index = d3.bisect(scales, d3.mouse(this.$el)[0], 1) - 1
                const data = this.data[index]

                this.tooltip.x = this.scale.x(this.parse.x(data.x))
                this.tooltip.index = index
                this.tooltip.show = true
            },
            tooltipFrameTransform(height, reflect = 1) {
                return `translate(${this.layout.tooltip.padding},${height + (reflect * this.layout.tooltip.padding)})`
            },
            getTooltipFrameBottomText(index) {
                return (index !== null) ? this.data[index].x : ''
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
            }
        }
    }
</script>

<style>
    .overlay {
        fill: none;
        pointer-events: all;
    }

    .hover-line {
        stroke: #B74779;
        stroke-width: 2px;
        stroke-dasharray: 3,3;
    }

    .x-frame rect,
    .point-frame rect {
        fill: gray;
        opacity: 0.95
    }
</style>
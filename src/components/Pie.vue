<template>
    <svg preserveAspectRatio="xMinYMin meet" :viewBox="viewBox">
        <g :transform="stageTransform">
            <g ref="slices"></g>
            <g ref="labels"></g>
            <g ref="lines"></g>
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
                    width: 670,
                    height: 240
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
        mounted() {
            this.drawSlices()
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
                return d3.scaleOrdinal().domain(Object.keys(this.data)).range(this.colors)
            },
            drawSlices() {
                const $slices = d3.select(this.$refs.slices)
                const $slice = $slices.selectAll('.slice').data(this.pie(this.data))
                    .enter()
                    .append('path')
                    .attr('d', this.arc())
                    .style('fill', d => this.colorScale(d))
                    .attr('class', 'slice')

                $slice.exit().remove()
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
                return Math.min(this.width, this.height) / 2
            }
        }
    }
</script>
<template>
    <svg preserveAspectRatio="xMinYMin meet" :viewBox="viewBox">
        <g ref="stage" :transform="stageTransform">
            <g ref="dayLabels" :transform="dayLabelsTransform"></g>
            <g ref="timeLabels" :transform="timeLabelsTransform"></g>
            <g ref="cards" :transform="cardsTransform"></g>
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
                    height: 250,
                    margin: {
                        left: 40,
                        right: 0,
                        top: 50,
                        bottom: 30
                    }
                })
            },
            data: {
                type: Array,
                default: () => [],
            },
            colors: {
                type: Array,
                default: () => ['#ffffd9', '#edf8b1', '#c7e9b4', '#7fcdbb', '#41b6c4', '#1d91c0', '#225ea8', '#253494', '#081d58']
            }
        },
        data() {
            return {
                days: ['Mo', 'Tu', 'We', 'Th', 'Fr', 'Sa', 'Su'],
                times: ['1a', '2a', '3a', '4a', '5a', '6a', '7a', '8a', '9a', '10a', '11a', '12a', '1p', '2p', '3p', '4p', '5p', '6p', '7p', '8p', '9p', '10p', '11p', '12p'],
                colorScale: this.getColorScale()
            }
        },
        mounted() {
            this.drawDayLabels()
            this.drawTimeLabels()
            this.drawCards()
        },
        methods: {
            getColorScale() {
                return d3.scaleQuantile().domain([0, d3.max(this.data, d => d.value)]).range(this.colors)
            },
            drawDayLabels() {
                const $dayLabel = d3.select(this.$refs.dayLabels).selectAll('.day-label').data(this.days)

                $dayLabel.enter()
                    .append('text')
                    .text(d => d)
                    .attr('x', 0)
                    .attr('y', (d, i) => i * this.gridSize)
                    .attr('class', (d, i) => (i >= 0 && i <= 4) ? 'day-label mono workweek' : 'day-label mono')
            },
            drawTimeLabels() {
                const $timeLabel = d3.select(this.$refs.timeLabels).selectAll('.time-label').data(this.times)

                $timeLabel.enter()
                    .append('text')
                    .text(d => d)
                    .attr('x', (d, i) => i * this.gridSize)
                    .attr('y', 0)
                    .attr('class', (d, i) => (i >= 8 && i <= 17) ? 'time-label mono worktime' : 'time-label mono')
            },
            drawCards() {
                const $cards = d3.select(this.$refs.cards).selectAll('.hour').data(this.data)
                const $card = $cards.enter()
                    .append('rect')
                    .attr('class', 'hour bordered')
                    .attr('x', d => (parseInt(d.hour) - 1) * this.gridSize)
                    .attr('y', d => (parseInt(d.day) - 1) * this.gridSize)
                    .attr('rx', 4)
                    .attr('ry', 4)
                    .attr('width', this.gridSize)
                    .attr('height', this.gridSize)
                    .style('fill', this.colors[0])

                $card.append('title')
                    .text(d => d.value)

                $card.transition()
                    .duration(1500)
                    .style('fill', d => this.colorScale(d.value))

                $cards.exit().remove()
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
            gridSize() {
                return Math.floor(this.layout.width / 24)
            },
            dayLabelsTransform() {
                return `translate(0,${this.gridSize})`
            },
            timeLabelsTransform() {
                return `translate(${this.gridSize},0)`
            },
            cardsTransform() {
                return `translate(${this.gridSize / 4.5},${this.gridSize / 3})`
            }
        }
    }
</script>

<style>
    .day-label,
    .time-label {
        text-anchor: end;
    }

    text.mono {
        font-size: 9pt;
        font-family: Consolas, Courier;
        fill: #aaa;
    }

    text.workweek,
    text.worktime {
        fill: #000;
    }

    rect.bordered {
        stroke: #e6e6e6;
        stroke-width: 2px;
    }
</style>
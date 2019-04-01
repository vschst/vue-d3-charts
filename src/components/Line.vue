<template>
  <line-chart
    v-if="loaded"
    :data="data"
    :lines="lines"
    :names="names"
    :tooltip-format="tooltipFormat"
    :axis-legend="axisLegend"
    :name="name"
  /><b v-else>Line chart loading...</b>
</template>

<script>
    import {timeParse, timeFormat} from 'd3-time-format'
    import {csv} from 'd3-fetch'
    import {format} from 'd3-format'

    import LineChart from '@/components/Charts/Line'

    export default {
        components: {
            LineChart
        },
        data() {
            return {
                loaded: false,
                lines: [],
                data: [],
                ticks: {
                    x: {count: 10, format: timeFormat('%B, %Y')},
                    y: {count: 30, format: d => d}
                },
                names: ['Block count', 'Average block input count'],
                tooltipFormat: {
                    x: timeFormat('%B, %Y'),
                    y: [d => d, format(".2f")]
                },
                axisLegend: {
                    x: 'Month',
                    y: ['Block count', 'Input count']
                },
                name: 'Bitcoin block count and average block input count by month',
            }
        },
        created() {
            this.loadData()
        },
        methods: {
            async loadData() {
                const data = await csv('line.csv', d => ({
                    x: d['month'],
                    'count()': Number(d['count()']),
                    'avg(input_count)': Number(d['avg(input_count)'])
                }))

                this.lines = data.columns.slice(1)
                this.data = data.slice()
                this.loaded = true
            }
        }
    }
</script>
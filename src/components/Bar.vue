<template>
  <bar-chart
    v-if="loaded"
    :data="data"
    :columns="columns"
    :colors="colors"
    :band="band"
    :names="names"
    :options="options"
    :tooltip-format="tooltipFormat"
    :axis-legend="axisLegend"
    :name="name"
  /><b v-else>Bar chart loading...</b>
</template>

<script>
    import {timeFormat} from 'd3-time-format'
    import {csv} from 'd3-fetch'
    import {format} from 'd3-format'

    import BarChart from '@/components/Charts/Bar'

    export default {
        components: {
            BarChart
        },
        data() {
            return {
                loaded: false,
                columns: [],
                data: [],
                colors: ['#98abc5', '#8a89a6'],
                band: {
                    ticks: 10,
                    format: timeFormat('%B, %Y')
                },
                names: ['Median gas used', 'Average gas price'],
                options: {
                    scaleLog: false,
                    legend: true,
                    tooltip: true,
                    axisCount: 2
                },
                tooltipFormat: {
                    band: timeFormat('%B, %Y'),
                    value: [d => d, format(".2f")]
                },
                axisLegend: {
                    x: 'Month',
                    y: ['Gas count', 'Gwei']
                },
                name: 'Ethereum median gas used and average gas price by month'
            }
        },
        created() {
            this.loadData()
        },
        methods: {
            async loadData() {
                const data = await csv('bar.csv', d => ({
                    name: d['month'],
                    'median(gas_used)': Number(d['median(gas_used)']),
                    'avg(gas_price)': Number(d['avg(gas_price)']) / 1000000000
                }))

                this.columns = data.columns.slice(1)
                this.data = data.slice()
                this.loaded = true
            }
        }
    }
</script>
<template>
  <scatter-chart
    v-if="loaded"
    :data="data"
    :lines="lines"
    :axis-legend="axisLegend"
    :name="name"
  />
  <b v-else>Scatter chart loading...</b>
</template>

<script>
    import {csv} from 'd3-fetch'

    import ScatterChart from '@/components/Charts/Scatter'

    export default {
        components: {
            ScatterChart
        },
        data() {
            return {
                loaded: false,
                lines: [],
                data: [],
                axisLegend: {
                    x: 'Block id',
                    y: ['Transactions count']
                },
                name: 'Litecoin transactions in block over last month'
            }
        },
        created() {
            this.loadData()
        },
        methods: {
            async loadData() {
                const data = await csv('scatter.csv', d => ({
                    x: Number(d['block_id']),
                    'count()': Number(d['count()'])
                }))

                this.lines = data.columns.slice(1)
                this.data = data.slice()
                this.loaded = true
            }
        }
    }
</script>
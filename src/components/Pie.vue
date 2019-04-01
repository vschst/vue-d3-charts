<template>
  <pie-chart
    v-if="loaded"
    :data="data"
    :argument="argument"
    :name="name"
  /><b v-else>Pie chart loading...</b>
</template>

<script>
    import {csv} from 'd3-fetch'

    import PieChart from '@/components/Charts/Pie'

    export default {
        components: {
            PieChart
        },
        data() {
            return {
                loaded: false,
                data: [],
                argument: 'Count',
                name: 'Bitcoin hashrate distribution for 2018 year'
            }
        },
        created() {
            this.loadData()
        },
        methods: {
            async loadData() {
                const data = await csv('pie.csv', d => ({
                    label: d['guessed_miner'],
                    value: Number(d['count()'])
                }))

                this.data = data.slice()
                this.loaded = true
            }
        }
    }
</script>
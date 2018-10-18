<template>
  <div>
    <h1>Multi-series line chart</h1>
    <line-chart :data="chartsData" :colors="colors" :names="names" :area="true"/>
  </div>
</template>

<script>
    import LineChart from './components/Line.vue'

    export default {
        name: 'app',
        data() {
            return {
                chartsData: [
                    [
                        {x: '2018-01-01', y: 3.4},
                        {x: '2018-02-01',  y: 3.1},
                        {x: '2018-03-01', y: 0.3},
                        {x: '2018-04-01', y: 0.4},
                        {x: '2018-05-01', y: 5.4},
                        {x: '2018-06-01', y: 7},
                        {x: '2018-07-01', y: 6.4}
                    ],
                    [
                        {x: '2018-01-01', y: 4.3},
                        {x: '2018-02-01',  y: 1.3},
                        {x: '2018-03-01', y: 3.0},
                        {x: '2018-04-01', y: 4},
                        {x: '2018-05-01', y: 4.5},
                        {x: '2018-06-01', y: 8},
                        {x: '2018-07-01', y: 4.6}
                    ]
                ],
                colors: ['#80b081', '#b08097'],
                names: ['Chart #1', 'Chart #2']
            }
        },
        components: {
            LineChart
        }
    }
</script>

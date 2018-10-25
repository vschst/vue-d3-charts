<template>
  <div>
    <h1>Multi-series line chart</h1>
    <line-chart :data="line.data" :lines="line.lines" :colors="line.colors" :names="line.names"/>
    <h1>Grouped Bar Chart</h1>
    <bar-chart :data="bar.data" :columns="bar.columns" :colors="bar.colors"/>
    <h1>Day / Hour Heatmap</h1>
    <heatmap :data="heatmap.data"/>
    <h1>Pie chart with labels</h1>
    <pie-chart :data="pie.data"/>
  </div>
</template>

<script>
    import LineChart from './components/Line'
    import BarChart from './components/Bar'
    import Heatmap from './components/Heatmap'
    import PieChart from './components/Pie'

    export default {
        name: 'app',
        data() {
            return {
                line: {
                    data: [
                        {x: '2017-10-01', y1: 2.2, y2: 1.1},
                        {x: '2017-11-01', y1: 2.5, y2: 1.4},
                        {x: '2017-12-01', y1: 2.9, y2: 0.9},
                        {x: '2018-01-01', y1: 3.4, y2: 0.3},
                        {x: '2018-02-01', y1: 3.1, y2: 1.3},
                        {x: '2018-03-01', y1: 0.3, y2: 3.0},
                        {x: '2018-04-01', y1: 0.4, y2: 4},
                        {x: '2018-05-01', y1: 5.4, y2: 4.5},
                        {x: '2018-06-01', y1: 6.3, y2: 5.6},
                        {x: '2018-07-01', y1: 6.2, y2: 4.6},
                        {x: '2018-08-02', y1: 2.1, y2: 1.35},
                        {x: '2018-09-01', y1: 3.1, y2: 3.4},
                        {x: '2018-10-01', y1: 3.0, y2: 3.5}
                    ],
                    lines: ['y1', 'y2'],
                    colors: ['#80b081', '#b08097'],
                    names: ['Chart #1', 'Chart #2']
                },
                bar: {
                    data: [
                        {name: 'CA', low: 5555, medium: 33, high: 333},
                        {name: 'TX', low: 1234, medium: 666, high: 555},
                        {name: 'IL', low: 225, medium: 523, high: 650},
                        {name: 'NY', low: 9999, medium: 862, high: 230}
                    ],
                    columns: ['low', 'medium', 'high'],
                    colors: ['#98abc5', '#8a89a6', '#7b6888']
                },
                heatmap: {
                    data: [
                        {day: '01', hour: '01', value: 12},
                        {day: '01', hour: '02', value: 44},
                        {day: '01', hour: '09', value: 27},
                        {day: '01', hour: '13', value: 37},
                        {day: '01', hour: '14', value: 47},
                        {day: '01', hour: '24', value: 77},
                        {day: '02', hour: '01', value: 55},
                        {day: '07', hour: '01', value: 14},
                        {day: '07', hour: '08', value: 34},
                        {day: '07', hour: '14', value: 54},
                        {day: '07', hour: '24', value: 84}
                    ]
                },
                pie: {
                    data: [
                        {label: 'CA', value: 555},
                        {label: 'LX', value: 383},
                        {label: 'IL', value: 225},
                        {label: 'NY', value: 123}
                    ]
                }
            }
        },
        components: {
            LineChart,
            BarChart,
            Heatmap,
            PieChart
        }
    }
</script>

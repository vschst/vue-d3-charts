<template>
    <g class="legend" v-bind:style="legendStyle">
        <rect width="8em" :height="`${yItem(1) + 0.5}em`" rx="0.1em" ry="0.1em"></rect>
        <circle v-for="(cd, i) in colors" :key="i" :cy="`${(yItem(i) - 0.3)}em`" cx="1em" r="0.3em" v-bind:style="{'fill': cd}"></circle>
        <text v-for="(nd, i) in names" :key="i" :y="`${yItem(i)}em`" x="2em">{{nd}}</text>
    </g>
</template>

<script>
    export default {
        props: ['width', 'height', 'colors', 'names'],

        methods: {
            yItem(index) {
                return 1 + 1.2 * index
            }
        },
        computed: {
            legendStyle() {
                const x = this.width / 2.5
                const y = this.height / 30

                return {'transform': `translate(${x}px,${y}px)`}
            }
        }
    }
</script>

<style>
    .legend rect {
        fill:white;
        fill-opacity: 0.8;
    }
</style>
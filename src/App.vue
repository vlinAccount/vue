<template>
  <div id="app">
    <!-- <img alt="Vue logo" src="./assets/logo.png">
    <HelloWorld msg="Welcome to Your Vue.js App"/> -->
    <FlowChart></FlowChart>
  </div>
</template>

<script>
import HelloWorld from './components/HelloWorld.vue'
import FlowChart from './components/flow-chart/index'

export default {
  name: 'App',
  components: {
    HelloWorld, FlowChart
  }
}
</script>

<style>
body {
  margin: 0;
}

</style>

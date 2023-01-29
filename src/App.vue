<template>
  <div id="app">
    <Segmentation msg="Welcome to Your Vue.js App" />
  </div>
</template>

<script>
import Segmentation from "./components/Segmentation.vue";

export default {
  name: "App",

  components: {
    Segmentation,
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
</style>

<template>
  <div id="app">
    <zk-sync />
  </div>
</template>

<script>
import ZkSync from './components/ZkSync.vue';

export default {
  name: 'App',
  components: {
    ZkSync,
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
  margin-top: 60px;
}
</style>

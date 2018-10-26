<template>
    <div id="app" class="nt-c-app">
        <img
            src="./assets/logo.png"
            width="100"
            height="100"
            alt="erer">
        <router-view />
    </div>
</template>

<script>
export default {
  name: 'App',
  data () {
    return {
      aasd: 'sdfsd'
    }
  }
}
</script>

<style lang="scss">
@import '/assets/styles/test-styles.scss';

.nt-c-app {
  color: #2c3e50;
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -moz-osx-font-smoothing: grayscale;
  -webkit-font-smoothing: antialiased;
  margin-top: 60;
  text-align: center;
}
</style>

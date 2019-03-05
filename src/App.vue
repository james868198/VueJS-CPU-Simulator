<template lang="pug">
  #app
    //- img(src="./assets/logo.png")
    router-view
</template>

<script>
export default {
  name: 'App'
}
</script>

<style>
#app {
  width: 100vw;
  height: 100vh;
  margin: 0 auto;
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
</style>

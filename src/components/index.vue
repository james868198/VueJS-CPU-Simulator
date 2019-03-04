<template lang="pug">
  .simulator
    h6 23r23r
      | {{inputs}}
    //- <h6> {{inputs}} </h6>
</template>

<script>

// import Parse from '../../utils/parse.js'
// import sample from '../../static/sample.json'
const axios = require('axios')

export default {
  name: 'simulator',
  data () {
    return {
      msg: 'Welcome to Your Vue.js App',
      inputs: []
    }
  },
  mounted () {
    // console.log(tt)
    // Parse.txtParse(this.$route.path + 'static/test.txt')
    // Parse.txtParse('../../static/tt.json')
    axios.get('../../static/sample.json').then(response => (
      this.inputs.push(response))
    )
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss">
/* h1, h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
} */
</style>

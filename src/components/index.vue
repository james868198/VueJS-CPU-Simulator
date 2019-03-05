<template lang="pug">
.simulator
  .simulator-container
    .simulator-top
      .simulator-top-container
        .simulator-top-left
          .simulator-top-left-container
            b-button.action-btn(variant="danger" size="lg") Input
            b-button.action-btn(variant="danger" size="lg"  v-on:click="run") Run
            b-button.action-btn(variant="danger" size="lg"  v-on:click="pause") Pause
            b-button.action-btn(variant="danger" size="lg"  v-on:click="stop")  Stop
        .simulator-top-right
          .simulator-top-right-container
            .strategy(v-for="input in inputs")
              .strategy-container
                .name
                  | {{input.strategy}}
                  | {{cycleTime}}
                .simulation
                  .simulation-container
                    .simulation-container-inner
                      .cycle(v-for="cycle in cycles")
                        | {{cycle}}
    .simulator-bottom
      .simulator-bottom-container
        .simulator-table(v-if="simulationTable")
          b-table(striped hover :items="simulationTable.threads", :fields="fields")
</template>

<script>
const axios = require('axios')

export default {
  name: 'simulator',
  data () {
    return {
      msg: 'Welcome to Your Vue.js App',
      inputs: [],
      cycles: [],
      simulation: null,
      cycleTime: 0,
      status: 'stop',
      simulationTable: null,
      fields: ['id', 'state', 'priority', 'arriveTime', 'burstTime', 'waitingTime', 'turnAroundTime', 'execTime']
    }
  },
  mounted () {
    // console.log(tt)
    // Parse.txtParse(this.$route.path + 'static/test.txt')
    // Parse.txtParse('../../static/tt.json')
    axios.get('../../static/test1.json').then((response) => {
      this.inputs.push(response.data)
      this.simulationTable = this.inputs[0]
      this.simulationTable.threads.forEach((thread, id) => {
        thread['id'] = id
        thread['waitingTime'] = 0
        thread['turnAroundTime'] = 0
        thread['execTime'] = 0
      })
    })
  },
  methods: {
    moveToCPU (ct) {
      if (!this.inputs[0].simulations) {
        console.log('[warning]no simulation')
        return
      }
      if (this.inputs[0].simulations.length > this.cycleTime) {
        this.cycles.push(this.inputs[0].simulations[ct])
        this.cycleTime++
      } else {
        clearInterval(this.simulation)
        this.status = 'stop'
      }
    },

    // button events
    // input () {

    // },
    run () {
      console.log('run test')
      if (this.status !== 'run') {
        this.simulation = setInterval(this.moveToCPU, 100, this.cycleTime)
        this.status = 'run'
      }
    },
    pause () {
      console.log('pause test')
      if (this.status === 'run') {
        clearInterval(this.simulation)
        this.status = 'pause'
      }
    },
    stop () {
      console.log('stop test')
      if (this.simulation) {
        clearInterval(this.simulation)
        this.simulation = null
        this.status = 'stop'
        this.cycleTime = 0
        this.cycles = []
      }
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss">
.simulator {
  position: relative;
  height: 100%;
  width: 100%;
  overflow: hidden;
  background-color: black;
  .simulator-container {
    position: relative;
    height: 100%;
    width: 100%;

    .simulator-top {
      position: relative;
      height: 40%;
      width: 100%;
      .simulator-top-container {
        position: relative;
        height: 100%;
        width: 100%;
        display: flex;
        flex-direction: row;
        .simulator-top-left {
          position: relative;
          height: 100%;
          width: 15%;
          display: inline-box;
          .simulator-top-left-container {
              position: relative;
              height: 100%;
              width: 100%;
              display: flex;
              flex-direction: column;
              .action-btn {
                display: inline-block;
                font-size: 16px;
                margin: 1em  1em;
                cursor: pointer;
              }
           }
        }

        .simulator-top-right {
          position: relative;
          height: 100%;
          width: 85%;
          display: inline-box;
          .simulator-top-right-container {
            position: relative;
            height: 100%;
            width: 100%;
            .strategy {
              position: relative;
              height: 100%;
              width: 100%;
              .strategy-container {
                position: relative;
                height: 100%;
                width: 100%;
                display: flex;
                flex-direction: row;
                .name {
                  background-color: white;
                  position: relative;
                  height: 100%;
                  width: 10%;
                  display: inline-block;
                  text-align: center;
                }
                .simulation {
                  background-color: green;
                  position: relative;
                  height: 100%;
                  width: 90%;
                  display: inline-block;

                  .simulation-container {
                    position: relative;
                    width: 36em;
                    overflow-x: scroll;
                    overflow-y: hidden;
                    .simulation-container-inner {
                      display: flex;
                      flex-direction: row;
                      .cycle {
                        background-color: yellow;
                        min-height: 2em;
                        min-width: 2em;
                        max-height: 2em;
                        max-width: 2em;
                        overflow: hidden;
                        display: inline-block;
                        margin-right:0.5em;
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
    .simulator-bottom {
      position: relative;
      height: 60%;
      width: 100%;
      overflow-y: scroll;
      overflow-x: hidden;
      .simulator-bottom-container {
        margin: 1em auto;
        position: relative;
        min-height: 20em;
        width: 95%;
        text-align: center;
        background-color: white;
        border-radius: 1em 1em;
        overflow: hidden;
        .simulator-table {
          position: relative;
          height: 60%;
          width: 100%;
        }
      }
    }
  }
}
</style>

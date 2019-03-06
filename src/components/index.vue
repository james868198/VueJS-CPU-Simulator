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
            .cycle-counter
              .cycle-counter-container
                .title
                  | Cycle Time
                .counter
                  | {{cycleTime}}
            .strategies
              .strategy(v-for="strategy in strategies")
                .strategy-container
                  .name
                    .name-container
                      | {{strategy.strategy}}
                  .simulation
                    .simulation-container
                      .simulation-container-inner
                        .cycle(v-for="thread in execThreads")
                          .vertical-center(v-if="thread>=0")
                            | t{{thread}}
                          .vertical-center(v-else)
                            | N/A
    .simulator-bottom
      .simulator-bottom-container
        .simulator-table(v-if="strategies[tableId]")
          b-table(striped hover :items="strategies[tableId].threads", :fields="fields")
</template>

<script>
const axios = require('axios')
const randomColor = require('randomcolor')

export default {
  name: 'simulator',
  data () {
    return {
      msg: 'Welcome to Your Vue.js App',
      strategies: [],
      execThreads: [],
      simulation: null,
      cycleTime: 0,
      status: 'stop',
      tableId: 0,
      fields: ['id', 'state', 'priority', 'arriveTime', 'burstTime', 'waitingTime', 'turnAroundTime', 'execTime']
    }
  },
  mounted () {
    // console.log(tt)
    // Parse.txtParse(this.$route.path + 'static/test.txt')
    // Parse.txtParse('../../static/tt.json')
    axios.get('../../static/test1.json').then((response) => {
      this.strategies.push(response.data)
      const color = randomColor({count: this.strategies[0].threads.length})
      // console.log(this.strategies, color)
      this.strategies[0].threads.forEach((thread, id) => {
        thread['id'] = id
        thread['state'] = 'waiting'
        thread['waitingTime'] = 0
        thread['turnAroundTime'] = 0
        thread['execTime'] = 0
        thread['color'] = color[id]
      })
    })
  },
  methods: {
    simulating (strategyId) {
      if (!this.strategies[strategyId]) {
        console.log('[warning] no strategy')
        return
      }
      if (!this.strategies[strategyId].simulations) {
        console.log('[warning] no simulation')
        return
      }
      const simulations = this.strategies[strategyId].simulations
      if (simulations.length <= this.cycleTime) {
        console.log('simulation stop')
        clearInterval(this.simulation)
        this.status = 'stop'
        return
      }
      // to ready
      simulations[this.cycleTime].moveToReadyList.forEach(threadId => {
        this.toReady(strategyId, threadId)
      })
      // to CPU
      if (simulations[this.cycleTime].moveToCPU >= 0) {
        // thread move to CPU
        this.execThreads.push(simulations[this.cycleTime].moveToCPU)
        this.toCPU(strategyId, simulations[this.cycleTime].moveToCPU)
      } else {
        // no change
        if (this.cycleTime === 0) {
          this.execThreads.push(-1)
        } else {
          this.execThreads.push(this.execThreads[this.cycleTime - 1])
        }
      }
      // to block
      if (simulations[this.cycleTime].moveToBlockList >= 0) {
        // thread move to CPU
        this.toBlock(strategyId, simulations[this.cycleTime].moveToBlockList)
      }
      // to block
      if (simulations[this.cycleTime].moveToFinishedList >= 0) {
        // thread move to CPU
        this.toFinish(strategyId, simulations[this.cycleTime].moveToFinishedList)
      }

      this.cycleTime++
    },
    // update status
    toReady (strategyId, threadId) {
      this.strategies[strategyId].threads[threadId].state = 'ready'
    },
    toCPU (strategyId, threadId) {
      this.strategies[strategyId].threads[threadId].state = 'processing'
    },
    toBlock (strategyId, threadId) {
      this.strategies[strategyId].threads[threadId].state = 'block'
    },
    toFinish (strategyId, threadId) {
      this.strategies[strategyId].threads[threadId].state = 'finish'
    },
    // button events
    // input () {

    // },
    run () {
      console.log('run test')
      if (this.status !== 'run') {
        this.simulation = setInterval(this.simulating, 100, 0)
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
        this.execThreads = []
      }
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss">
$margin-dist: 0.5em;
// $margin-dist1: 0.3em;
$border-radius-val: 0.1em;

.vertical-center {
  position: absolute;
  width: 100%;
  top:50%;
  transform: translateY(-50%);
  text-align: center;
}
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
            margin: $margin-dist;
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
            margin: $margin-dist;
            position: relative;
            height: 100%;
            width: 100%;
            display: flex;
            flex-direction: row;
            .cycle-counter {
              position: relative;
              height: 100%;
              width: 10%;
              display: inline-block;
              text-align: center;
              .cycle-counter-container {
                // margin: $margin-dist;
                margin-left: $margin-dist;
                margin-right: $margin-dist;
                background-color: white;
                border-radius: $border-radius-val $border-radius-val;
                text-align: center;
                .title {
                  font-size: 2em;
                }
                .counter {
                  font-size: 2em;
                }
              }
            }
            .strategies {
              position: relative;
              height: 100%;
              width: 90%;

              .strategy {
                background-color: white;
                position: relative;
                min-height: 3em;
                width: 95%;
                display: inline-block;
                .strategy-container {
                  // margin: $margin-dist;
                  position: relative;
                  height: 100%;
                  width: 100%;
                  display: flex;
                  flex-direction: row;
                  overflow: hidden;
                  .name {
                    position: relative;
                    min-height: 100%;
                    min-width: 10%;
                    display: inline-block;
                    text-align: center;
                  }
                  .simulation {
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
                          position: relative;
                          height: 100%;
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
        border-radius: $border-radius-val $border-radius-val;
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

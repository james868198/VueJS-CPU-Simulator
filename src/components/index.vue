<template lang="pug">
.simulator
  .simulator-container
    .simulator-top
      .simulator-top-container
        .simulator-top-left
          .simulator-top-left-container
            b-button.action-btn(variant="danger" size="lg"  v-on:click="") Input
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
                      .center
                        | {{strategy.strategy}}
                  .simulation
                    .simulation-container(ref="screen")
                      .simulation-container-inner(ref="bar")
                        .cycle.bg-danger(v-for="thread in execThreads")
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
      fields: ['id', 'state', 'priority', 'arriveTime', 'totalBurstAndIOTime', 'waitingTime', 'turnAroundTime', 'execTime'],
      cycleMicroSec: 1
    }
  },
  mounted () {
    // console.log(tt)
    // Parse.txtParse(this.$route.path + 'static/test.txt')
    // Parse.txtParse('../../static/tt.json')
    axios.get('../../static/test2.json').then((response) => {
      this.strategies.push(response.data)
      this.initialThreads(0, true)
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
      console.log('test', simulations.length, this.cycleTime)
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
        if (this.cycleTime === 0 || this.cycleTime === simulations.length - 1) {
          this.execThreads.push(-1)
        } else {
          this.execThreads.push(this.execThreads[this.cycleTime - 1])
        }
        this.scrollToRight()
      }
      // to block
      if (simulations[this.cycleTime].moveToBlockList >= 0) {
        // thread move to block
        this.toBlock(strategyId, simulations[this.cycleTime].moveToBlockList)
      }
      // to finish
      if (simulations[this.cycleTime].moveToFinishedList >= 0) {
        // thread move to finish
        this.toFinish(strategyId, simulations[this.cycleTime].moveToFinishedList)
      }
      // update waiting time and turnaroundTime
      this.updateTATAbdWaitingTime(strategyId)

      // cycle counter
      if (simulations.length - 1 <= this.cycleTime) {
        console.log('simulation stop')
        clearInterval(this.simulation)
        this.status = 'stop'
      } else {
        this.cycleTime++
      }
    },
    initialThreads (strategyId, setColor = null) {
      if (!this.strategies[strategyId].threads) {
        return
      }
      let color
      if (setColor) {
        color = randomColor({count: this.strategies[strategyId].threads.length})
      }
      this.strategies[strategyId].threads.forEach((thread, id) => {
        thread['id'] = id
        thread['state'] = 'waiting'
        thread['waitingTime'] = 0
        thread['turnAroundTime'] = 0
        thread['execTime'] = 'N/A'
        if (setColor) {
          thread['color'] = color[id]
        }
      })
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
      this.strategies[strategyId].threads[threadId].execTime = this.cycleTime
    },
    updateTATAbdWaitingTime (strategyId) {
      this.strategies[strategyId].threads.forEach(thread => {
        if (thread.state !== 'waiting' && thread.state !== 'finish') {
          thread.turnAroundTime++
        }
        if (thread.state === 'ready') {
          thread.waitingTime++
        }
      })
    },

    // action
    scrollToRight () {
      const bar = this.$refs.bar[0]
      const screen = this.$refs.screen[0]
      screen.scrollLeft = bar.scrollWidth
    },
    // button events
    // input () {

    // },
    run () {
      console.log('run test')
      if (this.status !== 'run') {
        this.simulation = setInterval(this.simulating, this.cycleMicroSec, 0)
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
        this.initialThreads(0)
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
              min-width: 6em;
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
                min-height: 5em;
                height: 30%;
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
                    min-width: 3em;
                    height: 100%;
                    width: 10%;
                    display: inline-block;
                    text-align: center;
                    font-size: 1.5em;

                    .name-container {
                      position: relative;
                      height: 100%;
                      width: 100%;
                      .center {
                        position: absolute;
                        width: 100%;
                        top:50%;
                        transform: translateY(-50%);
                        text-align: center;
                      }
                    }
                  }
                  .simulation {
                    position: relative;
                    height: 100%;
                    width: 90%;
                    display: inline-block;

                    .simulation-container {
                      position: relative;
                      height: 100%;
                      width: 100%;
                      padding-left:0.2em;
                      padding-right:0.2em;
                      overflow-x: scroll;
                      overflow-y: hidden;
                      &::-webkit-scrollbar { display: none; };
                      .simulation-container-inner {
                        position: absolute;
                        top:50%;
                        transform: translateY(-50%);
                        display: flex;
                        flex-direction: row;
                        .cycle {
                          background-color: yellow;
                          position: relative;
                          height: 100%;
                          width: 2em;
                          min-height: 3.5em;
                          min-width: 1.5em;

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

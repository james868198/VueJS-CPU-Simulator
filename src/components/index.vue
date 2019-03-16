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
            b-button.action-btn(variant="danger" size="lg"  v-on:click="reset")  Reset
        .simulator-top-right
          .simulator-top-right-container
            .cycle-counter
              .cycle-counter-container
                .cycle-counter-top
                  .cycle-counter-top-container
                    label.label Cycle Time
                    .counter {{cycleTime}}
                .cycle-counter-bottom
                  .cycle-counter-bottom-container
                    label.label Cycle Unit(ms):
                    b-form-input.range-input(type="number"  v-model="cycleMicroSec" min="1" max="1000")
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
                        .cpu-thread(v-for="thread in execThreads")
                          .cpu-thread-top.bg-danger
                            .thread-id(v-if="thread.id>=0")
                              | T{{thread.id}}
                            .thread-id(v-else)
                              | N/A
                          .cpu-thread-bottom
                            .cycle-time(v-if="thread.cycleTime%5 == 0")
                              | {{thread.cycleTime}}
                            .cycle-time(v-else)
                              | -------

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
      status: 'ready',
      tableId: 0,
      fields: ['id', 'state', 'priority', 'arriveTime', 'totalBurstAndIOTime', 'waitingTime', 'turnAroundTime', 'execTime'],
      cycleMicroSec: 500
    }
  },
  mounted () {
    // console.log(tt)
    // Parse.txtParse(this.$route.path + 'static/test.txt')
    // Parse.txtParse('../../static/tt.json')
    this.getJson('RR_t5')
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
      // console.log('simulations length:', simulations.length, 'cycles:', this.cycleTime)
      // to ready
      simulations[this.cycleTime].moveToReadyList.forEach(threadId => {
        this.toReady(strategyId, threadId)
      })

      // to CPU
      const thread = {
        cycleTime: this.cycleTime,
        id: -1
      }
      if (simulations[this.cycleTime].moveToCPU >= 0) {
        // thread move to CPU
        thread.id = simulations[this.cycleTime].moveToCPU
        this.toCPU(strategyId, simulations[this.cycleTime].moveToCPU)
      } else {
        // no change
        if (this.cycleTime === 0 || this.cycleTime === simulations.length - 1 || simulations[this.cycleTime].moveToFinishedList >= 0) {
          // no
          thread.id = -1
        } else {
          thread.id = this.execThreads[this.cycleTime - 1].id
        }
      }
      this.execThreads.push(thread)
      // console.log(this.execThreads)
      this.scrollToRight()

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
    getJson (fileName) {
      axios.get('../../static/' + fileName + '.json').then((response) => {
        this.strategies.push(response.data)
        this.initialThreads(0, true)
      })
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
      console.log('run simulation')
      if (this.status === 'ready') {
        this.simulation = setInterval(this.simulating, this.cycleMicroSec, 0)
        this.status = 'run'
      }
    },
    pause () {
      console.log('pause simulation')
      if (this.status === 'run') {
        clearInterval(this.simulation)
        this.status = 'ready'
      }
    },
    reset () {
      console.log('reset test')
      if (this.simulation) {
        clearInterval(this.simulation)
        this.simulation = null
        this.status = 'ready'
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
$margin-dist: 0.7em;
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
                margin-left: $margin-dist;
                margin-right: $margin-dist;
                position: relative;
                height: 100%;
                max-width: 100%;
                text-align: center;
                font-size: 1.5em;
                .cycle-counter-top {
                  position: relative;
                  height: 50%;
                  width: 100%;
                  .cycle-counter-top-container {

                    .label {
                      color: white;
                    }
                    .counter {
                      background-color: white;
                      border-radius: $border-radius-val $border-radius-val;
                    }
                  }
                }
                .cycle-counter-bottom {
                  position: relative;
                  height: 50%;
                  width: 100%;
                  .cycle-counter-bottom-container {
                    border-radius: $border-radius-val $border-radius-val;

                    .label {
                      color: white;
                    }
                    .range-input {

                    }
                  }
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
                height: 40%;
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
                      padding-left:0.1em;
                      padding-right:0.1em;
                      overflow-x: scroll;
                      overflow-y: hidden;
                      &::-webkit-scrollbar { display: none; };
                      .simulation-container-inner {
                        position: absolute;
                        top:50%;
                        transform: translateY(-50%);
                        display: flex;
                        flex-direction: row;
                        .cpu-thread {
                          position: relative;
                          height: 100%;
                          width: 2.5em;
                          min-height: 5em;
                          min-width: 1.5em;

                          overflow: hidden;
                          display: inline-block;
                          margin-right:0.5em;
                          .cpu-thread-top{
                            position: relative;
                            height: 80%;
                            min-height: 4em;
                            width: 100%;

                            .thread-id {
                              position: relative;
                              height: 100%;
                              width: 100%;
                              font-size: 1.2em;
                              text-align: center;
                            }
                          }
                          .cpu-thread-bottom{
                            background-color: white;
                            position: relative;
                            height: 20%;
                            min-height: 1em;
                            width: 100%;
                            .cycle-time {
                              position: relative;
                              height: 100%;
                              width: 100%;
                              font-size: 0.5em;
                              text-align: center;
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

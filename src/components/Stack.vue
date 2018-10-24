<template>
  <div class="stack-container">

    <h2 class="stack-h2">stack</h2>

    <div id="shadow-stack-for-stack-pointer">
      <svg id="stack-pointer"
           :style="`transition: all ${0.6 * speed}s; transform: translateY(-${stackQueue.length * 42 - 10}px);`"
           xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24"><path d="M24 12l-12-9v5h-12v8h12v5l12-9z"/></svg>
    </div>

    <div id="stack">
      <transition-group :name="beingInserted ? '': 'func-complete'">
        <div v-for="(func, indx) in stackQueue.slice().reverse()"
             @click="stackPop(func.name)"
             :key="func.name"
             class="stack-func-box"
             :style="`transition: all ${1 * speed}s`"
             :class="{ 'func-complete-item': !beingInserted, 'stack-head': indx === 0, 'in-execution': indx === 0 && looping }">
          {{ func.name }}
        </div>
      </transition-group>
    </div>


    <h2 class="paint">Paint</h2>

    <div id="timers">
      <h2>timers & async</h2>
      <transition-group name="timer-complete">
        <div v-for="timer in timers"
             :key="timer.name"
             :ref="timer.name"
             class="timer-box timer-complete-item"
             :class="{ 'queue-bg': timer.type === 'timer', 'micro-bg': timer.type === 'async' }">
          {{ timer.name }}
          {{ timer.value }}
        </div>
      </transition-group>
    </div>

    <h2 class="queue-h2">queue</h2>
    <div id="queue">
      <transition-group name="task-complete">
        <div v-for="task in macroQueue.slice().reverse()"
             @click="taskPop(task.name)"
             :key="task.name"
             :ref="task.name"
             style="transition: all 0.1s !important;"
             class="macro-task-box task-complete-item">
          {{ task.name }}
        </div>
      </transition-group>
    </div>


    <div id="eventloop">
      <svg class="spinner" height="200" width="250"
           :style="rotationStyleString"
           @click="rotate">
        <circle cx="100" cy="100" r="90" stroke="black" stroke-width="3" fill="tomato" />
        <path d="M220 100 L175 80 L175 120 Z" />
      </svg>
    </div>


    <h2 class="micro-h2">micro tasks</h2>
    <div id="micro">
      <transition-group name="micro-complete">
        <div v-for="task in microQueue.slice().reverse()"
             :key="task.name"
             :ref="task.name"
             @click="microPop(task.name)"
             class="macro-task-box micro-complete-item">
          {{ task.name }}
        </div>
      </transition-group>
    </div>

    <div id="controls">
      <button @click="callFunction">Call function</button>
      <button @click="scheduleUIEvent">Schedule UI event</button>
      <button @click="scheduleTimer('timer')">Schedule timer</button>
      <button @click="scheduleTimer('async')">Schedule network</button>
      <button @click="looping = !looping">{{ looping ? 'Pause loop' : 'Loop' }}</button>
      <button @click="speed += 0.1">+</button>
      <button @click="speed -= 0.1">-</button>
    </div>


    <div style="position:absolute;top:10px;right:10px;">
      System speed: {{ speed }}
    </div>

  </div>
</template>

<script lang="ts">
import Vue from 'vue';

export default Vue.extend({
  name: 'Stack',
  props: {
    msg: String,
  },
  data() {
    return {
      speed: 1.0,
      stackQueue: [{ name: 'func1' }, { name: 'func2' }],
      macroQueue: [{ name: 'T1' }, { name: 'T2' }],
      timers: [],
      microQueue: [],
      rotation: 0,
      rotationIndx: 0,
      possibleRotations: [65, 65, 230],
      rotating: false,
      looping: false,
      autoLoop: false,
      beingInserted: false,
    }
  },

  mounted() {
  },
  watch: {
    looping(val) {
      if (val) {
        this.loop();
      }
    },
    stackQueue(stack) {
      if (this.looping || !this.autoLoop) return;
      if (stack.length === 0) {
        this.rotate();
      }
    },
    macroQueue(tasks) {
      if (this.looping || !this.autoLoop) return;
      if (tasks.length !== 0) {
        this.rotate();
      }
    }
  },
  computed: {
    stackQueueOrdered() { return this.stackQueue.reverse() },
    rotationStyleString() {
      return `transition-duration: ${this.speed}s; transform: rotate(${this.rotation}deg)`;
    },
    currentState() { return ['stack', 'micro', 'macro'][this.rotationIndx]; }
  },
  methods: {

    loop() {
      if (!this.looping) return;
      const stateNames = ['stack', 'micro', 'macro'];
      const initialState = stateNames[this.rotationIndx];
      //console.log(`initial: ${initialState}`);

      const stackEmpty = this.stackQueue.length === 0;
      const microEmpty = this.microQueue.length === 0;
      const macroEmpty = this.macroQueue.length === 0;
      if (stackEmpty && microEmpty && macroEmpty) {
        console.log('NO-OP');
        setTimeout(() => this.loop(), 500 * this.speed);
        return;
      }



      const state = stateNames[this.rotationIndx];
      console.log(`state: ${state}`);

      if (state === 'micro') {

        if (microEmpty) {
          if (!macroEmpty) {
            setTimeout(() => this.rotate(), 1000 * this.speed);
          }
          setTimeout(() => this.loop(), 2100 * this.speed);
          return;
        }

        setTimeout(() => this.microPop(), 1000);
        setTimeout(() => this.loop(), 1500 * this.speed);

      } else if (state === 'stack') {
        if (this.stackQueue.length !== 0) {
          setTimeout(() => this.stackPop(), 1000 * this.speed);
          if (this.stackQueue.length === 1) {
            setTimeout(() => this.loop(), 1200 * this.speed);
          } else {
            setTimeout(() => this.loop(), 2000 * this.speed);
          }
        } else {
          console.log('stack empty')
          setTimeout(() => this.loop(), 500 * this.speed);
        }
      } else {
        if (this.macroQueue.length !== 0) {
          console.log('schedule taskPop');
          setTimeout(() => this.taskPop('_', true), 1000 * this.speed);
        }
        setTimeout(() => this.loop(), 2100 * this.speed);
      }

    },

    rotate(evt) {
      console.log('rotate ', this.rotating)
      if (this.rotating) return;

      if (evt && evt !== -1) {
        if (this.currentState === 'stack' && this.stackQueue.length !== 0) {
          console.log('CANNOT ROTATE:::: stack not empty');
          return;
        }
        if (this.currentState === 'micro' && this.microQueue.length !== 0) {
          console.log('CANNOT ROTATE:::: micro not empty');
          return;
        }
        if (this.currentState === 'macro' && this.stackQueue.length === 0) {
          console.log('CANNOT ROTATE:::: pickup macrotask!');
          return;
        }
      }

      this.rotating = true;
      console.log('rotation STARTED ', this.rotating);
      if (evt === -1) {
        this.rotation -= this.possibleRotations[this.rotationIndx];
        this.rotationIndx -= 1;
      } else {
        if (this.rotationIndx < this.possibleRotations.length - 1) {
          this.rotationIndx += 1;
        } else {
          this.rotationIndx = 0;
        }
        this.rotation += this.possibleRotations[this.rotationIndx];
      }

      setTimeout(() => this.rotating = false, 1000 * this.speed);

    },

    scheduleTimer(type) {
      const time = Math.floor(Math.random() * 10) * 1000;
      const timer = { name: `timer ${Math.floor(Math.random() * 10)}`, value: time, type: type };
      this.timers.push(timer)

      let lastTime: any = new Date();
      const countDown = () => {
        const now: any = new Date();
        timer.value -= now - lastTime;

        if (timer.value > 50) {
          lastTime = now;
          setTimeout(countDown, 50);
        } else {
          timer.value = 0;
          const ref = this.$refs[timer.name][0];
          const refPos = ref.getBoundingClientRect();

          if (type === 'timer') {
            const queuePos = document.getElementsByClassName('queue-h2')[0].getBoundingClientRect();
            ref.style.transform = `translate(${queuePos.left + queuePos.width / 2 - refPos.left}px, ${queuePos.top + 20 - refPos.top}px)`;
            setTimeout(() => {
              ref.style.display = 'none';
              this.timers.splice(this.timers.findIndex(t => t === timer), 1)
              this.scheduleUIEvent();
            }, 800);
          } else {
            const microPos = document.getElementsByClassName('micro-h2')[0].getBoundingClientRect();
            ref.style.transform = `translate(${microPos.left + microPos.width / 2 - refPos.left}px, ${microPos.top + 20 - refPos.top}px)`;
            setTimeout(() => {
              ref.style.display = 'none';
              this.timers.splice(this.timers.findIndex(t => t === timer), 1)
              this.scheduleMicroTask();
            }, 800);
          }

        }
      }

      setTimeout(countDown, 50);
    },

    callFunction() {
      // cannot be called if not executing stack
      // cannot be called outside of stack
      if (this.rotationIndx !== 0 || this.stackQueue.length === 0) {
        return;
      }
      if (this.stackQueue.length > 0) {
        const currExecFn = this.stackQueue.slice(-1)[0].name;
        console.log(currExecFn);
        const newNumber = parseInt(currExecFn.match(/[0-9]+/)[0]) + 1;
        this.stackQueue.push({ name: `func${newNumber}` });
      } else {
        this.stackQueue.push({ name: 'func1' });
      }
    },

    scheduleUIEvent() {
      if (this.macroQueue.length === 0) {
        this.macroQueue.push({ name: 'T1' });
      } else {
        const lastTask = this.macroQueue.slice(-1)[0].name;
        const newNumber = parseInt(lastTask.match(/[0-9]+/)[0]) + 1;
        this.macroQueue.push({ name: `T${newNumber}` });
      }
    },

    scheduleMicroTask() {
      if (this.microQueue.length === 0) {
        this.microQueue.push({ name: 'T1' });
      } else {
        const lastTask = this.microQueue.slice(-1)[0].name;
        const newNumber = parseInt(lastTask.match(/[0-9]+/)[0]) + 1;
        this.microQueue.push({ name: `T${newNumber}` });
      }
    },

    stackPop(name) {

      let popped = false;
      if (name && this.stackQueue.slice(-1)[0].name === name) {
        this.stackQueue.pop();
        popped = true;
      }


      if (!name) {
        this.stackQueue.pop();
        popped = true;
      }

      if (popped && this.stackQueue.length === 0) {
        setTimeout(() => this.rotate(), 1000 * this.speed);
      }

    },

    taskPop(name, force) {

      if (this.stackQueue.length !== 0) {
        throw new Error('ILLEGAL');
        return;
      }

      if (force || this.macroQueue[0].name === name) {
        const poppedTask = this.macroQueue[0];
        this.macroQueue = this.macroQueue.slice(1);
        setTimeout(() => this.stackQueue.push({ name: poppedTask.name }), 1000 * this.speed);
        if (this.stackQueue.length === 0) {
          this.beingInserted = true;
          setTimeout(() => this.beingInserted = false, 1000 * this.speed);
        }
        setTimeout(() => this.rotate(), 1100 * this.speed);
      }
    },

    microPop(name) {

      if (this.currentState !== 'micro') {
        throw new Error('CANNOT POP NOW')
      }

      if (name) {
        if (name !== this.microQueue[0].name) {
          throw new Error('CANNOT POP THAT ONE STUPID')
        }
      }

      const task = this.microQueue[0];
      this.microQueue = this.microQueue.slice(1);
      this.stackQueue.push({ name: task.name });
      console.log('rotating from microPop')
      this.rotate(-1);
    }

  },

});
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>


.paint {
  grid-row-start: 2;
  grid-row-end: 3;
  grid-column-start: 1;
  grid-column-end: 1;
  justify-self: start;
  margin-left: 20px;
}

button {
  padding: 10px;
  font-size: 20px;
}

.stack-container {
  position: absolute;
  max-width: 1200px;
  width: 90%;
  height: 90%;
  display: grid;
  grid-template-columns: 25% 25% 25% 25%;
  grid-template-rows: 25% auto 10% 10%;
  grid-column-gap: 25px;
  grid-row-gap: 25px;
  border: 1px solid black;
}

#stack {
  grid-row-start: 2;
  grid-row-end: 3;
  grid-column-start: 2;
  grid-column-end: 4;
  justify-self: center;
  width: 200px;
  background: #fbb637a1;
  display: grid;
  align-content: end;
  margin-left: 50px;
}

#shadow-stack-for-stack-pointer {
  grid-row-start: 2;
  grid-row-end: 3;
  grid-column-start: 2;
  grid-column-end: 4;
  justify-self: center;
  width: 200px;
  display: grid;
  align-content: end;
}

#stack-pointer {
  margin-left: -10px;
}

.stack-h2 {
  grid-row-start: 2;
  grid-row-end: 3;
  grid-column-start: 2;
  grid-column-end: 4;
  margin-top: -35px;
  margin-left: 50px;
}

.stack-func-box {
  border: 2px solid black;
  margin: 10px;
  padding: 5px;
  transform: all 1s;
}

.func-complete-item {
  transition: all 1s;
}

.func-complete-enter {
  opacity: 1;
  transform: translateY(100px);
  border: 2px solid black !important;
  background: none !important;
}

.func-complete-leave-to {
  opacity: 0;
  transform: translate(0px, 50px);
}

.func-complete-leave-active {
  position: absolute;
}

#timers {
  grid-row-start: 2;
  grid-row-end: 3;
  grid-column-start: 4;
  grid-column-end: 4;
  width: 150px;
  background: pink;
}

#timers h2 {
  margin-top: -65px;
}

#queue {
  grid-row-start: 1;
  grid-row-end: 2;
  grid-column-start: 1;
  grid-column-end: 3;
  background: lightgreen;
  height: 80px;
  margin: 50px 100px 0 50px;
  display: grid;
  grid-auto-flow: column;
  justify-content: end;
}

.queue-bg {
  background: lightgreen;
}

.timer-complete-item {
  transition: all 1s;
  display: inline-block;
}

.timer-complete-leave-to {
  transition: all 0s;
  opacity: 0;
}

.task-complete-item {
  transition: all 1s;
  display: inline-block;
  margin-right: 100px;
}

.task-complete-enter {
  opacity: 1;
  transform: translateY(-100px);
}

.task-complete-leave-to {
  transition: all 1s;
  opacity: 1;
  transform: translate(100px, 550px);
}

.task-complete-leave-active, .timer-complete-leave-active {
  position: absolute;
}

.queue-h2 {
  margin: 10px 50px 0 0;
  grid-row-start: 1;
  grid-row-end: 2;
  grid-column-start: 1;
  grid-column-end: 3;
}

.macro-task-box {
  border: solid 2px black;
  width: 50px;
  height: 50px;
  margin: 5px;
}

.timer-box {
  border: solid 2px red;
  width: 50px;
  height: 50px;
  padding: 10px;
  border-radius: 50px;
}
#eventloop {
  grid-row-start: 2;
  grid-row-end: 3;
  grid-column-start: 1;
  grid-column-end: 2;
}

.spinner {
  margin: 50px;
  transform-origin: 100px 100px;
}

.micro-h2 {
  margin: 10px 50px 0 0;
  grid-row-start: 3;
  grid-row-end: 4;
  grid-column-start: 1;
  grid-column-end: 3;
  margin-top: -35px;
}

#micro {
  grid-row-start: 3;
  grid-row-end: 4;
  grid-column-start: 1;
  grid-column-end: 3;
  justify-content: middle;
  background: #80deea;
  margin-left: 25px;
  display: grid;
  grid-auto-flow: column;
  justify-content: end;
}

.micro-bg {
  background: #80deea;
}

.micro-complete-item {
  transition: all 1s;
  display: inline-block;
}

.micro-complete-enter {
  opacity: 1;
  transform: translateY(-100px);
}

.micro-complete-leave-to {
  transition: all .5s;
  opacity: 1;
  transform: translate(100px, 550px);
}

#controls {
  grid-row-start: 4;
  grid-row-end: 4;
  grid-column-start: 1;
  grid-column-end: 4;
  justify-content: middle;
  margin-top: 25px;
}



.in-execution {
  border: 2px solid red;
  background: greenyellow;
}

.stack-head {
  cursor: url('data:image/svg+xml;base64,PHN2ZyB2aWV3Qm94PSIwIDAgOTAwIDgzMCIgd2lkdGg9IjUwcHgiIGhlaWdodD0iNTBweCIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj4KPGRlZnM+CjxmaWx0ZXIgaWQ9ImZpbHRlcjc4MjItMSI+CjxmZUdhdXNzaWFuQmx1ciBzdGREZXZpYXRpb249IjQuMDI0ODQiLz4KPC9maWx0ZXI+CjxyYWRpYWxHcmFkaWVudCBjeD0iNjgwLjEzOTIyIiBjeT0iODE1LjIzNTQxIiBncmFkaWVudFRyYW5zZm9ybT0ibWF0cml4KDEuMjEzMzgsLTMuNDU4OTUyNGUtNywzLjgzMjMxNjRlLTcsMS4zNDQzNiwtMTQ1LjEyOTk4LC0yODAuNzMxMzMpIiBncmFkaWVudFVuaXRzPSJ1c2VyU3BhY2VPblVzZSIgaWQ9InJhZGlhbEdyYWRpZW50Mzk0NSIgcj0iMTk4LjgyODEyIj4KPHN0b3Agb2Zmc2V0PSIwIiBzdG9wLWNvbG9yPSIjZmMwIi8+CjxzdG9wIG9mZnNldD0iMSIgc3RvcC1jb2xvcj0iI2U0MDAzOSIgc3RvcC1vcGFjaXR5PSIuOTY0NzEiLz4KPC9yYWRpYWxHcmFkaWVudD4KPHJhZGlhbEdyYWRpZW50IGN4PSI2ODAuMTM5MjIiIGN5PSI4MTUuMjM1NDEiIGdyYWRpZW50VHJhbnNmb3JtPSJtYXRyaXgoMS40OTQ3NCwuMDE0MTgsLS4wMDk3MiwxLjAyNDI0LC0zMjguNzE2NjgsLTI5LjU0MjkpIiBncmFkaWVudFVuaXRzPSJ1c2VyU3BhY2VPblVzZSIgaWQ9InJhZGlhbEdyYWRpZW50Mzk0NyIgcj0iMTM2LjY5NzI0Ij4KPHN0b3Agb2Zmc2V0PSIwIiBzdG9wLWNvbG9yPSIjZmZkNjAwIi8+CjxzdG9wIG9mZnNldD0iMSIgc3RvcC1jb2xvcj0iI2ZjMCIgc3RvcC1vcGFjaXR5PSIwIi8+CjwvcmFkaWFsR3JhZGllbnQ+CjwvZGVmcz4KPHRpdGxlPlBvdzwvdGl0bGU+CjxnIHRyYW5zZm9ybT0ibWF0cml4KDEuNDI1OCwuNzU5MTMsLS44MDA5MywxLjM5NzE1LDc3LjE4NDAxLC0xMjM4Ljg5MDY1KSI+CjxwYXRoIGQ9Im03NTMuNzgwNDIsNjExLjU3OTE4YzAsMC0zOC42MzU0LDgxLjM3MS03Mi4xMjUsOTAuNTMxMi0xNS40NDA1LDQuMjIzNC00MS41MDg1LTYuOTM3Mi02MS40Njg4LTE3LjU2MjQtMjAuNjcwMy0xOC4xNjI0LTM2Ljk2ODctMzUuNTYyNi0zNi45Njg3LTM1LjU2MjZzNDQuMTc2OCw5MS4xMDg1IDMyLjg0MzcsMTE4LjMxMjZjLS4wNDc2LjExNDMtLjEwNTcuMjI5Ny0uMTU2Mi4zNDM3LTMwLjQ5NjEsMTQuMTczMi05OS43MTg4LDEzLjY1NjMtOTkuNzE4OCwxMy42NTYzczIxLjU0MDMsNy4yNjY1NSA0My4zNDM4LDE3LjQwNjI1Yy0zNC44NDY5LDEwLjE5NzEtNzIuNTYyNSwxNS40Njg4LTcyLjU2MjUsMTUuNDY4OHM5Ny4zNjQ4LDE0Ljg2ODggMTEwLjAzMTIsMzcuNDM3NGMxLjMzMzMsMi4zNzU2IDIuNDYwNCw1LjAyMTMgMy4zNzUsNy44NzUtMTMuOTAyMiwzOC4wNTQ1LTU0LjUsOTAuMDkzOC01NC41LDkwLjA5MzhzMjcuMzI5Mi0xOC45ODYxIDU2LjAzMTMtMzMuOTA2MmMtNC4xODM0LDMyLjExMjQtMTMuMzc1LDYyLjAzMTItMTMuMzc1LDYyLjAzMTJzNTUuMzE5OS01Ny4xNTM0IDg2LjI1LTU2LjUzMTJjMi40ODQ0LjA0OTkgNS4xMDIzLjQ3MDMgNy44MTI1LDEuMTU2MiAyMi43NzgzLDM1LjE5MTUgNDEuNSw5Ni41NjI0NyA0MS41LDk2LjU2MjQ3cy0uMjA2MS0zNi43ODU3NyA0LjI4MTItNjkuMDYyNDdjMjEuNDg2MSwxNy44ODg1IDM5LjAzMTMsMzcuNTMxMiAzOS4wMzEzLDM3LjUzMTJzLTIzLjQ5NjgtNTYuMzA4OS0xNy45MDYzLTg2LjY4NzRjMjIuNjE4MS0xMi40NTkgNjQuNzY5OS0xMi4xODAxIDkyLjY4NzUtMTAuMzEyNiAyMy42MTk3LDcuODY0IDQyLjQzNzUsMTYuNzE4OCA0Mi40Mzc1LDE2LjcxODhzLTguMzk0OC01Ljc3MzUtMTkuNjI1LTE0LjY1NjJjMi45Nzc0LjM1NDEgNC43NS41OTM3IDQuNzUuNTkzN3MtMjYuMjg4Mi0xOC40NTM5LTQ2LjUzMTItMzkuNzgxM2MtOS4wMDYxLTEwLjc5NzEtMTUuNjc1NS0yMS43OTQzLTE2Ljc1LTMxLjUtMy4xNDcyLTI4LjQyODYgNDcuNDM3NS04Mi43MTg3NSA0Ny40Mzc1LTgyLjcxODc1cy0xMy41ODk1LDguNzk5My0zMS4xODc1LDE2LjM0MzdjMTMuNzI1Ny0yMi4yODkgMjguNjI1LTQwLjUzMTIgMjguNjI1LTQwLjUzMTJzLTczLjQwOTYsNDUuMjc2LTk3LjU2MjUsMjYuODc1Yy0xLjMwOTQtLjk5NzYtMi40OTgyLTIuMTU5NS0zLjU5MzgtMy40Mzc1LTUuNzAyMi0zNC40MTM2IDMuOTA2My05Mi4zNDM3IDMuOTA2My05Mi4zNDM3cy00Ljc0MjMsOS4wNDgxLTExLjg0MzgsMjAuOTM3NGMzLjkxOS0yOS4yMDMgMTEuNTMxMy01NS4yODEyIDExLjUzMTMtNTUuMjgxMnoiIGZpbGwtcnVsZT0iZXZlbm9kZCIgZmlsdGVyPSJ1cmwoI2ZpbHRlcjc4MjItMSkiIG9wYWNpdHk9Ii42ODI2OSIgdHJhbnNmb3JtPSJtYXRyaXgoMS4xMjE5LDAsMCwxLjEyMTksLTgzLjU5NzEsLTk5LjM3NTQ5KSIvPgo8cGF0aCBkPSJtNzQ4LjEyMzU4LDYxMS41NzkxOGMtMmUtMDA1LDAtMzguNjM1NDIsODEuMzcxLTcyLjEyNSw5MC41MzEyLTE1LjQ0MDQ2LDQuMjIzNC00MS41MDg1NC02LjkzNzItNjEuNDY4NzUtMTcuNTYyNC0yMC42NzAzNy0xOC4xNjI0LTM2Ljk2ODc1LTM1LjU2MjYtMzYuOTY4NzUtMzUuNTYyNnM0NC4xNzY4NCw5MS4xMDg1IDMyLjg0Mzc1LDExOC4zMTI2Yy0uMDQ3Ni4xMTQzLS4xMDU3Mi4yMjk3LS4xNTYyNS4zNDM3LTMwLjQ5NjA2LDE0LjE3MzItOTkuNzE4NzUsMTMuNjU2My05OS43MTg3NSwxMy42NTYzczIxLjU0MDI4LDcuMjY2NSA0My4zNDM3NSwxNy40MDYyYy0zNC44NDY4NywxMC4xOTcxLTcyLjU2MjUsMTUuNDY4OC03Mi41NjI1LDE1LjQ2ODhzOTcuMzY0OCwxNC44Njg4IDExMC4wMzEyNSwzNy40Mzc0YzEuMzMzMjQsMi4zNzU2IDIuNDYwMzYsNS4wMjEzIDMuMzc1LDcuODc1LTEzLjkwMjIzLDM4LjA1NDUtNTQuNSw5MC4wOTM4LTU0LjUsOTAuMDkzOHMyNy4zMjkxMi0xOC45ODYxIDU2LjAzMTI1LTMzLjkwNjJjLTQuMTgzMzUsMzIuMTEyNC0xMy4zNzUsNjIuMDMxMi0xMy4zNzUsNjIuMDMxMnM1NS4zMTk4OC01Ny4xNTM0IDg2LjI1LTU2LjUzMTJjMi40ODQzOS4wNDk5IDUuMTAyMzQuNDcwMyA3LjgxMjUsMS4xNTYyIDIyLjc3ODMxLDM1LjE5MTUgNDEuNSw5Ni41NjI1MiA0MS41LDk2LjU2MjUycy0uMjA2MDgtMzYuNzg1ODIgNC4yODEyNS02OS4wNjI1MmMyMS40ODYwOCwxNy44ODg1IDM5LjAzMTI1LDM3LjUzMTIgMzkuMDMxMjUsMzcuNTMxMnMtMjMuNDk2NzktNTYuMzA4OS0xNy45MDYyNS04Ni42ODc0YzIyLjYxODExLTEyLjQ1OSA2NC43Njk4Mi0xMi4xODAxIDkyLjY4NzUtMTAuMzEyNiAyMy42MTk3LDcuODY0IDQyLjQzNzUsMTYuNzE4OCA0Mi40Mzc1LDE2LjcxODhzLTguMzk0ODEtNS43NzM1LTE5LjYyNS0xNC42NTYyYzIuOTc3MzMuMzU0MSA0Ljc1LjU5MzcgNC43NS41OTM3cy0yNi4yODgyMS0xOC40NTM5LTQ2LjUzMTI1LTM5Ljc4MTNjLTkuMDA2MDgtMTAuNzk3MS0xNS42NzU1MS0yMS43OTQzLTE2Ljc1LTMxLjUtMy4xNDcyMy0yOC40Mjg2IDQ3LjQzNzUtODIuNzE4NyA0Ny40Mzc1LTgyLjcxODdzLTEzLjU4OTU0LDguNzk5My0zMS4xODc1LDE2LjM0MzdjMTMuNzI1NjktMjIuMjg5IDI4LjYyNS00MC41MzEyIDI4LjYyNS00MC41MzEycy03My40MDk2MSw0NS4yNzYtOTcuNTYyNSwyNi44NzVjLTEuMzA5NDQtLjk5NzYtMi40OTgyMi0yLjE1OTUtMy41OTM3NS0zLjQzNzUtNS43MDIyNC0zNC40MTM2IDMuOTA2MjUtOTIuMzQzNyAzLjkwNjI1LTkyLjM0MzdzLTQuNzQyMjgsOS4wNDgxLTExLjg0Mzc1LDIwLjkzNzRjMy45MTg5NC0yOS4yMDMgMTEuNTMxMjUtNTUuMjgxMiAxMS41MzEyNS01NS4yODEyeiIgZmlsbD0idXJsKCNyYWRpYWxHcmFkaWVudDM5NDUpIiBmaWxsLXJ1bGU9ImV2ZW5vZGQiLz4KPHBhdGggZD0ibTczNC4wOTc2OCw2NzcuNzU4MTZjLTFlLTAwNSwwLTI5LjQ5MzU3LDU0LjQ1NTQyLTUyLjgxOTksNTkuNTIyNTYtMTAuNzU0NjYsMi4zMzYyNi0yOC4yNDQ0My02LjI3NzAzLTQxLjU2MDQxLTE0LjI5OTktMTMuNTI4NTYtMTMuMjIzNDItMjQuMDgzMzQtMjUuNzY0MDYtMjQuMDgzMzQtMjUuNzY0MDZzMjcuMDA1Niw2NC4xNjE4NSAxOC4yMzI0NSw4Mi40MjUyNmMtLjAzNjkuMDc2Ny0uMDgxLjE1Mzg0LS4xMTk4LjIzMDI2LTIxLjQ1NDAyLDguNjE3OTktNjguOTU5NzEsNS43Mzc3My02OC45NTk3MSw1LjczNzczczE0LjUyMzI5LDUuNzc0NjMgMjkuMTIyNDYsMTMuNTMxNDRjLTI0LjI5Niw1LjcyOTQ4LTUwLjM4MTg0LDcuOTcyOC01MC4zODE4NCw3Ljk3MjhzNjYuMzAyOTgsMTMuNzYwMTggNzQuMTc1NzQsMjkuNzE2NjVjLjgyODY1LDEuNjc5NjEgMS41MDU5NSwzLjUzNzEyIDIuMDI5NzksNS41Mjk2OS0xMC45MzI4NCwyNS42MTg5Ny00MC43MDM1Nyw1OS44NjUyNy00MC43MDM1Nyw1OS44NjUyN3MxOS40NTUzNS0xMi4wMzc4MiAzOS43MDUtMjEuMjM0MDVjLTQuMDQzNTksMjEuODk0MDQtMTEuNDQ1NTcsNDIuMDk5MzQtMTEuNDQ1NTcsNDIuMDk5MzRzNDAuMDY0NzEtMzcuMjIwMjcgNjEuMjc2OTYtMzUuNjY0NzFjMS43MDM4My4xMjQ5MSAzLjQ4NTg0LjUwOTAzIDUuMzIxNDYsMS4wNzg4MSAxNC4zNTQ1MSwyNC45OTE1OSAyNC45Njg4Nyw2Ny44MDg2MSAyNC45Njg4Nyw2Ny44MDg2MXMxLjIwMDUzLTI1LjI2MjY4IDUuNDU4ODEtNDcuMjU4NGMxNC4wOTg1OCwxMy4wNjUxMyAyNS40Mjc1MywyNy4xOTA4MyAyNS40Mjc1MywyNy4xOTA4M3MtMTQuMDc3MzctMzkuNTE1ODYtOS4xMzA5My02MC4xNjgxNGMxNS45ODI5LTcuNzI4NTIgNDQuOTExODEtNS45OTkyNiA2NC4wMTA0NS0zLjY5ODY2IDE1LjkyOTExLDYuMjYwNzEgMjguNTI1MzYsMTMuMDI2NDMgMjguNTI1MzYsMTMuMDI2NDNzLTUuNTUyOC00LjI3MDAxLTEyLjkzODc3LTEwLjc3ODExYzIuMDMxMTUuMzUxNzQgMy4yMzk0Mi41ODA5IDMuMjM5NDIuNTgwOXMtMTcuMzc0ODMtMTMuNjI4NS0zMC40OTQ1NC0yOS4wMDkyNWMtNS43ODkxOC03Ljc0MTI2LTkuOTY2ODUtMTUuNTM0NjctMTAuMzUwNDUtMjIuMjM3MjctMS4xMjM1OS0xOS42MzIzNSAzNS41ODU3Ni01NS4wNTk1OCAzNS41ODU3Ni01NS4wNTk1OHMtOS42NTA4NSw1LjU0NTM2LTIyLjAwNzkxLDEwLjA4MjkyYzEwLjIzNjQ3LTE0LjgwMTY4IDIxLjEzMTA1LTI2Ljc4MjIzIDIxLjEzMTA1LTI2Ljc4MjIzcy01Mi4wNTA4MywyOC40MDU5Mi02Ny45NjE2LDE0Ljg5MTY1Yy0uODYyNTktLjczMjY3LTEuNjM2MzYtMS41NzM3NC0yLjM0MTg2LTIuNDkxMTItMi42NTkzOC0yMy44MzQ1NSA2LjA1MDctNjMuMjU1NjkgNi4wNTA3LTYzLjI1NTY5cy0zLjU4NTg4LDYuMDM4OTQtOC44OTUxMSwxMy45NDI0MWMzLjc1NTkxLTE5LjkwNjI0IDkuOTMzNS0zNy41MzIzOSA5LjkzMzUtMzcuNTMyMzl6IiBmaWxsPSJ1cmwoI3JhZGlhbEdyYWRpZW50Mzk0NykiIGZpbGwtcnVsZT0iZXZlbm9kZCIgb3BhY2l0eT0iLjYxMDU4Ii8+CjwvZz4KPGcgdHJhbnNmb3JtPSJtYXRyaXgoLjAzNTk5LC0xLjU3MDQ0LC0xLjY1NDg0LC0uMDM0MTUsMTc4Ni41MzQ3MSwxNTE2LjYwMTQ4KSI+CjxwYXRoIGQ9Im03NTMuNzgwNDIsNjExLjU3OTE4YzAsMC0zOC42MzU0LDgxLjM3MS03Mi4xMjUsOTAuNTMxMi0xNS40NDA1LDQuMjIzNC00MS41MDg1LTYuOTM3Mi02MS40Njg4LTE3LjU2MjQtMjAuNjcwMy0xOC4xNjI0LTM2Ljk2ODctMzUuNTYyNi0zNi45Njg3LTM1LjU2MjZzNDQuMTc2OCw5MS4xMDg1IDMyLjg0MzcsMTE4LjMxMjZjLS4wNDc2LjExNDMtLjEwNTcuMjI5Ny0uMTU2Mi4zNDM3LTMwLjQ5NjEsMTQuMTczMi05OS43MTg4LDEzLjY1NjMtOTkuNzE4OCwxMy42NTYzczIxLjU0MDMsNy4yNjY1NSA0My4zNDM4LDE3LjQwNjI1Yy0zNC44NDY5LDEwLjE5NzEtNzIuNTYyNSwxNS40Njg4LTcyLjU2MjUsMTUuNDY4OHM5Ny4zNjQ4LDE0Ljg2ODggMTEwLjAzMTIsMzcuNDM3NGMxLjMzMzMsMi4zNzU2IDIuNDYwNCw1LjAyMTMgMy4zNzUsNy44NzUtMTMuOTAyMiwzOC4wNTQ1LTU0LjUsOTAuMDkzOC01NC41LDkwLjA5MzhzMjcuMzI5Mi0xOC45ODYxIDU2LjAzMTMtMzMuOTA2MmMtNC4xODM0LDMyLjExMjQtMTMuMzc1LDYyLjAzMTItMTMuMzc1LDYyLjAzMTJzNTUuMzE5OS01Ny4xNTM0IDg2LjI1LTU2LjUzMTJjMi40ODQ0LjA0OTkgNS4xMDIzLjQ3MDMgNy44MTI1LDEuMTU2MiAyMi43NzgzLDM1LjE5MTUgNDEuNSw5Ni41NjI0NyA0MS41LDk2LjU2MjQ3cy0uMjA2MS0zNi43ODU3NyA0LjI4MTItNjkuMDYyNDdjMjEuNDg2MSwxNy44ODg1IDM5LjAzMTMsMzcuNTMxMiAzOS4wMzEzLDM3LjUzMTJzLTIzLjQ5NjgtNTYuMzA4OS0xNy45MDYzLTg2LjY4NzRjMjIuNjE4MS0xMi40NTkgNjQuNzY5OS0xMi4xODAxIDkyLjY4NzUtMTAuMzEyNiAyMy42MTk3LDcuODY0IDQyLjQzNzUsMTYuNzE4OCA0Mi40Mzc1LDE2LjcxODhzLTguMzk0OC01Ljc3MzUtMTkuNjI1LTE0LjY1NjJjMi45Nzc0LjM1NDEgNC43NS41OTM3IDQuNzUuNTkzN3MtMjYuMjg4Mi0xOC40NTM5LTQ2LjUzMTItMzkuNzgxM2MtOS4wMDYxLTEwLjc5NzEtMTUuNjc1NS0yMS43OTQzLTE2Ljc1LTMxLjUtMy4xNDcyLTI4LjQyODYgNDcuNDM3NS04Mi43MTg3NSA0Ny40Mzc1LTgyLjcxODc1cy0xMy41ODk1LDguNzk5My0zMS4xODc1LDE2LjM0MzdjMTMuNzI1Ny0yMi4yODkgMjguNjI1LTQwLjUzMTIgMjguNjI1LTQwLjUzMTJzLTczLjQwOTYsNDUuMjc2LTk3LjU2MjUsMjYuODc1Yy0xLjMwOTQtLjk5NzYtMi40OTgyLTIuMTU5NS0zLjU5MzgtMy40Mzc1LTUuNzAyMi0zNC40MTM2IDMuOTA2My05Mi4zNDM3IDMuOTA2My05Mi4zNDM3cy00Ljc0MjMsOS4wNDgxLTExLjg0MzgsMjAuOTM3NGMzLjkxOS0yOS4yMDMgMTEuNTMxMy01NS4yODEyIDExLjUzMTMtNTUuMjgxMnoiIGZpbGwtcnVsZT0iZXZlbm9kZCIgZmlsdGVyPSJ1cmwoI2ZpbHRlcjc4MjItMSkiIG9wYWNpdHk9Ii42ODI2OSIgdHJhbnNmb3JtPSJtYXRyaXgoMS4xMjE5LDAsMCwxLjEyMTksLTgzLjU5NzEsLTk5LjM3NTQ5KSIvPgo8cGF0aCBkPSJtNzQ4LjEyMzU4LDYxMS41NzkxOGMtMmUtMDA1LDAtMzguNjM1NDIsODEuMzcxLTcyLjEyNSw5MC41MzEyLTE1LjQ0MDQ2LDQuMjIzNC00MS41MDg1NC02LjkzNzItNjEuNDY4NzUtMTcuNTYyNC0yMC42NzAzNy0xOC4xNjI0LTM2Ljk2ODc1LTM1LjU2MjYtMzYuOTY4NzUtMzUuNTYyNnM0NC4xNzY4NCw5MS4xMDg1IDMyLjg0Mzc1LDExOC4zMTI2Yy0uMDQ3Ni4xMTQzLS4xMDU3Mi4yMjk3LS4xNTYyNS4zNDM3LTMwLjQ5NjA2LDE0LjE3MzItOTkuNzE4NzUsMTMuNjU2My05OS43MTg3NSwxMy42NTYzczIxLjU0MDI4LDcuMjY2NSA0My4zNDM3NSwxNy40MDYyYy0zNC44NDY4NywxMC4xOTcxLTcyLjU2MjUsMTUuNDY4OC03Mi41NjI1LDE1LjQ2ODhzOTcuMzY0OCwxNC44Njg4IDExMC4wMzEyNSwzNy40Mzc0YzEuMzMzMjQsMi4zNzU2IDIuNDYwMzYsNS4wMjEzIDMuMzc1LDcuODc1LTEzLjkwMjIzLDM4LjA1NDUtNTQuNSw5MC4wOTM4LTU0LjUsOTAuMDkzOHMyNy4zMjkxMi0xOC45ODYxIDU2LjAzMTI1LTMzLjkwNjJjLTQuMTgzMzUsMzIuMTEyNC0xMy4zNzUsNjIuMDMxMi0xMy4zNzUsNjIuMDMxMnM1NS4zMTk4OC01Ny4xNTM0IDg2LjI1LTU2LjUzMTJjMi40ODQzOS4wNDk5IDUuMTAyMzQuNDcwMyA3LjgxMjUsMS4xNTYyIDIyLjc3ODMxLDM1LjE5MTUgNDEuNSw5Ni41NjI1MiA0MS41LDk2LjU2MjUycy0uMjA2MDgtMzYuNzg1ODIgNC4yODEyNS02OS4wNjI1MmMyMS40ODYwOCwxNy44ODg1IDM5LjAzMTI1LDM3LjUzMTIgMzkuMDMxMjUsMzcuNTMxMnMtMjMuNDk2NzktNTYuMzA4OS0xNy45MDYyNS04Ni42ODc0YzIyLjYxODExLTEyLjQ1OSA2NC43Njk4Mi0xMi4xODAxIDkyLjY4NzUtMTAuMzEyNiAyMy42MTk3LDcuODY0IDQyLjQzNzUsMTYuNzE4OCA0Mi40Mzc1LDE2LjcxODhzLTguMzk0ODEtNS43NzM1LTE5LjYyNS0xNC42NTYyYzIuOTc3MzMuMzU0MSA0Ljc1LjU5MzcgNC43NS41OTM3cy0yNi4yODgyMS0xOC40NTM5LTQ2LjUzMTI1LTM5Ljc4MTNjLTkuMDA2MDgtMTAuNzk3MS0xNS42NzU1MS0yMS43OTQzLTE2Ljc1LTMxLjUtMy4xNDcyMy0yOC40Mjg2IDQ3LjQzNzUtODIuNzE4NyA0Ny40Mzc1LTgyLjcxODdzLTEzLjU4OTU0LDguNzk5My0zMS4xODc1LDE2LjM0MzdjMTMuNzI1NjktMjIuMjg5IDI4LjYyNS00MC41MzEyIDI4LjYyNS00MC41MzEycy03My40MDk2MSw0NS4yNzYtOTcuNTYyNSwyNi44NzVjLTEuMzA5NDQtLjk5NzYtMi40OTgyMi0yLjE1OTUtMy41OTM3NS0zLjQzNzUtNS43MDIyNC0zNC40MTM2IDMuOTA2MjUtOTIuMzQzNyAzLjkwNjI1LTkyLjM0MzdzLTQuNzQyMjgsOS4wNDgxLTExLjg0Mzc1LDIwLjkzNzRjMy45MTg5NC0yOS4yMDMgMTEuNTMxMjUtNTUuMjgxMiAxMS41MzEyNS01NS4yODEyeiIgZmlsbD0idXJsKCNyYWRpYWxHcmFkaWVudDM5NDUpIiBmaWxsLXJ1bGU9ImV2ZW5vZGQiLz4KPHBhdGggZD0ibTczNC4wOTc2OCw2NzcuNzU4MTZjLTFlLTAwNSwwLTI5LjQ5MzU3LDU0LjQ1NTQyLTUyLjgxOTksNTkuNTIyNTYtMTAuNzU0NjYsMi4zMzYyNi0yOC4yNDQ0My02LjI3NzAzLTQxLjU2MDQxLTE0LjI5OTktMTMuNTI4NTYtMTMuMjIzNDItMjQuMDgzMzQtMjUuNzY0MDYtMjQuMDgzMzQtMjUuNzY0MDZzMjcuMDA1Niw2NC4xNjE4NSAxOC4yMzI0NSw4Mi40MjUyNmMtLjAzNjkuMDc2Ny0uMDgxLjE1Mzg0LS4xMTk4LjIzMDI2LTIxLjQ1NDAyLDguNjE3OTktNjguOTU5NzEsNS43Mzc3My02OC45NTk3MSw1LjczNzczczE0LjUyMzI5LDUuNzc0NjMgMjkuMTIyNDYsMTMuNTMxNDRjLTI0LjI5Niw1LjcyOTQ4LTUwLjM4MTg0LDcuOTcyOC01MC4zODE4NCw3Ljk3MjhzNjYuMzAyOTgsMTMuNzYwMTggNzQuMTc1NzQsMjkuNzE2NjVjLjgyODY1LDEuNjc5NjEgMS41MDU5NSwzLjUzNzEyIDIuMDI5NzksNS41Mjk2OS0xMC45MzI4NCwyNS42MTg5Ny00MC43MDM1Nyw1OS44NjUyNy00MC43MDM1Nyw1OS44NjUyN3MxOS40NTUzNS0xMi4wMzc4MiAzOS43MDUtMjEuMjM0MDVjLTQuMDQzNTksMjEuODk0MDQtMTEuNDQ1NTcsNDIuMDk5MzQtMTEuNDQ1NTcsNDIuMDk5MzRzNDAuMDY0NzEtMzcuMjIwMjcgNjEuMjc2OTYtMzUuNjY0NzFjMS43MDM4My4xMjQ5MSAzLjQ4NTg0LjUwOTAzIDUuMzIxNDYsMS4wNzg4MSAxNC4zNTQ1MSwyNC45OTE1OSAyNC45Njg4Nyw2Ny44MDg2MSAyNC45Njg4Nyw2Ny44MDg2MXMxLjIwMDUzLTI1LjI2MjY4IDUuNDU4ODEtNDcuMjU4NGMxNC4wOTg1OCwxMy4wNjUxMyAyNS40Mjc1MywyNy4xOTA4MyAyNS40Mjc1MywyNy4xOTA4M3MtMTQuMDc3MzctMzkuNTE1ODYtOS4xMzA5My02MC4xNjgxNGMxNS45ODI5LTcuNzI4NTIgNDQuOTExODEtNS45OTkyNiA2NC4wMTA0NS0zLjY5ODY2IDE1LjkyOTExLDYuMjYwNzEgMjguNTI1MzYsMTMuMDI2NDMgMjguNTI1MzYsMTMuMDI2NDNzLTUuNTUyOC00LjI3MDAxLTEyLjkzODc3LTEwLjc3ODExYzIuMDMxMTUuMzUxNzQgMy4yMzk0Mi41ODA5IDMuMjM5NDIuNTgwOXMtMTcuMzc0ODMtMTMuNjI4NS0zMC40OTQ1NC0yOS4wMDkyNWMtNS43ODkxOC03Ljc0MTI2LTkuOTY2ODUtMTUuNTM0NjctMTAuMzUwNDUtMjIuMjM3MjctMS4xMjM1OS0xOS42MzIzNSAzNS41ODU3Ni01NS4wNTk1OCAzNS41ODU3Ni01NS4wNTk1OHMtOS42NTA4NSw1LjU0NTM2LTIyLjAwNzkxLDEwLjA4MjkyYzEwLjIzNjQ3LTE0LjgwMTY4IDIxLjEzMTA1LTI2Ljc4MjIzIDIxLjEzMTA1LTI2Ljc4MjIzcy01Mi4wNTA4MywyOC40MDU5Mi02Ny45NjE2LDE0Ljg5MTY1Yy0uODYyNTktLjczMjY3LTEuNjM2MzYtMS41NzM3NC0yLjM0MTg2LTIuNDkxMTItMi42NTkzOC0yMy44MzQ1NSA2LjA1MDctNjMuMjU1NjkgNi4wNTA3LTYzLjI1NTY5cy0zLjU4NTg4LDYuMDM4OTQtOC44OTUxMSwxMy45NDI0MWMzLjc1NTkxLTE5LjkwNjI0IDkuOTMzNS0zNy41MzIzOSA5LjkzMzUtMzcuNTMyMzl6IiBmaWxsPSJ1cmwoI3JhZGlhbEdyYWRpZW50Mzk0NykiIGZpbGwtcnVsZT0iZXZlbm9kZCIgb3BhY2l0eT0iLjYxMDU4Ii8+CjwvZz4KPC9zdmc+Cg==') 10 20, auto;
}

</style>

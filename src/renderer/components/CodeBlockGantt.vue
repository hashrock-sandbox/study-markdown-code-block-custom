<template>
  <svg class="gantt" :width="svgWidth" :height="tasks.length * 32 + 48" @pointermove="onDrag" @pointerup="stopDrag">
    <!-- 全体を32px下げる（日付用余白） -->
    <g transform="translate(0, 48)">
      <!-- 背景 -->
      <rect class="background" x="0" y="0" :width="svgWidth" :height="tasks.length * 32"></rect>
      <!-- 月 -->
      <text v-for="(line, index) in lines" :x="line.x" y="-28" text-anchor="start" font-weight="900" font-size="0.8rem" fill="#9C9" :key="index">{{line.labelMonth}}</text>
      <!-- 日付 -->
      <text v-for="(line, index) in lines" :x="line.x + 12" y="-8" text-anchor="middle" font-size="0.8rem" :fill="line.color" :key="index">{{line.label}}</text>
      <!-- 日付区切り線 -->
      <line v-for="(line, index) in lines" :x1="line.x" y1="0" :x2="line.x" :y2="tasks.length * 32" class="gridline" :key="index" />
      <!-- タスク -->
      <g v-for="(task, index) in tasks" :transform="`translate(${scale(task.start)}, ${index * 32})`" :key="index" :class="{'dragging': index === selectedIndex}">
        <rect class="task" x="0" y="4" :width="scaleLength(task.end - task.start)" height="24" @pointerdown="startDrag($event, index)"></rect>
        <text class="taskname" x="-4" y="20" font-size="12" text-anchor="end" fill="black" line-height="32">{{task.name}}</text>
      </g>
      <rect v-if="dragoverIndex > -1 && dragoverIndex !== selectedIndex" class="dragover" x="0" :y="32 * dragoverIndex" :width="svgWidth" height="32"></rect>
    </g>
  </svg>
</template>
<script>
  import * as gantt from "./gantt-compiler"
  import * as util from "./gantt-util.js"
  import * as scale from "d3-scale"

  export default {
    props: {
      "input": String
    },
    data() {
      return {
        tasks: [],
        lines: [],
        displayRange: {
          start: -2,
          end: 24
        },
        svgWidth: 600,
        selectedIndex: -1,
        dragOffset: {
          x: 0,
          y: 0
        },
        dragging: "none",
        dragoverIndex: -1
      }
    },
    methods: {
      onDrag(e) {
        if (this.dragging === "move") {
          const len = this.selectedItem.end - this.selectedItem.start
          //差分値を基点に反映
          this.selectedItem.start = this.invert(e.offsetX - this.dragOffset.x)
          this.selectedItem.end = this.selectedItem.start + len

          this.dragoverIndex = Math.floor((e.offsetY - 48)/32)
        }
        if(this.dragging === "resize-x"){
          this.selectedItem.end = this.invert(e.offsetX)
        }
      },
      startDrag(e, index) {
        this.dragging = "move"
        this.selectedIndex = index
        //ページ左上とオブジェクト左上の差分から、ドラッグ開始位置（オブジェクト相対座標）を取得
        this.dragOffset.x = e.offsetX - this.scale(this.selectedItem.start)
        this.dragOffset.y = e.offsetY - index * 32 - 48

        const len = this.selectedItem.end - this.selectedItem.start
        if (e.offsetX > this.scale(this.selectedItem.end) - 10) {
          this.dragging = "resize-x"
        }
      },      
      stopDrag() {
        if (this.dragging !== "none") {
          this.selectedItem.start = util.roundHMSfromEpoc(this.selectedItem.start)
          this.selectedItem.end = util.roundHMSfromEpoc(this.selectedItem.end)
        }
        if (this.dragging === "move") {
          if(this.selectedIndex !== this.dragoverIndex){
            const task = this.tasks.splice(this.selectedIndex, 1)
            this.tasks.splice(this.dragoverIndex, 0, task[0])
          }
        }
        if(this.dragging !== "none"){
          this.$emit("change", gantt.serialize(this.tasks));          
        }

        this.dragging = "none"
        this.selectedIndex = -1;
        this.dragoverIndex = -1;
        
      },
      scaleLength(epocdiff) {
        return epocdiff / (24 * 60 * 60 * 1000) * this.svgWidth / this.displayRangeLength
      },
      scale(epoc) {
        return scale.scaleLinear().domain(this.timeRange).range([0, this.svgWidth])(epoc)
      },
      invert(x) {
        return scale.scaleLinear().domain(this.timeRange).range([0, this.svgWidth]).invert(x)
      },
      generateLine() {
        let lines = []
        const start = this.timeRange[0];
        const end = this.timeRange[1];
        const len = end - start;
        let month = -1;
        for (let i = 0; i < this.displayRangeLength; i++) {
          const reldate = util.getRelativeDate(this.displayRange.start + i)
          const t = (reldate.getTime() - start) / len * this.svgWidth;
          let color = "#888888";
          if (reldate.getDay() === 0) {
            color = "#FF8888";
          }
          if (reldate.getDay() === 6) {
            color = "#8888FF";
          }
          let monthStr = ""
          if(month != reldate.getMonth()+1){
            month = reldate.getMonth()+1
            monthStr = reldate.getMonth()+1 + "月"
          }

          lines.push({ x: Math.round(t), label: reldate.getDate(), color: color, labelMonth: monthStr})
        }
        this.lines = lines;
      },
      setTasks(input){
        this.tasks = gantt.compile(input)
      },
    },
    watch: {
      input(){
        this.setTasks(this.input)
      }
    },
    computed: {
      selectedItem(){
        return this.tasks[this.selectedIndex]
      },
      timeRange() {
        return [
          util.getRelativeDate(this.displayRange.start).getTime(),
          util.getRelativeDate(this.displayRange.end).getTime()
        ]
      },
      displayRangeLength() {
        return (this.displayRange.end - this.displayRange.start);
      }
    },
    mounted() {
      this.generateLine();
      this.setTasks(this.input)
    }
  }
</script>
<style>
  .task {
    fill: rgb(144, 144, 255);
    cursor: pointer;
  }

  .background {
    fill: #f5f5f5;
  }

  .gridline {
    stroke: rgb(253, 253, 253);
    stroke-width: 2
  }
  svg.gantt{
    cursor: default;
    user-select: none;
  }
  .taskname{
    cursor: default;
  }
  .dragging{
    opacity: 0.5;
  }
  .dragover{
    opacity: 0.1;
  }
</style>
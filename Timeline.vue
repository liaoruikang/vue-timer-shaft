<template>
  <div class="timeline_container" ref="mainRef" @mousedown="down" @wheel="wheel">
    <div class="scaleplate" ref="scaleplateRef"></div>
    <span class="currentTime">{{ currentDate + ' ' + currentTime }}</span>
    <div class="timeline_content" ref="axisRef" :style="{ left: left + 'px' }" @click="jump">
      <div class="time" v-for="(item, index) in lineList" :this.key="index" :style="{ left: item.left + 'px' }">
        <span class="time_item" :style="!timeLimit(item.timestamp - 86400000, 'boolean') && 'color: #99999950'">{{ item.time }}</span>
        <span :class="item.lineType"></span>
      </div>
    </div>
  </div>
</template>
<script>
export default {
  name: 'timeline',
  props: {
    currentDate: {
      type: String,
      default: '',
    },
  },
  data() {
    return {
      lineList: [],
      left: 0,
      infoWidth: 0,
      currentTime: '00:00:00',
      currentTimestamp: 0,
      proportion: 120,
      zoom: 120,
      isMove: false,
    }
  },
  watch: {
    currentDate() {
      this.currentTimestamp = this.timeLimit(this.currentTimestamp)
      this.currentTime = this.formatTime(this.currentTimestamp)
      this.$refs.axisRef.style.transition = 'left 0.3s'
      setTimeout(() => {
        this.$refs.axisRef.style.transition = 'none'
      }, 300)
      this.location()
      this.initAxis()
    },
  },
  mounted() {
    this.initAxis()
    window.addEventListener('resize', this.resize)
    window.addEventListener('mouseup', this.up)
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.resize)
  },
  methods: {
    resize() {
      this.initAxis()
      this.location()
    },
    initAxis() {
      this.infoWidth = (120 / this.proportion) * this.$refs.mainRef.offsetWidth
      this.$refs.axisRef.style.width = this.infoWidth * 2 + 'px'
      const { min, max } = this.getTimestampLimit()
      let timestamp = min
      let i = 0
      this.lineList = []
      while (timestamp <= max) {
        const left = (timestamp / 86400000) * this.infoWidth - this.infoWidth / 2
        this.lineList.push({
          time: i % 10 == 0 ? this.formatTime(timestamp) : '',
          timestamp,
          left,
          lineType: i % 10 == 0 ? 'line' : 'simpleLine',
        })
        timestamp += 86400000 / this.zoom
        i++
      }
    },
    formatTime(time, type = 'time') {
      if (type == 'time') {
        if (typeof time != 'number') return null
        let h = parseInt((time / 1000 / 60 / 60) % 24)
        let m = parseInt((time / 1000 / 60) % 60)
        let s = parseInt((time / 1000) % 60)
        h = h < 10 ? `0${h}` : h
        m = m < 10 ? `0${m}` : m
        s = s < 10 ? `0${s}` : s
        return `${h}:${m}:${s}`
      } else if (type == 'timestamp') {
        const date = new Date()
        let timestamp = date.getHours() * 60 * 60 * 1000
        timestamp += date.getMinutes() * 60 * 1000
        timestamp += date.getSeconds() * 1000
        return timestamp
      }
    },
    location() {
      const x = (this.currentTimestamp / 86400000) * this.infoWidth + this.infoWidth / 2 - this.$refs.mainRef.offsetWidth / 2
      this.left = -x
    },
    down(e) {
      window.addEventListener('mousemove', this.move)
      const rect = this.$refs.mainRef.getBoundingClientRect()
      this.downX = e.pageX - rect.left
    },
    move(e) {
      this.isMove = true
      const rect = this.$refs.mainRef.getBoundingClientRect()
      let x = e.pageX - rect.left
      if (x < this.downX) {
        this.currentTimestamp += 1000 * this.proportion * Math.abs(this.downX - x)
      } else {
        this.currentTimestamp -= 1000 * this.proportion * Math.abs(this.downX - x)
      }
      this.currentTimestamp = this.timeLimit(this.currentTimestamp)
      this.downX = x
      this.currentTime = this.formatTime(this.currentTimestamp)
      this.location()
      this.initAxis()
    },
    up() {
      setTimeout(() => {
        this.isMove && this.emit()
        this.isMove = false
      }, 0)
      window.removeEventListener('mousemove', this.move)
    },
    wheel(e) {
      if (e.wheelDelta > 0) {
        // 缩小刻度
        if (this.proportion <= 1 && this.proportion > 0.8) {
          this.proportion -= 0.1
        } else if (this.proportion <= 30 && this.proportion > 1) {
          this.proportion = this.proportion - 0.5
        } else if (this.proportion > 1) {
          this.proportion = this.proportion - 1
        }
      } else {
        // 放大刻度
        if (this.proportion <= 1 && this.proportion > 0.8) {
          this.proportion += 0.1
        } else if (this.proportion <= 30) {
          this.proportion = this.proportion + 0.5
        } else if (this.proportion < 120) {
          this.proportion = Math.round(this.proportion) + 1
        }
      }
      if (this.proportion > 90) {
        this.zoom = 120 * (120 / 120)
      } else if (this.proportion <= 90 && this.proportion > 60) {
        this.zoom = 120 * (120 / 90)
      } else if (this.proportion <= 60 && this.proportion > 45) {
        this.zoom = 120 * (120 / 60)
      } else if (this.proportion <= 45 && this.proportion > 30) {
        this.zoom = 120 * (120 / 45)
      } else if (this.proportion <= 30 && this.proportion > 20) {
        this.zoom = 120 * (120 / 30)
      } else if (this.proportion <= 20 && this.proportion > 15) {
        this.zoom = 120 * (120 / 20)
      } else if (this.proportion <= 15 && this.proportion > 10) {
        this.zoom = 120 * (120 / 15)
      } else if (this.proportion <= 10 && this.proportion > 5) {
        this.zoom = 120 * (120 / 10)
      } else if (this.proportion <= 5 && this.proportion > 3) {
        this.zoom = 120 * (120 / 5)
      } else if (this.proportion <= 3 && this.proportion > 1) {
        this.zoom = 120 * (120 / 3)
      } else {
        this.zoom = 120 * (120 / 1)
      }
      this.initAxis()
      this.location()
    },
    jump(e) {
      if (this.isMove) return
      const rect = this.$refs.mainRef.getBoundingClientRect()
      const offsetX = e.pageX - rect.left + Math.abs(this.left) - this.infoWidth / 2
      this.currentTimestamp = this.timeLimit((offsetX / this.infoWidth) * 86400000)
      this.currentTime = this.formatTime(this.currentTimestamp)
      this.emit()
      this.$refs.axisRef.style.transition = 'left 0.3s'
      setTimeout(() => {
        this.$refs.axisRef.style.transition = 'none'
      }, 300)
      this.location()
      this.initAxis()
      // this.isShow()
    },
    reset() {
      this.currentTimestamp = 0
      this.currentTime = this.formatTime(this.currentTimestamp)
      this.location()
      this.initAxis()
    },
    calibration(timestamp) {
      if (timestamp >= 86400000) return 86400000
      const h = Math.floor((timestamp / 1000 / 60 / 60) % 24) * 1000 * 60 * 60
      const m = Math.floor((timestamp / 1000 / 60) % 60) * 1000 * 60
      const s = Math.floor((timestamp / 1000) % 60) * 1000
      return h + m + s
    },
    timeLimit(timestamp, type = 'deafult') {
      const today = `${new Date().getFullYear()}-${new Date().getMonth() + 1}-${new Date().getDate()}`
      if (this.currentDate == today) {
        if (type == 'deafult') {
          timestamp = this.calibration(timestamp)
          return timestamp < 0 ? 0 : timestamp > this.formatTime(null, 'timestamp') ? this.formatTime(null, 'timestamp') : timestamp
        } else if (type == 'boolean') {
          return timestamp < 0 ? false : timestamp > this.formatTime(null, 'timestamp') ? false : true
        }
      } else {
        if (type == 'deafult') {
          timestamp = this.calibration(timestamp)
          return timestamp < 0 ? 0 : timestamp > 86400000 ? 86400000 : timestamp
        } else if (type == 'boolean') {
          return timestamp < 0 ? false : timestamp > 86400000 ? false : true
        }
      }
    },
    getTimestampLimit() {
      const x = (this.currentTimestamp / 86400000) * this.infoWidth + this.infoWidth / 2 - this.$refs.mainRef.offsetWidth / 2
      let min = ((Math.abs(x) + this.infoWidth / 2) / this.infoWidth) * 86400000
      let max = ((Math.abs(x) + this.infoWidth / 2 + this.$refs.mainRef.offsetWidth) / this.infoWidth) * 86400000
      min = parseInt(min / 1000 / 60 / 60) * 1000 * 60 * 60
      if (this.zoom == 120 * (120 / 120)) {
        if ((min / 1000 / 60 / 60) % 2 !== 0) {
          min = (min / 1000 / 60 / 60 - 1) * 1000 * 60 * 60
        }
      } else if (this.zoom == 120 * (120 / 90)) {
        if ((min / 1000 / 60 / 60) % 3 !== 0) {
          min = (min / 1000 / 60 / 60 - ((min / 1000 / 60 / 60) % 3)) * 1000 * 60 * 60
        }
      } else if (this.zoom == 120 * (120 / 45)) {
        if ((min / 1000 / 60 / 60) % 3 !== 0) {
          min = (min / 1000 / 60 / 60 - ((min / 1000 / 60 / 60) % 6)) * 1000 * 60 * 60
        }
      }
      return {
        min,
        max,
      }
    },
    emit() {
      this.$emit('change', {
        start_time: this.currentTime,
        start_timestamp: this.currentTimestamp,
      })
    },
  },
}
</script>
<style lang="less" scoped>
.timeline_container {
  position: relative;
  height: 45px;
  background-color: #3a68a3;
  margin-top: 10px;
  overflow: hidden;
  cursor: grab;
  user-select: none;
  -webkit-user-select: none;
  &:active {
    cursor: grabbing;
  }
  .scaleplate {
    position: absolute;
    width: 1px;
    height: 100%;
    left: 50%;
    top: 0;
    background-color: #fff;
    z-index: 10;
  }
  .currentTime {
    position: absolute;
    left: 50%;
    top: 0;
    transform: translateX(-50%);
    z-index: 10;
    font-size: 13px;
    color: #fff;
  }
  .timeline_content {
    position: relative;
    background-color: #fdd894;
    height: 10px;
    top: 28px;
    margin: 0;
    cursor: pointer;
    .time {
      position: absolute;
      width: 1px;
      height: 40px;
      flex-direction: column;
      align-items: center;
      margin-top: -15px;
      .line {
        display: block;
        height: 25px;
        width: 1px;
        background-color: rgba(255, 255, 255, 0.5);
        margin-top: 14px;
      }
      .simpleLine {
        display: block;
        background-color: rgba(255, 255, 255, 0.5);
        width: 1px;
        height: 13px;
        margin-top: 14px;
      }
      .time_item {
        position: absolute;
        transform: translateX(-50%);
        color: rgba(255, 255, 255, 0.5);
        font-size: 12px;
      }
    }
  }
}
</style>

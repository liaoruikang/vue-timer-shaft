<template>
  <div
    class="timeline_container"
    ref="mainRef"
    @mousedown="down"
    @wheel="wheel"
  >
    <div class="scaleplate" style="width: 1px" ref="scaleplateRef"></div>
    <span class="currentTime">{{ currentDate + " " + currentTime }}</span>
    <div
      class="timeline_content"
      ref="axisRef"
      :style="{ left: left + 'px' }"
      @click="jump"
    >
      <div
        class="time"
        v-for="(item, index) in lineList"
        :this.key="index"
        :style="{ left: item.left + 'px', width: '1px' }"
      >
        <span
          class="time_item"
          :style="
            !infinite &&
            !timeLimit(item.timestamp - 86400000, 'boolean') &&
            'color: #99999950'
          "
          >{{ item.time }}</span
        >
        <span
          :class="item.lineType"
          :style="
            !infinite &&
            !timeLimit(item.timestamp - 86400000, 'boolean') &&
            'background-color: #ffffff30'
          "
        ></span>
      </div>
    </div>
  </div>
</template>
<script>
export default {
  name: "timeline",
  props: {
    currentDate: {
      type: String,
      default: "",
    },
    value: {
      type: String,
      default: "00:00:00",
    },
    infinite: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      lineList: [],
      left: 0,
      infoWidth: 0,
      currentTime: this.value,
      currentTimestamp: 0,
      proportion: 120,
      zoom: 120,
      isMove: false,
      timer: null,
      intervalTimer: null,
      animateTimer: null,
    };
  },
  watch: {
    currentDate() {
      this.currentTimestamp = this.timeLimit(this.currentTimestamp);
      this.currentTime = this.formatTime(this.currentTimestamp);
      if (!this.infinite) {
        this.$nextTick(() => {
          this.$emit("input", this.currentTime);
          this.$emit("change", {
            start_time: this.currentTime,
            start_timestamp: this.currentTimestamp,
          });
        });
        this.$refs.axisRef.style.transition = "left 0.3s";
        setTimeout(() => {
          this.$refs.axisRef.style.transition = "none";
        }, 300);
        this.location();
        this.initAxis();
      }
    },
  },
  mounted() {
    // 初始化时间
    this.initAxis();
    this.currentTimestamp = this.timeLimit(
      this.formatTime(this.currentTime, "timestamp")
    );
    this.currentTime = this.formatTime(this.currentTimestamp);
    this.location();
    this.initAxis();

    window.addEventListener("resize", this.resize);
    window.addEventListener("mouseup", this.up);
    window.addEventListener("keydown", this.keydown);
    window.addEventListener("keyup", this.keyup);
    clearInterval(this.intervalTimer);
    this.intervalTimer = setInterval(() => {
      this.$forceUpdate();
    }, 1000);
  },
  beforeDestroy() {
    window.removeEventListener("resize", this.resize);
    window.removeEventListener("mouseup", this.up);
    window.removeEventListener("keydown", this.keydown);
    window.addEventListener("keyup", this.keyup);
    clearInterval(this.intervalTimer);
  },
  methods: {
    resize() {
      this.initAxis();
      this.location();
    },
    initAxis(delay = { ld: 0, rd: 0 }) {
      this.infoWidth = (120 / this.proportion) * this.$refs.mainRef.offsetWidth;
      this.$refs.axisRef.style.width = this.infoWidth * 3 + "px";
      const { min, max } = this.getTimestampLimit(delay);
      let timestamp = min;
      let i = 0;
      this.lineList = [];
      while (timestamp <= max) {
        const left = (timestamp / 86400000) * this.infoWidth;
        const x =
          (this.currentTimestamp / 86400000) * this.infoWidth +
          this.infoWidth -
          this.$refs.mainRef.offsetWidth / 2;
        if (left >= x - 30 && left <= x + this.$refs.mainRef.offsetWidth + 30) {
          this.lineList.push({
            time: i % 10 == 0 ? this.formatTime(timestamp) : "",
            timestamp,
            left,
            lineType: i % 10 == 0 ? "line" : "simpleLine",
          });
        }
        timestamp += 86400000 / this.zoom;
        i++;
      }
    },
    formatTime(time, type = "time") {
      if (type == "time") {
        if (typeof time != "number") return null;
        let h = parseInt((time / 1000 / 60 / 60) % 24);
        let m = parseInt((time / 1000 / 60) % 60);
        let s = parseInt((time / 1000) % 60);
        h = h < 10 ? `0${h}` : h;
        m = m < 10 ? `0${m}` : m;
        s = s < 10 ? `0${s}` : s;
        return `${h}:${m}:${s}`;
      } else if (type == "timestamp") {
        if (!isNaN(time?.split(":")[0])) {
          let [h, m, s] = time.split(":").map((item) => {
            return parseInt(item);
          });
          h = h * 1000 * 60 * 60;
          m = m * 1000 * 60;
          s = s * 1000;
          return h + m + s;
        } else {
          const date = new Date();
          let timestamp = date.getHours() * 60 * 60 * 1000;
          timestamp += date.getMinutes() * 60 * 1000;
          timestamp += date.getSeconds() * 1000;
          return timestamp;
        }
      }
    },
    location() {
      const x =
        (this.currentTimestamp / 86400000) * this.infoWidth +
        this.infoWidth -
        this.$refs.mainRef.offsetWidth / 2;
      this.left = -x;
    },
    down(e) {
      window.addEventListener("mousemove", this.move);
      const rect = this.$refs.axisRef.getBoundingClientRect();
      this.downX = e.pageX - rect.left;
    },
    move(e) {
      this.isMove = true;
      const rect = this.$refs.mainRef.getBoundingClientRect();
      let x = e.pageX - rect.left - this.downX;
      if (x > -(this.infoWidth - this.$refs.mainRef.offsetWidth / 2)) {
        // 是否无限滚动
        if (this.infinite) {
          x = -(
            this.infoWidth +
            this.infoWidth / 2 -
            this.$refs.mainRef.offsetWidth / 2
          );
          this.downX =
            this.downX +
            (this.infoWidth / 2 +
              this.infoWidth -
              this.$refs.mainRef.offsetWidth / 2);
          const date = new Date(
            new Date(this.currentDate).getTime() - 86400000
          );
          const d = `${date.getFullYear()}-${
            date.getMonth() + 1
          }-${date.getDate()}`;
          this.$emit("update:currentDate", d);
        } else {
          x = -(this.infoWidth - this.$refs.mainRef.offsetWidth / 2);
        }
      }
      // 是否无限滚动
      if (this.infinite) {
        if (
          x <
          -(
            this.infoWidth +
            this.infoWidth -
            this.$refs.mainRef.offsetWidth / 2
          )
        ) {
          x = -(this.infoWidth - this.$refs.mainRef.offsetWidth / 2);
          this.downX = this.downX - this.infoWidth;
          const date = new Date(
            new Date(this.currentDate).getTime() + 86400000
          );
          const d = `${date.getFullYear()}-${
            date.getMonth() + 1
          }-${date.getDate()}`;
          this.$emit("update:currentDate", d);
        }
      } else {
        if (
          x <
          -(
            this.infoWidth * (this.timeLimit() / 86400000) +
            this.infoWidth -
            this.$refs.mainRef.offsetWidth / 2
          )
        ) {
          x = -(
            this.infoWidth * (this.timeLimit() / 86400000) +
            this.infoWidth -
            this.$refs.mainRef.offsetWidth / 2
          );
        }
      }
      this.currentTimestamp =
        ((Math.abs(x) - this.infoWidth + this.$refs.mainRef.offsetWidth / 2) /
          this.infoWidth) *
        86400000;
      this.currentTimestamp = this.calibration(this.currentTimestamp);
      this.currentTime = this.formatTime(this.currentTimestamp);
      this.left = x;
      this.initAxis();
      // if (x < this.downX) {
      //   this.currentTimestamp += 500 * this.proportion * Math.abs(this.downX - x)
      // } else {
      //   this.currentTimestamp -= 500 * this.proportion * Math.abs(this.downX - x)
      // }
      // this.currentTimestamp = this.timeLimit(this.currentTimestamp)
      // this.downX = x
      // this.currentTime = this.formatTime(this.currentTimestamp)
      // this.location()
      // this.initAxis()
    },
    up() {
      setTimeout(() => {
        if (this.isMove) {
          this.$emit("input", this.currentTime);
          this.$emit("change", {
            start_time: this.currentTime,
            start_timestamp: this.currentTimestamp,
          });
        }
        this.isMove = false;
      }, 0);
      window.removeEventListener("mousemove", this.move);
    },
    wheel(e) {
      if (e.wheelDelta > 0) {
        // 缩小刻度
        if (this.proportion > 90) {
          this.proportion = this.proportion - 2;
        } else if (this.proportion <= 90 && this.proportion > 60) {
          this.proportion = this.proportion - 1.8;
        } else if (this.proportion <= 60 && this.proportion > 45) {
          this.proportion = this.proportion - 1.2;
        } else if (this.proportion <= 45 && this.proportion > 30) {
          this.proportion = this.proportion - 0.9;
        } else if (this.proportion <= 30 && this.proportion > 20) {
          this.proportion = this.proportion - 0.6;
        } else if (this.proportion <= 20 && this.proportion > 15) {
          this.proportion = this.proportion - 0.4;
        } else if (this.proportion <= 15 && this.proportion > 10) {
          this.proportion = this.proportion - 0.3;
        } else if (this.proportion <= 10 && this.proportion > 5) {
          this.proportion = this.proportion - 0.2;
        } else if (this.proportion <= 5 && this.proportion > 3) {
          this.proportion = this.proportion - 0.1;
        } else if (this.proportion <= 3 && this.proportion > 1) {
          this.proportion = this.proportion - 0.06;
        } else if (this.proportion > 0.5) {
          this.proportion = this.proportion - 0.02;
        }
      } else {
        // 放大刻度
        if (this.proportion < 120 && this.proportion > 90) {
          this.proportion = this.proportion + 2;
        } else if (this.proportion <= 90 && this.proportion > 60) {
          this.proportion = this.proportion + 1.8;
        } else if (this.proportion <= 60 && this.proportion > 45) {
          this.proportion = this.proportion + 1.2;
        } else if (this.proportion <= 45 && this.proportion > 30) {
          this.proportion = this.proportion + 0.9;
        } else if (this.proportion <= 30 && this.proportion > 20) {
          this.proportion = this.proportion + 0.6;
        } else if (this.proportion <= 20 && this.proportion > 15) {
          this.proportion = this.proportion + 0.4;
        } else if (this.proportion <= 15 && this.proportion > 10) {
          this.proportion = this.proportion + 0.3;
        } else if (this.proportion <= 10 && this.proportion > 5) {
          this.proportion = this.proportion + 0.2;
        } else if (this.proportion <= 5 && this.proportion > 3) {
          this.proportion = this.proportion + 0.1;
        } else if (this.proportion <= 3 && this.proportion > 1) {
          this.proportion = this.proportion + 0.06;
        } else {
          this.proportion = this.proportion + 0.02;
        }
        if (this.proportion > 120) {
          this.proportion = 120;
        }
      }
      if (this.proportion > 90) {
        this.zoom = 120 * (120 / 120);
      } else if (this.proportion <= 90 && this.proportion > 60) {
        this.zoom = 120 * (120 / 90);
      } else if (this.proportion <= 60 && this.proportion > 45) {
        this.zoom = 120 * (120 / 60);
      } else if (this.proportion <= 45 && this.proportion > 30) {
        this.zoom = 120 * (120 / 45);
      } else if (this.proportion <= 30 && this.proportion > 20) {
        this.zoom = 120 * (120 / 30);
      } else if (this.proportion <= 20 && this.proportion > 15) {
        this.zoom = 120 * (120 / 20);
      } else if (this.proportion <= 15 && this.proportion > 10) {
        this.zoom = 120 * (120 / 15);
      } else if (this.proportion <= 10 && this.proportion > 5) {
        this.zoom = 120 * (120 / 10);
      } else if (this.proportion <= 5 && this.proportion > 3) {
        this.zoom = 120 * (120 / 5);
      } else if (this.proportion <= 3 && this.proportion > 1) {
        this.zoom = 120 * (120 / 3);
      } else {
        this.zoom = 120 * (120 / 1);
      }
      this.initAxis();
      this.location();
    },
    async jump(e) {
      if (this.isMove) return;
      const rect = this.$refs.mainRef.getBoundingClientRect();
      const offsetX =
        e.pageX - rect.left + Math.abs(this.left) - this.infoWidth;
      this.currentTimestamp = (offsetX / this.infoWidth) * 86400000;
      this.currentTime = this.formatTime(this.currentTimestamp);
      // 如果开启无限轴则先执行动画后再进行无缝跳转
      if (this.infinite) {
        this.location();
        if (this.currentTimestamp < 0) {
          let step = Math.abs(this.currentTimestamp);
          this.initAxis({
            ld: step,
            rd: 0,
          });
        } else {
          this.initAxis({
            ld: 0,
            rd: 0,
          });
        }
        this.currentTimestamp = this.timeLimit(this.currentTimestamp);
        this.currentTime = this.formatTime(this.currentTimestamp);
        this.$refs.axisRef.style.transition = "left 0.3s";
        await new Promise((resolve) => {
          setTimeout(() => {
            this.$refs.axisRef.style.transition = "none";
            resolve();
          }, 300);
        });
      } else {
        this.$refs.axisRef.style.transition = "left 0.3s";
        setTimeout(() => {
          this.$refs.axisRef.style.transition = "none";
        }, 300);
      }
      this.currentTimestamp = this.timeLimit(this.currentTimestamp);
      this.currentTime = this.formatTime(this.currentTimestamp);
      this.location();
      this.initAxis();

      this.$emit("input", this.currentTime);
      this.$emit("change", {
        start_time: this.currentTime,
        start_timestamp: this.currentTimestamp,
      });
    },
    keydown(e) {
      switch (e.keyCode) {
        case 37:
          // 左方向
          this.currentTimestamp -= 1500 * this.proportion;
          this.currentTimestamp = this.timeLimit(this.currentTimestamp);
          this.currentTime = this.formatTime(this.currentTimestamp);
          this.initAxis();
          this.location();
          break;
        case 39:
          //  右方向
          this.currentTimestamp += 1500 * this.proportion;
          this.currentTimestamp = this.timeLimit(this.currentTimestamp);
          this.currentTime = this.formatTime(this.currentTimestamp);
          this.initAxis();
          this.location();
          break;
        case 107:
          //  放大
          this.wheel({
            wheelDelta: 1,
          });
          break;
        case 109:
          //  缩小
          this.wheel({
            wheelDelta: -1,
          });
          break;
      }
    },
    keyup(e) {
      if (e.keyCode == 37 || e.keyCode == 39) {
        clearTimeout(this.timer);
        this.$emit("input", this.currentTime);
        this.timer = setTimeout(() => {
          this.$emit("change", {
            start_time: this.currentTime,
            start_timestamp: this.currentTimestamp,
          });
        }, 300);
      }
    },
    reset() {
      this.currentTimestamp = 0;
      this.currentTime = this.formatTime(this.currentTimestamp);
      this.$emit("input", this.currentTime);
      this.location();
      this.initAxis();
    },
    calibration(timestamp) {
      if (timestamp >= 86400000) return 86400000;
      const h = Math.floor((timestamp / 1000 / 60 / 60) % 24) * 1000 * 60 * 60;
      const m = Math.floor((timestamp / 1000 / 60) % 60) * 1000 * 60;
      const s = Math.round((timestamp / 1000) % 60) * 1000;
      return h + m + s;
    },
    timeLimit(timestamp, type = "deafult") {
      if (this.infinite) {
        if (timestamp < 0) {
          const date = new Date(
            new Date(this.currentDate).getTime() - 86400000
          );
          const d = `${date.getFullYear()}-${
            date.getMonth() + 1
          }-${date.getDate()}`;
          this.$emit("update:currentDate", d);
          return 86399000 + timestamp;
        } else if (timestamp > 86400000) {
          const date = new Date(
            new Date(this.currentDate).getTime() + 86400000
          );
          const d = `${date.getFullYear()}-${
            date.getMonth() + 1
          }-${date.getDate()}`;
          this.$emit("update:currentDate", d);
          return timestamp - 86400000;
        } else {
          return timestamp;
        }
      }
      const today = `${new Date().getFullYear()}-${
        new Date().getMonth() + 1
      }-${new Date().getDate()}`;
      if (this.currentDate == today) {
        if (type == "deafult") {
          if (timestamp == undefined) return this.formatTime(null, "timestamp");
          timestamp = this.calibration(timestamp);
          return timestamp < 0
            ? 0
            : timestamp > this.formatTime(null, "timestamp")
            ? this.formatTime(null, "timestamp")
            : timestamp;
        } else if (type == "boolean") {
          return timestamp < 0
            ? false
            : timestamp > this.formatTime(null, "timestamp")
            ? false
            : true;
        }
      } else {
        if (type == "deafult") {
          if (timestamp == undefined) return 86399000;
          timestamp = this.calibration(timestamp);
          return timestamp < 0
            ? 0
            : timestamp > 86399000
            ? 86399000
            : timestamp;
        } else if (type == "boolean") {
          return timestamp < 0 ? false : timestamp > 86399000 ? false : true;
        }
      }
    },
    getTimestampLimit(delay) {
      const x =
        (this.currentTimestamp / 86400000) * this.infoWidth +
        this.infoWidth / 2 -
        this.$refs.mainRef.offsetWidth / 2;
      let min =
        ((Math.abs(x) + this.infoWidth / 2) / this.infoWidth) * 86400000 -
        delay.ld * 2;
      let max =
        ((Math.abs(x) + this.infoWidth / 2 + this.$refs.mainRef.offsetWidth) /
          this.infoWidth) *
          86400000 +
        delay.rd * 2;
      min = parseInt(min / 1000 / 60 / 60) * 1000 * 60 * 60;
      if (this.zoom == 120 * (120 / 120)) {
        if ((min / 1000 / 60 / 60) % 2 !== 0) {
          min = (min / 1000 / 60 / 60 - 1) * 1000 * 60 * 60;
        }
      } else if (this.zoom == 120 * (120 / 90)) {
        if ((min / 1000 / 60 / 60) % 3 !== 0) {
          min =
            (min / 1000 / 60 / 60 - ((min / 1000 / 60 / 60) % 3)) *
            1000 *
            60 *
            60;
        }
      } else if (this.zoom == 120 * (120 / 45)) {
        if ((min / 1000 / 60 / 60) % 3 !== 0) {
          min =
            (min / 1000 / 60 / 60 - ((min / 1000 / 60 / 60) % 6)) *
            1000 *
            60 *
            60;
        }
      }
      return {
        min,
        max,
      };
    },
  },
};
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
      height: 40px;
      flex-direction: column;
      align-items: center;
      margin-top: -15px;
      .line {
        display: block;
        height: 25px;
        background-color: rgba(255, 255, 255, 0.7);
        margin-top: 14px;
      }
      .simpleLine {
        display: block;
        background-color: rgba(255, 255, 255, 0.7);
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

<template>
  <div class="timeline_container" ref="main" @mousedown="down" @wheel="wheel">
    <div class="scaleplate" ref="scaleplateRef"></div>
    <span class="currentTime">{{ time + " " + currentTime }}</span>
    <div
      class="timeline_content"
      ref="contentRef"
      :style="{ left: left + 'px', width: inWidth + 'px' }"
    >
      <template v-for="(item, index) in lineList">
        <div
          class="time"
          v-if="item.show"
          :key="index"
          :style="{ left: item.left + 'px' }"
        >
          <span class="time_item">{{ item.time }}</span>
          <span class="line"></span>
        </div>
      </template>
    </div>
  </div>
</template>
<script>
export default {
  name: "timeline",
  data() {
    return {
      time: "",
      currentTime: "00:00:00",
      currentTimestamp: 0,
      isDown: false,
      left: 0,
      downX: 0,
      timer: null,
      inWidth: 0,
      zoom: 1,
      lineList: [],
      downTime: "",
      timer: null,
      // 精度 初始每15秒一像素
      precision: 15,
    };
  },
  props: {
    day: {
      type: Date,
      default: () => new Date(),
    },
  },
  watch: {
    day: {
      deep: true,
      immediate: true,
      handler() {
        let year = this.day.getFullYear();
        let month = this.day.getMonth() + 1;
        let day = this.day.getDate();
        month = month < 10 ? "0" + month : month + "";
        day = day < 10 ? "0" + day : day + "";
        this.time = `${year}-${month}-${day}`;
        this.$nextTick(() => {
          this.updateTimeLine();
        });
      },
    },
  },
  beforeDestroy() {
    clearInterval(this.timer);
  },
  mounted() {
    this.updateTimeLine();
    this.timer = setInterval(() => {
      this.updateTimeLine();
    }, 1000);
    this.init();
    window.addEventListener("mouseup", () => {
      if (this.isDown) this.up();
    });
    window.addEventListener("mousemove", this.move);
  },
  methods: {
    // 初始化位置
    init() {
      this.left = this.$refs.scaleplateRef.offsetLeft;
    },
    // 获取当前时分秒时间戳
    getTimestamp() {
      const currentH = new Date().getHours();
      const currentM = new Date().getMinutes();
      const currentS = new Date().getSeconds();
      let year = new Date().getFullYear();
      let month = new Date().getMonth() + 1;
      let day = new Date().getDate();
      month = month < 10 ? "0" + month : month + "";
      day = day < 10 ? "0" + day : day + "";
      if (this.time == `${year}-${month}-${day}`) {
        return (
          1000 * currentS + 1000 * 60 * currentM + 1000 * 60 * 60 * currentH
        );
      } else {
        return 1000 * 60 * 60 * 24;
      }
    },
    // 判断当前刻度是否在可视范围内
    isVisible(target) {
      if (
        target.left + this.left > -5 &&
        target.left + this.left < this.$refs.main.offsetWidth + 5
      )
        return true;
      return false;
    },
    // 初始化时间列表
    updateTimeLine() {
      const timestamp = this.getTimestamp();
      // 设置时间轴长度
      const newWidth =
        timestamp /
        1000 /
        (this.precision * this.zoom < 1 ? 1 : this.precision * this.zoom);
      this.inWidth = newWidth;
      const list = [];
      let newTimestamp = 0;
      while (newTimestamp <= timestamp) {
        // 将时间戳转换为时间格式
        time = this.formatTime(newTimestamp);
        // 计算每个时间线位置
        const proportion = newTimestamp / timestamp;
        const left = this.inWidth * proportion;
        // 获取当前刻度可视状态 只渲染可视范围内的刻度
        let show = this.isVisible({ left });
        if (left <= this.inWidth) {
          list.push({
            time,
            left,
            show,
          });
        }
        // 对不同精度显示不同时间段刻度
        const step = (this.precision * this.zoom) / this.precision;
        if (step <= 0.95 && step > 0.9) {
          newTimestamp += 1000 * 60 * 110;
        } else if (step <= 0.9 && step > 0.85) {
          newTimestamp += 1000 * 60 * 100;
        } else if (step <= 0.85 && step > 0.8) {
          newTimestamp += 1000 * 60 * 90;
        } else if (step <= 0.8 && step > 0.75) {
          newTimestamp += 1000 * 60 * 80;
        } else if (step <= 0.75 && step > 0.7) {
          newTimestamp += 1000 * 60 * 70;
        } else if (step <= 0.7 && step > 0.65) {
          newTimestamp += 1000 * 60 * 60;
        } else if (step <= 0.65 && step > 0.6) {
          newTimestamp += 1000 * 60 * 50;
        } else if (step <= 0.6 && step > 0.55) {
          newTimestamp += 1000 * 60 * 40;
        } else if (step <= 0.55 && step > 0.5) {
          newTimestamp += 1000 * 60 * 34;
        } else if (step <= 0.5 && step > 0.45) {
          newTimestamp += 1000 * 60 * 28;
        } else if (step <= 0.45 && step > 0.4) {
          newTimestamp += 1000 * 60 * 22;
        } else if (step <= 0.4 && step > 0.35) {
          newTimestamp += 1000 * 60 * 16;
        } else if (step <= 0.35 && step > 0.3) {
          newTimestamp += 1000 * 60 * 13;
        } else if (step <= 0.3 && step > 0.25) {
          newTimestamp += 1000 * 60 * 10;
        } else if (step <= 0.25 && step > 0.2) {
          newTimestamp += 1000 * 60 * 8;
        } else if (step <= 0.2 && step > 0.15) {
          newTimestamp += 1000 * 60 * 6;
        } else if (step <= 0.15 && step > 0.1) {
          newTimestamp += 1000 * 60 * 4;
        } else if (step <= 0.1 && step > 0.05) {
          newTimestamp += 1000 * 60 * 2;
        } else if (step <= 0.05) {
          newTimestamp += 1000 * 60 * 1;
        } else {
          newTimestamp += 1000 * 60 * 120;
        }
      }
      this.lineList = list;
      this.$nextTick(() => {
        this.location();
      });
    },
    // 根据当前选中时间戳更新时间轴位置
    location() {
      const timestamp = this.getTimestamp();
      let proportion = 0;
      if (this.currentTimestamp > timestamp + 1000) {
        proportion = 1;
        this.currentTime = this.formatTime(timestamp);
        this.currentTimestamp = timestamp;
        this.updateVisible();
        this.emit();
      } else {
        proportion = this.currentTimestamp / timestamp;
      }
      const left =
        this.inWidth * proportion - this.$refs.scaleplateRef.offsetLeft;
      this.left = -left;
    },
    // 鼠标按下
    down(e) {
      this.isDown = true;
      const rect = this.$refs.contentRef.getBoundingClientRect();
      this.downX = e.pageX - rect.left;
      this.downTime = this.currentTime;
    },
    // 鼠标松开
    up() {
      this.isDown = false;
      this.emit(true);
    },
    emit(check) {
      this.$nextTick(() => {
        const start_time = this.time + " " + this.currentTime;
        const date = new Date(
          new Date(start_time).getTime() + 1000 * 60 * 60 * 2
        );
        let year = this.day.getFullYear();
        let month = date.getMonth() + 1;
        let day = date.getDate();
        month = month < 10 ? "0" + month : month + "";
        day = day < 10 ? "0" + day : day + "";
        let hh = date.getHours();
        let mm = date.getMinutes();
        let ss = date.getSeconds();
        hh = hh < 10 ? "0" + hh : hh;
        mm = mm < 10 ? "0" + mm : mm;
        ss = ss < 10 ? "0" + ss : ss;
        const end_time = `${year}-${month}-${day} ${hh}:${mm}:${ss}`;
        if (check && this.downTime == this.currentTime) return;
        this.$emit("change", {
          start_time,
          end_time,
        });
      });
    },
    // 鼠标移动
    move(e) {
      if (!this.isDown) return;
      const rect = this.$refs.main.getBoundingClientRect();
      const scaleplate = this.$refs.scaleplateRef;
      let x = e.pageX - rect.left - this.downX;
      if (x > scaleplate.offsetLeft) {
        x = scaleplate.offsetLeft;
        const rect = this.$refs.contentRef.getBoundingClientRect();
        this.downX = e.pageX - rect.left;
      } else if (Math.abs(x) > this.inWidth - scaleplate.offsetLeft) {
        x = -(this.inWidth - scaleplate.offsetLeft);
        const rect = this.$refs.contentRef.getBoundingClientRect();
        this.downX = e.pageX - rect.left;
      }
      this.left = x;
      this.getCurrentTime(x);
      this.updateVisible();
    },
    // 获取当前标尺选中时间
    getCurrentTime(offset) {
      const offsetX = Math.abs(offset - this.$refs.scaleplateRef.offsetLeft);
      // 计算时间
      const proportion = offsetX / this.$refs.contentRef.offsetWidth;
      const timestamp = this.getTimestamp() * proportion;
      const time = this.formatTime(timestamp);
      this.currentTime = time;
      this.currentTimestamp = timestamp;
    },
    formatTime(timestamp) {
      let hh = parseInt((timestamp / 1000 / 60 / 60) % 24);
      let mm = parseInt((timestamp / 1000 / 60) % 60);
      let ss = parseInt((timestamp / 1000) % 60);
      hh = hh < 10 ? "0" + hh : hh;
      mm = mm < 10 ? "0" + mm : mm;
      ss = ss < 10 ? "0" + ss : ss;
      return `${hh}:${mm}:${ss}`;
    },
    // 滚动滚轮放大缩小时间刻度
    wheel(e) {
      if (e.wheelDelta > 0) {
        // 缩小刻度
        if (this.zoom > 0) this.zoom -= 0.05;
      } else {
        // 放大刻度
        if (this.zoom < 1) this.zoom += 0.05;
      }
      this.updateTimeLine();
      this.updateVisible();
    },
    // update可视刻度
    updateVisible() {
      this.$nextTick(() => {
        this.lineList.forEach((item) => {
          item.show = this.isVisible(item);
        });
      });
    },
  },
};
</script>
<style scope>
.timeline_container {
  position: relative;
  height: 45px;
  background-color: #3a68a3;
  margin-top: 10px;
  overflow: hidden;
  cursor: grab;
  user-select: none;
  -webkit-user-select: none;
}
.timeline_container:active {
  cursor: grabbing;
}
.timeline_container .scaleplate {
  position: absolute;
  width: 2px;
  height: 100%;
  left: 50%;
  top: 0;
  background-color: #fff;
  z-index: 10;
}
.timeline_container .currentTime {
  position: absolute;
  left: 50%;
  top: 0;
  transform: translateX(-50%);
  z-index: 10;
  font-size: 13px;
}

.timeline_container .timeline_content {
  position: relative;
  background-color: #fdd894;
  height: 10px;
  top: 28px;
  margin: 0;
}
.timeline_container .timeline_content .time {
  position: absolute;
  display: flex;
  width: 2px;
  height: 40px;
  flex-direction: column;
  align-items: center;
  margin-top: -15px;
}
.timeline_container .timeline_content .time .line {
  height: 25px;
  width: 2px;
  background-color: rgba(255, 255, 255, 0.5);
}
.timeline_container .timeline_content .time .time_item {
  color: rgba(255, 255, 255, 0.5);
  font-size: 12px;
}
</style>

<template>
  <div @mouseup="allMouseUp">
    <a-row type="flex" justify="space-around">
      <a-col span="11">
        <a-divider>
          <p class="subtitle">工单预览</p>
        </a-divider>
        <div class="preview-box">
          <template v-if="!previewVisible">
            <p class="message-info">请点击上传按钮上传工单</p>
          </template>
          <p class="img-con" @mousedown="onMouseDown" @mouseup="onMounseUp">
            <img :src="previewImage" ref="dragAble" class="dragAble" />
            <canvas
              ref="previewCanvas"
              :height="canvasHeight + 'px'"
              :width="canvasWidth + 'px'"
            ></canvas>
          </p>
        </div>
      </a-col>
      <a-col span="11">
        <a-divider>
          <p class="subtitle">工单识别结果</p>
        </a-divider>
        <div v-if="resultShow === 0" class="result-box">
          <p class="message-info">请选择一张工单查看识别结果</p>
        </div>
        <div v-else-if="resultShow === 1" class="result-box">
          <!-- <div class="spin-root"> -->
          <a-spin class="spinner" size="large">
            <p class="message-info">
              {{ message }}
            </p>
          </a-spin>
          <!-- </div> -->
        </div>
        <div
          v-else-if="resultShow === 2"
          class="result-box"
          v-bind:style="{ minHeight: 600 + 'px' }"
        >
          <ul>
            <li class="table-header"
              v-if="tableName !== ''"
            >
            <strong style="margin:0 auto">{{ tableName }}</strong>

            </li>
            <li

              v-for="(value, keyName) in tableResult"
              :class="{'longresult':resultItemType(value) === 'long','nestresult':resultItemType(value) === 'nest' || resultItemType(value) === 'nestTwice'}"
              v-bind:key="keyName"
              v-on:click="onTextResultClick(keyName)"
            >
              <template v-if="resultItemType(value) === 'normal'">


                <a-row type="flex" justify="space-around" style="width: 100%">
                  <a-col :span="9">
                      <strong>{{ keyName }}</strong>

                  </a-col>
                  <a-col :span="10">
                    {{ value }}
                  </a-col>
                </a-row>
              </template>
              <template v-if="resultItemType(value) === 'long'">
                <strong style="margin:0 auto;margin-bottom:1rem">{{ keyName }}</strong>
                {{ value }}
              </template>

              <template v-if="resultItemType(value) === 'nest'">
                <strong style="margin:0 auto">{{ keyName }}</strong>
                <a-row v-for="item in value" :key="item[0]" type="flex" justify="space-around" style="width: 100%;margin-top:1rem">
                  <a-col :span="9">
                    <strong>{{ item[0] }}</strong>

                  </a-col>
                  <a-col :span="10">
                    {{ item[1] }}
                  </a-col>
                </a-row>
              </template>

              <template v-if="resultItemType(value) === 'nestTwice'">
                <strong style="margin:0 auto">{{ keyName }}</strong>
                <a-row v-for="item in value" :key="item[0]" type="flex" justify="space-around" style="width: 100%;margin-top:1rem">
                  <a-col :span="9">
                    <strong>{{ item[0] }}</strong>

                  </a-col>
                  <a-col :span="10">
                    {{ item[1] }}
                  </a-col>
                </a-row>
              </template>

            </li>
            <li class="table-extra" v-for="(value, keyName) in longResult" :key="keyName" >
                <strong style="margin:0 auto">{{ keyName }}</strong>
                {{ value }}
            </li>

          </ul>
        </div>
      </a-col>
    </a-row>
  </div>
</template>

<script>
import { message } from "ant-design-vue";
// import { delete } from 'vue/types/umd';
export default {
  name: "resultTable",
  props: ["previewImage", "axiosResult"],

  data() {
    return {
      resultShow: 0,
      message: "服务器处理中，请稍后查询",
      previewVisible: false,
      tableResult: {},
      tableName: "",
      longResult:{},    //
      textClickIndex: -1,

      imgWidth: 0,
      imgHeight: 0,

      isdrag: false,
      x: 0,
      y: 0,
      nTX: 0,
      nTY: 0,
      canvasHeight: 300,
      canvasWidth: 300,
      oDragObj: {},

      defaultColor: "#0000FF",
      hightLightColor: "#FF0000",
    };
  },
  watch: {
    /**
     * @description: 父组件负责请求识别结果，当服务器返回结果后，这个属性会改变。
     * @param {*} result 新值
     * @return {*}
     */
    axiosResult: function (result) {
      console.log('-------')
      console.log(result);

      //队列中
      if (result.data.status === 0) {
        this.resultShow = 1;
        this.message = "服务器处理中，请稍后查询";
      } else if (result.data.status === 1) {
        const res = result.data.result;

        if (res.error) {
          message.info("工单识别异常");
          return;
        }
        // 服务器处理完成，展示结果
        this.resultShow = 2;

        if ("dtime" in res) {
          let dtime = res.dtime.toFixed(2); //  toFixed(this.tableResult.dtime);
          message.info("该工单识别用时：" + dtime + " s");
        }
        this.tableResult = JSON.parse(res.table_data);

        console.log(this.tableResult);
        this.extraHandleResult();


      }
    },
  },
  mounted() {},

  methods: {

    chooseAnotherTable() {
      console.log('chooseAnotherTables')
      this.previewVisible = 1;
      this.resultShow = 1;
      this.message = "服务器处理中，请稍后查询";
      this.tableName = "";

    },

    resultItemType(item){
      if(typeof(item) === 'string'){
        // 太长的项目需要换行显示
        if( item.length > 50){
          return 'long';
        }
        return 'normal'
      }
      if(typeof(item) === 'object'){
        if(typeof(item[0][0]) === 'object'){
          return 'nestTwice'
        }
        return 'nest'
      }
    },

      /**
     * @description: 对表格识别结果进行特殊处理
     * @param {*} index
     * @return {*}
     */
    extraHandleResult() {
      // 表名需提取出来置顶显示
      if ("表名" in this.tableResult) {
        this.tableName = this.tableResult["表名"];
        delete this.tableResult["表名"];
        // this.tableResult.
      }

      // for ( let item in this.tableResult){
      //     if(this.tableResult[item].length > 50){
      //         this.longResult[item] = this.tableResult[item];
      //         delete this.tableResult[item]
      //     }
      // }
    },

    /**
     * @description: 监听识别文字点击事件。点击后重新绘制矩形框，且高亮选中文本。
     * @param {*} index
     * @return {*}
     */
    onTextResultClick(index) {
      this.textClickIndex = index;
      this.paintResult();
    },
    /**
     * @description: 在画布上画一个矩形
     * @param {*} ctx 画布
     * @param {*} points 坐标
     * @return {*}
     */
    paintRectangle(ctx, points, lineColor, lineWidth) {
      if (points.length !== 4) {
        console.log("bad rectangle");
        return;
      }

      ctx.strokeStyle = lineColor;
      ctx.lineWidth = lineWidth;
      ctx.moveTo(points[0][0], points[0][1]);
      for (let i = 1; i < 4; i++) {
        ctx.lineTo(points[i][0], points[i][1]);
        ctx.stroke();
      }
      ctx.lineTo(points[0][0], points[0][1]);
      ctx.stroke();
    },
    /**
     * @description: 展示结果，在图片上画矩形
     * @param {*} result
     * @return {*}
     */
    paintResult() {
      // let Ctx = document.getElementsByTagName('canvas')[0].getContext("2d");
      // this.clearCanvas();
      // // 绘制矩形
      // Ctx.beginPath();
      // for(let one of this.tableResult.ocr_data){
      //     this.paintRectangle(Ctx,this.pointConvert(one.text_box_position),this.defaultColor,1);
      // }
      // Ctx.closePath();
      // // 绘制高亮状态
      // if(this.textClickIndex !== -1){
      //     Ctx.beginPath();
      //     this.paintRectangle(Ctx,this.pointConvert(this.tableResult.ocr_data[this.textClickIndex].text_box_position),
      //     this.hightLightColor,3);
      //     Ctx.closePath();
      // }
    },
    /**
     * @description: 文字位置的坐标转换
     * @param {*} origin 四个点的坐标
     * @return {*}
     */
    pointConvert(origin) {
      if (origin.length !== 4) {
        console.log("bad points");
        return origin;
      }

      let res = new Array(4);
      for (let index = 0; index < 4; index++) {
        let oImg = this.$refs.dragAble;
        let offsetWidth = oImg.offsetWidth;
        let offsetHeight = oImg.offsetHeight;
        let newX = (origin[index][0] / this.imgWidth) * offsetWidth;
        let newY = (origin[index][1] / this.imgHeight) * offsetHeight;

        res[index] = [newX, newY];
      }
      return res;
    },
    /**
     * @description: 初始化滚轮放大缩小事件
     * @param {*}
     * @return {*}
     */
    wheelInit() {
      let oImg = this.$refs.dragAble;
      let that = this;
      this.fnWheel(oImg, function (down, oEvent) {
        let oldWidth = oImg.offsetWidth;
        let oldHeight = oImg.offsetHeight;

        let cood = that.getCoord(oImg);

        // 鼠标相对图片左上角的坐标
        let evCoord = {
          top: oEvent.clientY - cood.top,
          left: oEvent.clientX - cood.left,
        };

        let scaleX = evCoord.left / oldWidth; //比例
        let scaleY = evCoord.top / oldHeight;

        if (down) {
          oImg.style.width = oImg.offsetWidth * 0.9 + "px";
          oImg.style.height = oImg.offsetHeight * 0.9 + "px";
        } else {
          oImg.style.width = oImg.offsetWidth * 1.1 + "px";
          oImg.style.height = oImg.offsetHeight * 1.1 + "px";
        }

        let newWidth = oImg.offsetWidth;
        let newHeight = oImg.offsetHeight;
        let oldLeft = oImg.offsetLeft;
        let oldTop = oImg.offsetTop;

        oImg.style.left = oldLeft - scaleX * (newWidth - oldWidth) + "px";
        oImg.style.top = oldTop - scaleY * (newHeight - oldHeight) + "px";

        that.canvasSizeFollow();
        const _this = that;
        that.$nextTick(() => {
          _this.paintResult();
        });
      });
    },
    /**
     * @description: 设置画布大小、位置跟随图片
     * @param {*}
     * @return {*}
     */
    canvasSizeFollow() {
      let oImg = this.$refs.dragAble;
      this.canvasHeight = oImg.offsetHeight;
      this.canvasWidth = oImg.offsetWidth;
      let oCanvas = this.$refs.previewCanvas;

      oCanvas.style.top = oImg.offsetTop + "px";
      oCanvas.style.left = oImg.offsetLeft + "px";
    },
    canvasPosFollow() {
      let oImg = this.$refs.dragAble;
      // this.canvasHeight = oImg.offsetHeight;
      // this.canvasWidth = oImg.offsetWidth;
      let oCanvas = this.$refs.previewCanvas;

      oCanvas.style.top = oImg.offsetTop + "px";
      oCanvas.style.left = oImg.offsetLeft + "px";
    },
    /**
     * @description: 图片初始化，设置图片大小、位置
     * @param {*}
     * @return {*}
     */
    pictureInit() {
      // let that = this;
      return new Promise((resolve) => {
        let img = new Image();
        img.src = this.previewImage;
        let that = this;
        img.onload = function () {
          that.imgWidth = img.width;
          that.imgHeight = img.height;

          that.$refs.dragAble.style.width = img.width + "px";
          that.$refs.dragAble.style.height = img.height + "px";
          that.$refs.dragAble.style.top = "0px";
          that.$refs.dragAble.style.left = "0px";
          resolve();
        };
      });
    },

    clearCanvas() {
      // 清除画布
      let oCanvas = document.getElementsByTagName("canvas")[0];
      if (oCanvas === null) {
        console.log("convas not found");
        return;
      }
      let Ctx = oCanvas.getContext("2d");
      Ctx.clearRect(0, 0, this.canvasWidth, this.canvasHeight);
    },

    /**
     * @description: 点击左侧列表中的图片后会调用这个函数
     * @param {*} uid 图片uid
     * @return {*}
     */
    pictureClicked(uid) {
      console.log("uid=", uid);

      // this.tableResult = null;
      // this.clearCanvas();
      const that = this;

      // 必须下一次tick执行，否则图片不会更新
      this.$nextTick(() => {
        that.pictureInit().then(() => {
          that.wheelInit();
          // that.canvasSizeFollow();
        });
      });
    },
    getScrollTop: function (doc) {
      doc = doc || document;
      return Math.max(doc.documentElement.scrollTop, doc.body.scrollTop);
    },
    getScrollLeft: function (doc) {
      doc = doc || document;
      return Math.max(doc.documentElement.scrollLeft, doc.body.scrollLeft);
    },
    getCoord: function (el) {
      var _t = 0;
      var _l = 0;
      if (document.documentElement.getBoundingClientRect) {
        var box = el.getBoundingClientRect();
        var oDoc = el.ownerDocument;
        if (navigator.userAgent.indexOf("MSIE 6.0") >= 0) {
          _t = box.top - 2 + this.getScrollTop(oDoc);
          _l = box.left - 2 + this.getScrollLeft(oDoc);
        } else {
          _t = box.top + this.getScrollTop(oDoc);
          _l = box.left + this.getScrollLeft(oDoc);
        }
      } else {
        while (el.offsetParent) {
          _t += el.offsetTop;
          _l += el.offsetLeft;
          el = el.offsetParent;
        }
      }
      return { top: _t, left: _l };
    },

    allMouseUp() {
      this.isdrag = false;
    },
    onMounseUp() {
      this.isdrag = false;
    },
    fnWheel(obj, fncc) {
      obj.onmousewheel = fn;
      if (obj.addEventListener) {
        obj.addEventListener("DOMMouseScroll", fn, false);
      }

      function fn(ev) {
        let oEvent = ev;
        // console.log(oEvent)
        let down = true;

        if (oEvent.detail) {
          down = oEvent.detail > 0;
        } else {
          down = oEvent.wheelDelta < 0;
        }
        // console.log(down)

        if (fncc) {
          fncc.call(this, down, oEvent);
        }

        if (oEvent.preventDefault) {
          oEvent.preventDefault();
        }

        return false;
      }
    },
    moveMouse(e) {
      if (this.isdrag) {
        let oImg = this.$refs.dragAble;
        const newTop = this.nTY + e.clientY - this.y;
        const newLeft = this.nTX + e.clientX - this.x;

        oImg.style.top = newTop + "px";
        oImg.style.left = newLeft + "px";

        // const oCanvas = this.$refs.previewCanvas;
        // const oImg = this.$refs.dragAble;
        //
        // oCanvas.style.top = (newTop - oImg.offsetHeight) + "px";
        // oCanvas.style.left = newLeft + "px";

        this.canvasPosFollow();

        // console.log(oImg.style)
        return false;
      }
    },

    onMouseDown(e) {
      // let oDragHandle = e.target;
      // let topElement = "HTML";
      // while (oDragHandle.tagName !== topElement && oDragHandle.className !== "dragAble") {
      //     oDragHandle = oDragHandle.parentElement;
      // }
      // if (oDragHandle.className === "dragAble") {
      let oImg = this.$refs.dragAble;
      this.isdrag = true;
      // this.oDragObj = oDragHandle;
      this.nTY = parseInt(oImg.style.top + 0);
      this.y = e.clientY;
      this.nTX = parseInt(oImg.style.left + 0);
      this.x = e.clientX;
      document.onmousemove = this.moveMouse;
      return false;
      // }
    },
  },
};
</script>

<style scoped>
.spinner {
  width: 100%;
  height: 100%;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
.dragAble {
  position: absolute;
  /*cursor: move;*/
  cursor: grab;
  float: left;
}
canvas {
  float: left;
  position: absolute;
  pointer-events: none;
}
.preview-box,
.result-box {
  box-sizing: content-box;
  border: 2px solid black;
  position: relative;
  min-height: 30rem;
}
.message-info {
  color: black;
  font-size: 20px;
  font-weight: bold;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
.subtitle {
  font-size: 1.5rem;
}

.img-con {
  position: relative;
  min-height: 30rem;
  width: 100%;
  height: 100%;
  overflow: hidden;
  /*border: 1px solid red;*/
}
.table-header{
    font-size: 1.5rem;
    line-height: 1.5rem;

}
ul {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 100%;
  /* min-height: 200px; */
  height: 100%;
  overflow-y: scroll;
  /*text-align: center;*/
  /*display: inline-block;*/
}
li {
  margin: 0.7rem;
  font-size: 1rem;
  line-height: 1rem;
  list-style: none;
  border: 2px solid grey;
  /* text-align: left; */
  display: flex;
  /* justify-content: space-around; */
}
.longresult{
    display: flex;
    flex-direction:column;
    text-align: left;
}
  .nestresult{
    display: flex;
    flex-direction:column;
    text-align: left;
  }
</style>

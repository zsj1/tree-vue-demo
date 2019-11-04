<template>
  <div>
    <!-- 选择json文件按钮 -->
    <el-container>
      <el-header>
        <el-button type="primary" @click="dialogVisible = true">Load from File</el-button>
      </el-header>
    </el-container>
    <!-- 操作echarts图的toolbox，这里是自定义样式和方法完成的，也可使用echarts配置项里的toolbox，就是比较丑 -->
    <div class="toolbox" v-show="toolboxVisible">
      <ul>
        <li @click="treeGraphChangeIsExpand(false)">
          <i class="el-icon-remove-outline"></i>收起
          <br />全部
        </li>
        <li @click="treeGraphChangeIsExpand(true)">
          <i class="el-icon-circle-plus-outline"></i>展开
          <br />全部
        </li>
        <li @click="treeGraphRefresh">
          <i class="el-icon-refresh"></i>重置
        </li>
        <li @click="treeGraphDownload">
          <i class="el-icon-download"></i>下载
        </li>
      </ul>
    </div>
    <el-tabs value="treeGraph" tab-position="left" v-show="tabVisible" @tab-click="handleTabClick">
      <el-tab-pane name="rawData" label="原始数据">
        <el-card class="box-card">
          <div>
            {{uploadFilename}}
            <pre>{{uploadFileContent}}</pre>
          </div>
        </el-card>
      </el-tab-pane>
      <el-tab-pane name="treeGraph" label="树图">
        <el-card class="box-card" :body-style="{ padding: '0px' }">
          <!-- echarts图的containerDiv -->
          <div id="myTreeChart"></div>
        </el-card>
      </el-tab-pane>
    </el-tabs>
    <!-- 选择json文件的对话框，弹框选择，腾出更多空间呈现echarts的图 -->
    <el-dialog title="Load JSON document from file" :visible.sync="dialogVisible">
      <el-upload
        action="#"
        accept=".json"
        ref="upload"
        :on-change="loadJsonFromFile"
        :limit="1"
        :on-exceed="handleExceed"
        :auto-upload="false"
      >
        <el-button size="small" type="primary">点击上传</el-button>
        <div slot="tip" class="el-upload__tip">只能上传json文件</div>
      </el-upload>
      <span slot="footer">
        <el-button type="warning" @click="dialogVisible = false">cancel</el-button>
        <el-button type="success" @click="loadJsonFromFileConfirmed">confirm</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
/* eslint-disable */
import $ from "jquery";
let FileSaver = require("file-saver");
export default {
  name: "HelloWorld",
  data() {
    return {
      uploadFilename: null, // 当前上传文件名
      uploadFileContent: null, // 当前上传文件内容
      uploadFiles: [], // 上传文件list
      dialogVisible: false, // 上传文件对话框是否可见参数
      toolboxVisible: false, // 树图工具栏是否可见参数
      tabVisible: false, // tab切换栏是否可见参数
      treeData: null, // 生成树图的数据
      myTreeChart: null, // 树图变量
      initialTreeDepth: 2, // 树初始展开层数
      option: null // echarts图配置项
    };
  },
  methods: {
    // 标签页点击切换触发函数
    handleTabClick(tab, event) {
      this.toolboxVisible = tab.name === "treeGraph" ? true : false;
    },
    // 处理选择超过一个1个json文件的提示函数
    handleExceed(files, fileList) {
      this.$message.warning(
        `当前限制选择 1 个文件，本次选择了 ${
          files.length
        } 个文件，共选择了 ${files.length + fileList.length} 个文件`
      );
    },
    // 处理上传文件的函数
    loadJsonFromFile(file, fileList) {
      this.uploadFilename = file.name;
      this.uploadFiles = fileList;
    },
    // 提交上传文件函数
    loadJsonFromFileConfirmed() {
      if (this.uploadFiles) {
        // 重置绘图数据
        this.treeData = null;
        // 隐藏tabs标签栏
        this.tabVisible = false;
        // 隐藏工具栏
        this.toolboxVisible = false;
        // 这里因为限制了传入一个json文件，多个文件的话逻辑需要修改
        for (let i = 0; i < this.uploadFiles.length; i++) {
          let file = this.uploadFiles[i];
          if (!file) continue;
          let reader = new FileReader();
          // 读出json文件中的数据
          reader.onload = async e => {
            try {
              let document = JSON.parse(e.target.result);
              this.uploadFileContent = document;
              // 使用递归深度遍历的方法，利用已有json数据，构造出生成树图的json数据
              this.handleJsonContent(document);
              // 利用生成的数据绘图
              this.drawTreeChart();
              // 显示tabs标签栏
              this.tabVisible = true;
              // 显示图工具栏
              this.toolboxVisible = true;
            } catch (err) {
              this.$message.error(
                `Load JSON document from file error: ${err.message}`
              );
            }
          };
          reader.readAsText(file.raw);
        }
      }
      // 隐藏上传文件对话框
      this.dialogVisible = false;
      // 清空upload中的缓存文件
      this.$refs.upload.clearFiles();
    },
    // json文件处理函数
    handleJsonContent(document) {
      let rawTreeData = { root: document };
      // 深度递归遍历生成树图构图所用数据
      this.deepTraversal(rawTreeData, []);
      // 可用如何代码下载生成的json校对
      // let content = JSON.stringify(this.treeData);
      // let blob = new Blob([content], { type: "text/plain;charset=utf-8" });
      // FileSaver.saveAs(blob, "tree.json");
    },
    // obj是当前深度下的子obj, keyList用于维护当前深度下的key
    deepTraversal(obj, keyList) {
      // 遍历当前深度下的全部key
      Object.keys(obj).forEach(key => {
        if (!this.treeData) {
          this.treeData = {
            name: key,
            ...this.getNodeStyle(keyList.length, key, true),
            children: [],
            deep: keyList.length,
            expandStatus: keyList.length < this.initialTreeDepth ? true : false
          };
        } else {
          // 利用指针定位到当前深度位置下的子obj以继续添加数据
          let deepObj = this.treeData;
          keyList.forEach((curKey, index) => {
            if (!(deepObj.constructor === Array)) {
              deepObj = deepObj["children"];
            } else {
              deepObj.forEach((item, index) => {
                if (item["name"] === curKey) {
                  deepObj = deepObj[index]["children"];
                  return;
                }
              });
            }
          });
          // 最后一层节点不需要添加children属性，特殊处理
          if (Object.keys(obj[key]).length === 0) {
            deepObj.push({
              name: key,
              ...this.getNodeStyle(keyList.length, key, false),
              // deep参数是该节点的深度，可能有用，目前未用到
              deep: keyList.length,
              expandStatus:
                keyList.length < this.initialTreeDepth ? true : false
            });
          } else {
            deepObj.push({
              name: key,
              ...this.getNodeStyle(keyList.length, key, true),
              children: [],
              deep: keyList.length,
              expandStatus:
                keyList.length < this.initialTreeDepth ? true : false
            });
          }
        }
        // 这里使用递归，属性类型为对象则进一步遍历
        if (typeof obj[key] === "object") {
          // 如果对象内部是array则需遍历array，进一步深度遍历array中的每个子obj
          if (obj[key].constructor === Array) {
            obj[key].forEach(item => {
              this.deepTraversal(item, [...keyList, key]);
            });
          } else {
            this.deepTraversal(obj[key], [...keyList, key]);
          }
        }
      });
    },
    // 获取树图节点的样式设置部分的obj
    getNodeStyle(deep, key, isHasChildren) {
      // 关于label，树图不支持在传入的data中设置根节点以外的label样式，如需在此配置需要修改源代码
      let color = ["#3d3b4f", "#75878a", "#e9f1f9", "#f2be45"];
      let fontSize = "12px";
      let fontFamily = "Arial";
      let baseStyle = {
        symbolSize: [
          this.getTextSize(fontSize, fontFamily, key).width + 15,
          20
        ],
        symbol: "roundRect",
        itemStyle: {
          borderWidth: 0,
          color: isHasChildren ? (deep <= 1 ? color[deep] : color[2]) : color[3]
        },
        emphasis: {
          itemStyle: {
            borderWidth: 0,
            color: "#2add9c"
          }
        }
      };
      return baseStyle;
    },
    // 获取树图节点的name（名称）字段在实际页面中的width和height，注意这里和字体大小以及字体样式有关
    getTextSize(fontSize, fontFamily, text) {
      let span = document.createElement("span");
      let result = {};
      result.width = span.offsetWidth;
      result.height = span.offsetHeight;
      span.style.visibility = "hidden";
      span.style.fontSize = fontSize;
      span.style.fontFamily = fontFamily;
      span.style.display = "inline-block";
      document.body.appendChild(span);
      if (typeof span.textContent != "undefined") {
        span.textContent = text + "空格";
      } else {
        span.innerText = text;
      }
      result.width = Math.ceil(
        parseFloat(window.getComputedStyle(span).width) - result.width
      );
      result.height = Math.ceil(
        parseFloat(window.getComputedStyle(span).height) - result.height
      );
      document.body.removeChild(span);
      return result;
    },
    // 设置树图所在的containerDiv的宽高，以实现无滚动条当前页面自适应
    resizeGraphContainer() {
      let $myTreeChart = $("#myTreeChart");
      let bodyMargin = parseInt($("body").css("margin")) * 2;
      let headerHeight = parseInt($(".el-header").css("height"));
      $myTreeChart.css(
        "height",
        window.innerHeight - headerHeight - bodyMargin - 5
      );
      $myTreeChart.css("width", window.innerWidth - bodyMargin - 96);
    },
    // 树图刷新函数
    treeGraphRefresh() {
      this.drawTreeChart();
    },
    // 树图下载函数
    treeGraphDownload() {
      this.myTreeChart.resize();
      setTimeout(function() {
        let myCanvas = $("#myTreeChart").find("canvas")[0];
        let image = myCanvas.toDataURL("image/jpeg");
        let $a = document.createElement("a");
        $a.setAttribute("href", image);
        $a.setAttribute("download", "echarts图片");
        $a.click();
      }, 1000);
    },
    // 树图全部节点收起或者展开函数
    treeGraphChangeIsExpand(status) {
      // 注意这里两个属性一个是内置一个是外部添加的,需要对不同的数据进行修改
      this.myTreeChart._chartsViews[0]._data.tree._nodes.forEach(
        (item, index) => {
          item.isExpand = status;
        }
      );
      this.myTreeChart._chartsViews[0]._data._rawData._data.forEach(
        (item, index) => {
          item.expandStatus = status;
        }
      );
      // 使用resize触发刷新
      this.myTreeChart.resize();
    },
    // 绘制树图的主函数
    drawTreeChart() {
      // 解决this指针冲突
      let self = this;
      // 防止出现“There is a myChart instance already initialized on the dom.”的警告
      if (
        this.myTreeChart !== null &&
        this.myTreeChart !== "" &&
        this.myTreeChart !== undefined
      ) {
        this.myTreeChart.dispose();
      }
      // 初始化echarts实例容器div的宽高
      this.resizeGraphContainer();
      // 基于准备好的dom，初始化echarts实例
      this.myTreeChart = this.$echarts.init(
        document.getElementById("myTreeChart")
      );
      // 屏幕大小自适应，重置容器高宽
      $(window).on("resize", function() {
        self.resizeGraphContainer();
        self.myTreeChart.resize();
      });
      // 绘制过程加载loading
      this.myTreeChart.showLoading();
      // 生成配置项参数
      this.option = {
        backgroundColor: "#fff", // 背景色
        tooltip: {
          trigger: "item",
          triggerOn: "mousemove"
        },
        series: [
          {
            type: "tree", // 绘制图的类型
            roam: true, // 是否开启缩放和拖动，这里false是因为无法解决canvas在下载的时候只下载当前屏幕内容，若开启则可能导致下载图不完整，官方推荐策略也是不开启，目前官方文件未给出较好的缩放和拖拽外部api
            data: [this.treeData], // 绘图所用的数据
            // 距离上下左右容器的距离，default 12%过大，这里缩小下
            top: "3%",
            left: "3%",
            bottom: "3%",
            right: "150px", // 为使得工具栏不重叠,改为固定px
            // 这里设置节点内文字样式，原本可在getNodeStyle配置化设置，但是除跟节点之外，在data中直接配置都是失效的，原因未查明
            label: {
              normal: {
                position: "inside",
                verticalAlign: "middle",
                align: "center",
                formatter: function(param) {
                  let expandStatusIcon = "";
                  let name = "";
                  if (param.data.deep > 1 || !param.data.children) {
                    name = "{black|" + param.name + "  }";
                  } else {
                    name = "{white|" + param.name + "  }";
                  }
                  if (param.data.children) {
                    if (param.data.expandStatus) {
                      expandStatusIcon = "{icon|-}";
                    } else {
                      expandStatusIcon = "{icon|+}";
                    }
                  }
                  return name + expandStatusIcon;
                },
                // 富文本样式
                rich: {
                  black: {
                    color: "#000000",
                    fontFamily: "Arial",
                    fontSize: 12
                  },
                  white: {
                    color: "#ffffff",
                    fontFamily: "Arial",
                    fontSize: 12
                  },
                  icon: {
                    width: 14,
                    height: 14,
                    color: "#ffffff",
                    fontFamily: "Arial",
                    fontSize: 16,
                    backgroundColor: "#409eff",
                    borderColor: "#003472",
                    borderRadius: 7,
                    // lineHeight: 14,
                    borderWidth: 1
                  }
                }
              }
            },
            initialTreeDepth: this.initialTreeDepth, // 树图初始深度
            expandAndCollapse: true, // 子树折叠和展开的交互
            animationDuration: 550, // 初始动画时间
            animationDurationUpdate: 750 // 图改变时重绘动画时间
          }
        ]
      };
      this.myTreeChart.clear();
      this.myTreeChart.setOption(this.option, true);
      // setTimeout(this.myTreeChart.setOption(option, true), 500);
      this.myTreeChart.hideLoading();
      // 修改expandStatus
      this.myTreeChart.on("click", e => {
        const name = e.data.name;
        const curNode = this.myTreeChart._chartsViews[0]._data.tree._nodes.find(
          item => {
            return item.name === name;
          }
        );
        this.myTreeChart._chartsViews[0]._data._rawData._data.forEach(
          (item, index) => {
            if (item.name === curNode.name && item.deep === (curNode.depth - 1)) {
              item.expandStatus = !item.expandStatus;
            }
          }
        );
        // 使用resize触发刷新
        this.myTreeChart.resize();
      });
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.el-header {
  line-height: 60px;
}

.toolbox ul {
  padding: 0;
  right: 20px;
  bottom: 80px;
  background-color: #ffffff;
  box-shadow: 0 0 0.5em rgba(0, 0, 0, 0.35);
  list-style-type: none;
  position: absolute;
  z-index: 20;
}

.toolbox ul li {
  width: 45px;
  text-align: center;
  font-size: 12px;
  color: #666;
  cursor: pointer;
  padding: 6px 0;
  border-bottom: 1px solid #c0ccd0;
}

.toolbox ul li:hover {
  background-color: #128bed;
  color: #ffffff;
}

.toolbox ul li i {
  font-size: 20px;
  display: block;
  margin-bottom: 5px;
}

#myTreeChart {
  /* border: 1px solid #EBEEF5;
  background-color: #FFF;
  color: #303133; */
}
</style>

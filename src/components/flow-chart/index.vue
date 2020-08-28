<!--
流程图展示组件
@Author: linqian
@Date: 2020/06/22
-->
<template>
  <div>
    <div id="myDiagramDiv" ref="myDiagramDiv" style="width: 100%; height: 100vh; overflow: auto;"></div>
    <!-- 详情弹出框 -->
    <div class="shallow-box" v-if="isShow" :style="modalStyle" ref="shallowBox">
      <span class="close" @click="closeModal">X</span>
      <div class="header">
        <span class="title">{{detailBindData.name}}</span>
        <span
          id="time"
          class="time"
          v-if="detailBindData.showCountDown"
        >{{detailBindData.countTime}}</span>
      </div>
      <div class="form">
        <div class="form-item" v-if="detailBindData.actStartTime">
          <span class="label">转入时间</span>
          <span class="text">{{detailBindData.actStartTime}}</span>
        </div>
        <div class="form-item" v-if="detailBindData.actEndTime">
          <span class="label">转出时间</span>
          <span class="text">{{detailBindData.actEndTime}}</span>
        </div>
        <div class="form-item" v-if="detailBindData.assigneeList.length > 0">
          <span class="label">处理人</span>
          <span class="text">{{detailBindData.assigneeList.join(" ")}}</span>
        </div>
        <div class="form-item" v-if="detailBindData.statusStr">
          <span class="label">状态</span>
          <span class="text">{{detailBindData.statusStr}}</span>
        </div>
        <div class="form-item" v-if="detailBindData.dueDateNum">
          <span class="label">办理期限</span>
          <span class="text">{{detailBindData.dueDateNum}}H</span>
        </div>
        <div class="form-item" v-if="detailBindData.beforeDueLimit">
          <span class="label">催办期限</span>
          <span class="text">{{detailBindData.beforeDueLimit}}H</span>
        </div>
        <div class="form-item" v-if="detailBindData.description">
          <span class="label">任务描述</span>
          <span class="text">{{detailBindData.description}}</span>
        </div>
        <div class="form-item" style="color: red;" v-if="detailBindData.tipMsgStr">
          <span id="tipMsgSpan">{{detailBindData.tipMsgStr}}</span>
        </div>
        <div class="form-item" v-if="detailBindData.approveMessage">
          <span class="label">审批信息</span>
          <span class="text">{{detailBindData.approveMessage}}</span>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
window.go = require("./js/go.js");
// 绘图模拟数据
import modalData from "./js/data.js";
// 颜色变量
const lightGray = "rgb(233,233,233)";
const lightBlue = "rgb(227,239,255)";
const blue = "rgb(64,144,255)";
const green = "rgb(52,159,29)";

const $ = window.go.GraphObject.make;

export default {
  props: {
    modelData: {
      type: Object,
      default: () => {
        return modalData;
      }
    },
    // 详情框扩展字段
    otherData: String
  },
  data() {
    return {
      taskDetail: {},
      isShow: false,
      textColor: {
        ready: "gray",
        complete: blue,
        ing: "white"
      },
      borderColor: {
        ready: "gray",
        complete: blue,
        ing: green
      },
      myDiagram: null,
      modalStyle: {},
      timer: null,
      // 详情框绑定的数据
      detailBindData: {}
    };
  },
  computed: {
    // 已生成的任务
    taskList() {
      return this.modelData.data;
    },
    // 所有节点数据
    allNodeList() {
      return JSON.parse(this.modelData.json).dataModel;
    }
  },
  methods: {
    // 关闭弹框
    closeModal() {
      this.isShow = false;
      clearTimeout(this.timer);
    },

    getSubContainerDrawData(id) {
      for (const key in this.allNodeList) {
        const element = this.allNodeList[key];
        if (key === id) {
          return element;
        }
      }
    },

    // 返回当前节点(非子容器)的状态
    // 三种状态: 未开始、进行中、 已完成
    getTaskStatus(id) {
      const item = this.taskList.find(el => el.id === id);
      if (item === undefined) {
        return "ready";
      } else {
        return item.activity === false ? "complete" : "ing";
      }
    },

    // 返回子容器状态，根据子容器里面的节点状态判断,所有节点都完成 - 完成， 部分节点进行中 - 进行中 ， 节点都没有开始 - 未开始
    getSubContainerStatus(id) {
      const node = this.getSubContainerDrawData(id);
      if (!node) {
        return "ready";
      }
      const { nodeDataArray } = node.diagramModel;
      const groupItems = nodeDataArray.filter(el => el.category !== "event");
      // 交集
      let attr = [...groupItems].filter(x =>
        [...this.taskList].some(y => y.id === x.id)
      );
      if (groupItems.length === 0 || attr.length === 0) {
        return "ready";
      } else if (attr.length < groupItems.length) {
        return "ing";
      } else {
        return "complete";
      }
    },
    // 添加子容器里面的节点数据
    addGroupItems(data, myDiagram) {
      myDiagram.startTransaction("addGroupContents");
      const subKey = data.key;
      const id = data.id;
      const { nodeDataArray } = JSON.parse(this.modelData.json); // 主图的绘图数据
      const node = this.getSubContainerDrawData(id);
      // 子容器的绘图数据
      let subNodeDataArray = node.diagramModel.nodeDataArray;
      let subLinkDataArray = node.diagramModel.linkDataArray;
      // const isEnd =
      //   subNodeDataArray.find(
      //     el => el.category === "event" && el.type === "end"
      //   ) === undefined
      //     ? false
      //     : true;
      const isEnd =
        subNodeDataArray.find(
          el => el.category === "event" && el.text === "终止"
        ) === undefined
          ? false
          : true;

      // 数据模型中需要添加子节点集合， 去除开始、结束/终止节点
      let nodeAttr = subNodeDataArray.filter(el => el.category !== "event");

      // 注意:  子容器里的节点key不唯一，所以给key添加一个时间戳 格式为${key}_ ${time}
      let time = new Date().getTime();
      nodeAttr.map(item => {
        item.group = subKey; // 组成员标识
        item.key = item.key + "_" + time;
      });

      // 修改连线集合中对应的节点key
      subLinkDataArray.map(item => {
        item.to = item.to + "_" + time;
        item.from = item.from + "_" + time;
      });

      // 将节点添加到数据模型中
      for (let i = 0; i < nodeAttr.length; i++) {
        let nodeItem = nodeAttr[i];
        myDiagram.model.addNodeData(nodeItem);
      }

      // 获取当前子容器节点
      let subNode = myDiagram.findNodeForKey(subKey);

      // 进入子容器线集合
      let linkIntoAttr = [];

      // 从子容器出来线集合
      let linkOutOfAttr = [];

      // 获取进入节点的线
      subNode.findLinksInto().each(item => {
        linkIntoAttr.push(item.data);
      });
      // 获取从节点出来的线
      subNode.findLinksOutOf().each(item => {
        linkOutOfAttr.push(item.data);
      });

      // 删除子组件本身的连线
      let removeLinks = [];
      subNode.findLinksConnected().each(function(link) {
        removeLinks.push(link.data);
      });
      myDiagram.model.removeLinkDataCollection(removeLinks);

      const subStartLink = subLinkDataArray.find(el => el.sourceRef === "pre");

      for (let i = 0; i < linkIntoAttr.length; i++) {
        let linkItem = linkIntoAttr[i];
        linkItem.to = subStartLink.to;
      }

      // 新增流入的连线
      subLinkDataArray = [...subLinkDataArray, ...linkIntoAttr];

      const subEndLink = subLinkDataArray.find(
        el => el.targetRef === "next" || el.targetRef === "end"
      );

      const subEndIndex = subLinkDataArray.findIndex(
        el => el.targetRef === "next" || el.targetRef === "end"
      );

      // 判断该流程是否是终止状态，如果是，则最终指向主流程的结束， 如果不是，则最终指向子容器的下一节点
      if (isEnd) {
        subLinkDataArray[subEndIndex].to = nodeDataArray.filter(
          el => el.type === "end"
        ).key;
      } else {
        for (let i = 0; i < linkOutOfAttr.length; i++) {
          let linkItem = linkOutOfAttr[i];
          linkItem.from = subEndLink.from;
        }
        subLinkDataArray = [...subLinkDataArray, ...linkOutOfAttr];
      }

      subLinkDataArray = subLinkDataArray.filter(
        item =>
          item.sourceRef !== "pre" &&
          item.targetRef !== "next" &&
          item.targetRef !== "end"
      );

      console.log("subNodeDataArray", subNodeDataArray);
      console.log("subLinkDataArray", subLinkDataArray);

      for (let i = 0; i < subLinkDataArray.length; i++) {
        const linkItem = subLinkDataArray[i];
        myDiagram.model.addLinkData(linkItem);
      }

      myDiagram.commitTransaction("addGroupContents");

      // 当前画布中的绘图数据
      const aa = JSON.parse(myDiagram.model.toJson());
      console.log(aa.nodeDataArray.length, aa.linkDataArray.length);
    },

    // 获取连线颜色 -
    // 普通节点： 根据sourceRef和targetRef分为去匹配已经生成的任务，如果有两个id中有任意一个不存在于taskList中，则说明是未走过流程
    // 子容器节点： 根据子节点
    getLinkColor(data) {
      const { sourceRef, targetRef } = data;
      let fromNodeStatus = false;
      let toNodeStatus = false;
      // 判断节点是普通节点还是子容器节点
      const isSubNodeFrom = sourceRef.startsWith("subContainer");
      const isSubNodeTo = targetRef.startsWith("subContainer");

      if (isSubNodeFrom) {
        fromNodeStatus = this.getSubContainerStatus(sourceRef);
      } else {
        fromNodeStatus = this.taskList.find(el => el.id === sourceRef);
      }

      if (isSubNodeTo) {
        toNodeStatus = this.getSubContainerStatus(targetRef);
      } else {
        toNodeStatus = this.taskList.find(el => el.id === targetRef);
      }
      if (
        fromNodeStatus !== "ready" &&
        fromNodeStatus !== undefined &&
        toNodeStatus !== "ready" &&
        toNodeStatus !== undefined
      ) {
        return blue;
      } else {
        return "gray";
      }
    },

    // 返回节点背景图模板
    getTaskPictureTemplate() {
      const { getSubContainerStatus, getTaskStatus } = this;
      return $(
        go.Picture,
        { name: "pictureTemplate" },
        new go.Binding("width", "category", function(category) {
          return category === "gateway" ? 50 : 180;
        }),
        new go.Binding("height", "category", function(category) {
          return category === "gateway" ? 50 : 34;
        }),
        new go.Binding("source", function(e) {
          const status =
            e.category === "subprocess"
              ? getSubContainerStatus(e.id)
              : getTaskStatus(e.id);

          return require("./images/" + e.type + "_" + status + ".png");
        })
      );
    },

    // 获取详情框数据
    getlDetail(e, o) {
      // 模拟接口返回的详情数据
      const obj = {
        statusStr: "进行中",
        actStartTime: "2020-06-23 18:01",
        actEndTime: null,
        dueDate: "2020-06-24 18:01:11",
        dueDateNum: "24",
        name: "节点2-同意",
        approveMessage: "不同意,信息项不全",
        assigneeList: [],
        description: "任务描述",
        beforeDueLimit: "10"
      };

      this.renderModal(obj);
      this.isShow = true;

      this.$nextTick(() => {
        // 弹框展示
        let left = e.Dr.offsetX;
        let top = e.Dr.offsetY + 30;
        
        const diagramDom = this.$refs['myDiagramDiv'];
        const dom = this.$refs['shallowBox'];

        // 防止弹框超出画布范围
        if (left + dom.clientWidth > diagramDom.clientWidth) {
          left = left - dom.clientWidth;
        }
        if (top + dom.clientHeight > diagramDom.clientHeight) {
          top = top - 50 - dom.clientHeight;
        }
        // 设置弹出框的位置
        this.modalStyle = {
          left: left + "px",
          top: top + "px"
        };
      });
    },
    // 渲染弹出框信息
    renderModal(obj) {
      this.detailBindData = obj;
      const { actStartTime, dueDate, beforeDueLimit } = obj;
      let tipMsg = [];
      let currentTime = new Date().getTime();

      // 倒计时计算 - 前提：流入时间/办理期限不为空
      if (actStartTime) {
        if (dueDate) {
          // 任务截止时间 - 当前时间 ---> > 0 计算倒计时 ， <= 0 提示节点超时
          let differTime = new Date(dueDate).getTime() - currentTime; // 任务截止与流入时间差值
          if (differTime > 0) {
            // 倒计时
            let h = Math.floor(differTime / 1000 / 60 / 60);
            let m = Math.floor((differTime / 1000 / 60) % 60);
            let s = Math.floor((differTime / 1000) % 60);
            h = h < 10 ? "0" + h : h;
            m = m < 10 ? "0" + m : m;
            s = s < 10 ? "0" + s : s;
            this.detailBindData = {
              ...this.detailBindData,
              showCountDown: true,
              countTime: h + ":" + m + ":" + s
            };
            this.timer = setTimeout(() => {
              this.renderModal(obj);
            }, 1000);
          } else {
            // 系统超时
            tipMsg.push("节点超时");
          }
        }

        // 系统催办时间 - 当前时间 ---> > 0 不催办， <= 0 提示系统催办
        if (beforeDueLimit) {
          let differPress =
            new Date(actStartTime).getTime() +
            Number(beforeDueLimit) * 60 * 60 * 1000 -
            currentTime; // 催办时间与流入时间差值
          if (differPress <= 0) {
            tipMsg.push("系统催办");
          }
        }
      }

      this.detailBindData = {
        ...this.detailBindData,
        tipMsgStr: tipMsg.length > 0 ? tipMsg.join("/") : ""
      };

      console.log("详情数据", this.detailBindData);
    }
  },
  mounted() {
    const {
      getTaskStatus,
      getSubContainerStatus,
      textColor,
      borderColor,
      getLinkColor,
      addGroupItems,
      getTaskPictureTemplate,
      getlDetail,
      closeModal,
      getSubContainerDrawData
    } = this;
    let { myDiagram, isShow } = this;
    myDiagram = $(go.Diagram, "myDiagramDiv", {
      "toolManager.mouseWheelBehavior": go.ToolManager.WheelZoom,
      initialContentAlignment: go.Spot.Left,
      allowCopy: false,
      allowDelete: false,
      layout: $(go.TreeLayout, {
        angle: 90,
        // layerSpacing: 0, // 节点间垂直距离
        nodeSpacing: 120 // 横向节点的间距宽度
      })
      // "grid.visible": true, // 是否展示网格
    });

    // 返回文字节点模板
    function getTextTemplate() {
      return $(
        go.TextBlock,
        {
          font: "10pt Lato, Helvetica, Arial, sans-serif",
          overflow: go.TextBlock.OverflowEllipsis,
          maxLines: 1,
          maxSize: new go.Size(120, NaN),
          margin: new go.Margin(0, 0, 0, 15),
          wrap: go.TextBlock.WrapFit,
          name: "textTemplate"
        },
        new go.Binding("stroke", function(e) {
          const status =
            e.category === "subprocess"
              ? getSubContainerStatus(e.id)
              : getTaskStatus(e.id);
          return textColor[status];
        }),
        new go.Binding("text", "text", function(text) {
          // 只截取7个字符长度，超过部分用'...'表示
          if (text.length > 7) {
            return text.substr(0, 7) + "...";
          } else {
            return text;
          }
        })
      );
    }

    // 返回悬浮提示框节点模板
    function getToolTipTemplate() {
      return $(
        "ToolTip",
        $(go.TextBlock, { margin: 3 }, new go.Binding("text"))
      );
    }

    // 定义事件-开始/结束模板
    // 结束节点 -> 只有“未开始/已完成” 状态, 开始节点 -> 只有“已完成”状态
    const eventNodeTemplate = $(
      go.Node,
      "Table",
      $(
        go.Panel,
        "Spot",
        { desiredSize: new go.Size(75, 75) },
        new go.Binding("location", "loc", go.Point.parse).makeTwoWay(
          go.Point.stringify
        ),
        $(
          go.Shape,
          "Circle",
          {
            stroke: null,
            portId: "",
            fromLinkable: true,
            fromLinkableSelfNode: true,
            fromLinkableDuplicates: true,
            toLinkable: true,
            toLinkableSelfNode: true,
            toLinkableDuplicates: true,
            cursor: "pointer"
          },
          new go.Binding("fill", function(e) {
            return e.type == "start" ? lightBlue : lightGray;
          })
        ),
        $(
          go.Shape,
          "Circle",
          {
            desiredSize: new go.Size(55, 55),
            strokeWidth: 2,
            stroke: null
          },
          new go.Binding("fill", function(e) {
            if (e.type == "start") {
              return blue;
            } else {
              return getTaskStatus(e) === "ready" ? "white" : blue;
            }
          })
        ),
        $(
          go.TextBlock,
          {
            font: "bold 14pt helvetica, bold arial, sans-serif"
          },
          new go.Binding("stroke", function(e) {
            if (e.type == "start") {
              return "white";
            } else {
              return getTaskStatus(e) === "ready" ? "gray" : "white";
            }
          }),
          new go.Binding("text")
        )
      )
    );

    // 定义节点默认模板
    const defaultNodeTemplate = $(
      go.Node,
      "Auto",
      getTaskPictureTemplate(),
      getTextTemplate(),
      {
        toolTip: getToolTipTemplate(),
        click: function(e, o) {
          return getlDetail(e, o);
        },
        selectionChanged: function(e) {
          if (!e.isSelected) {
            closeModal();
          }
        }
      }
    );

    // 定义网关节点模板
    const gatewayNodeTemplate = $(go.Node, "Auto", getTaskPictureTemplate());

    // 定义子容器分组
    const defaultGroupTemplate = $(
      go.Group,
      "Auto",
      {
        layout: $(go.TreeLayout, {
          angle: 90,
          arrangement: go.TreeLayout.ArrangementHorizontal,
          isRealtime: false
        }),
        isSubGraphExpanded: false,
        subGraphExpandedChanged: function(group) {
          // 动态修改节点属性值
          let box = group.findObject("box");
          let pictureTemplate = group.findObject("pictureTemplate");
          let textTemplate = group.findObject("textTemplate");
          box.visible = group.isSubGraphExpanded;
          pictureTemplate.visible = !box.visible;
          textTemplate.visible = !box.visible;
          if (group.memberParts.count === 0) {
            addGroupItems(group.data, myDiagram);
          }
        }
      },
      $(
        go.Shape,
        "Rectangle",
        {
          name: "box",
          fill: null,
          visible: false,
          strokeDashArray: [10, 10] // 设置虚线
        },
        new go.Binding("stroke", "id", function(id) {
          let status = getSubContainerStatus(id);
          return borderColor[status];
        })
      ),
      $(
        go.Panel,
        "Table",
        { margin: new go.Margin(5, 0, 0, 0) },
        getTaskPictureTemplate(),
        getTextTemplate(),
        $(
          "SubGraphExpanderButton",
          {
            alignment: go.Spot.Right,
            margin: new go.Margin(0, 5, 0, 5),
            width: 18,
            height: 18
          },
          new go.Binding("visible", "id", function(id) {
            let node = getSubContainerDrawData(id);
            if (node) {
              // 子容器的绘图数据
              let subNodeDataArray = node.diagramModel.nodeDataArray;
              let subLinkDataArray = node.diagramModel.linkDataArray;

              // 节点或者连线，两者有一个没有数据，则默认不绘图
              if (
                subNodeDataArray.length === 0 ||
                subLinkDataArray.length === 0 ||
                subNodeDataArray.length === 2 ||
                subLinkDataArray.length < subNodeDataArray.length - 1
              ) {
                return false;
              } else {
                return true;
              }
            } else {
              return false;
            }
          })
        ),
        $(go.Placeholder, {
          row: 1,
          columnSpan: 2,
          padding: 10,
          alignment: go.Spot.TopLeft
        })
      ),

      {
        toolTip: getToolTipTemplate()
      }
    );

    /*******************  节点模板引用    ***************************/
    myDiagram.nodeTemplateMap.add("", defaultNodeTemplate);
    myDiagram.nodeTemplateMap.add("event", eventNodeTemplate);
    myDiagram.nodeTemplateMap.add("gateway", gatewayNodeTemplate);

    /*******************  分组模板引用    ***************************/
    myDiagram.groupTemplateMap.add("subprocess", defaultGroupTemplate);

    /*******************  连线模板    ***************************/

    myDiagram.linkTemplate = $(
      go.Link, // the whole link panel
      {
        routing: go.Link.AvoidsNodes
        // curve: go.Link.JumpOver,
        // corner: 5,
      },
      $(
        go.Shape, // the link path shape
        { isPanelMain: true, strokeWidth: 1 },
        new go.Binding("stroke", function(e) {
          let color = getLinkColor(e);
          return color;
        })
        // new go.Binding("stroke", "isSelected", function (sel) {
        //   return sel ? "dodgerblue" : "gray";
        // }).ofObject()
      ),
      // 箭头
      $(
        go.Shape,
        { toArrow: "standard", strokeWidth: 0 },
        new go.Binding("fill", function(e) {
          return getLinkColor(e);
        })
      ),
      $(
        go.Panel,
        "Auto",
        $(
          go.TextBlock,
          {
            textAlign: "center",
            font: "9pt helvetica, arial, sans-serif",
            margin: 4
          },
          new go.Binding("text")
        )
      )
    );

    // 鼠标滚轮滚动，隐藏弹框
    const tool = myDiagram.toolManager;
    tool.standardMouseWheel = function() {
      if (!isShow) {
        isShow = true;
      }
      go.ToolManager.prototype.standardMouseWheel.call(tool);
    };

    // 加载数据
    myDiagram.model = go.Model.fromJson(this.modelData.json);
  }
};
</script>

<style lang="scss">
.shallow-box {
  position: absolute;
  top: 300px;
  left: 100px;
  padding: 12px;
  width: 400px;
  border: 1px solid lightgray;
  background: white;
  z-index: 10000;
  .close {
    position: absolute;
    right: 10px;
    top: 5px;
    cursor: pointer;
  }
  .header {
    display: flex;
    align-items: center;
    border-bottom: 1px solid lightgray;
    padding-bottom: 12px;
    .title {
      width: 270px;
      display: inline-block;
      // white-space: nowrap;
      // overflow: hidden;
      // text-overflow: ellipsis;
    }
    .time {
      margin-left: 30px;
      background: rgb(227, 239, 255);
      height: 24px;
      padding: 0 5px;
    }
  }
  .form-item {
    display: flex;
    margin: 10px 0;
    .label {
      width: 80px;
      color: gray;
      display: inline-block;
    }
    .text {
      width: calc(100% - 80px);
    }
  }
}
</style>


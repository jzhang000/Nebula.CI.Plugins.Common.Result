<template>
  <a-layout
    id="components-layout-demo-custom-trigger"
    style="height: 100vh; position: fixed; width: 100%"
  >
    <a-layout-sider
      width="250px"
      :style="{
        overflow: 'hidden',
        width: '250px',
        height: '100vh',
        position: 'fixed',
        left: 0,
        padding: '1rem 0 2.5rem 0',
        background: '#fff',
      }"
    >
      <div class="search">
        <a-input-search
          v-model="searchStr"
          placeholder="输入文件名"
          style="width: 100%; padding: 5px"
          @search="onSearch"
          @change="onSearch"
        ></a-input-search>
      </div>
      <div class="treeDiv" @click="clearMenu">
        <a-tree
          :treeData="treeData"
          showIcon
          defaultExpandAll
          :selectedKeys="selectedKeys"
          :expanded-keys="expandedKeys"
          :replaceFields="replaceFields"
          @select="onSelect"
          @expand="onExpand"
          @rightClick="onRightClick"
        >
          <img
            src="../assets/folder.png"
            alt=""
            slot="folder"
            style="padding-bottom: 4px"
          />
          <img
            src="../assets/file.png"
            alt=""
            slot="file"
            style="padding-bottom: 4px"
          />
          <img
            src="../assets/c.png"
            alt=""
            slot="c"
            style="padding-bottom: 4px"
          />
          <img
            src="../assets/css.png"
            alt=""
            slot="css"
            style="padding-bottom: 4px"
          />
          <img
            src="../assets/EXCEL.png"
            alt=""
            slot="excel"
            style="padding-bottom: 4px"
          />
          <img
            src="../assets/word.png"
            alt=""
            slot="word"
            style="padding-bottom: 4px"
          />
          <img
            src="../assets/js.png"
            alt=""
            slot="js"
            style="padding-bottom: 4px"
          />
          <img
            src="../assets/html.png"
            alt=""
            slot="html"
            style="padding-bottom: 4px"
          />
          <img
            src="../assets/json.png"
            alt=""
            slot="json"
            style="padding-bottom: 4px"
          />
          <img
            src="../assets/XML.png"
            alt=""
            slot="xml"
            style="padding-bottom: 4px"
          />

          <template slot="title" slot-scope="{ name }">
            <span
              v-html="
                name.replace(
                  new RegExp(searchValue, 'g'),
                  '<span style=color:#f50>' + searchValue + '</span>'
                )
              "
            ></span>
          </template>
        </a-tree>
        <div :style="this.tmpStyle" v-if="this.NodeTreeItem">
          <div class="menu-item" @click="downloadFile">
            <a-tooltip placement="bottom" title="下载文件">
              <a-icon type="download" />
            </a-tooltip>
          </div>
        </div>
      </div>
    </a-layout-sider>
    <a-layout>
      <a-layout-content
        :style="{ margin: '4px 2px 0px 252px', background: '#fff' }"
      >
        <iframe
          :src="src"
          id="propertyIframe"
          ref="iframe"
          width="100%"
          height="100%"
          frameborder="no"
          border="0"
          scrolling="auto"
        ></iframe>
      </a-layout-content>
    </a-layout>
  </a-layout>
</template>
<script>
import axios from "axios";

const baseUrl = "http://172.18.67.167:32100";
const catalogPath = "/api/ci/plugins/common/result/catalog/";

export default {
  data() {
    return {
      baseUrl,
      catalogPath,
      replaceFields: { title: "name" },
      selectedKeys: [],
      searchValue: "",
      expandedKeys: [],
      treeData: [],
      searchStr: "",
      treeStyle: {
        height: "",
      },
      src: "",
      NodeTreeItem: null, // 右键菜单
      tmpStyle: "",
    };
  },
  mounted() {
    let params = window.location.search;
    let pipelineHistoryId = params.split("=")[1];
    this.getCatalog("1");
  },
  methods: {
    onSearch() {
      var vm = this;
      //添加这行代码是为了只点击搜索才触发
      vm.searchValue = vm.searchStr;
      //如果搜索值为空，则不展开任何项，expandedKeys置为空
      //如果搜索值不位空，则expandedKeys的值要为搜索值的父亲及其所有祖先
      if (vm.searchValue === "") {
        vm.expandedKeys = [];
      } else {
        //首先将展开项与展开项副本置为空
        vm.expandedKeys = [];
        vm.backupsExpandedKeys = [];
        //获取所有存在searchValue的节点
        let candidateKeysList = vm.getkeyList(vm.searchValue, vm.treeData, []);
        //遍历满足条件的所有节点
        candidateKeysList.map((item) => {
          //获取每个节点的母亲节点
          var key = vm.getParentKey(item, vm.treeData);
          //当item是最高一级，父key为undefined，将不放入到数组中
          //如果母亲已存在于数组中，也不放入到数组中
          if (key && !vm.backupsExpandedKeys.some((item) => item === key))
            vm.backupsExpandedKeys.push(key);
        });
        let length = this.backupsExpandedKeys.length;
        for (let i = 0; i < length; i++) {
          vm.getAllParentKey(vm.backupsExpandedKeys[i], vm.treeData);
        }
        vm.expandedKeys = vm.backupsExpandedKeys.slice();
      }
    },
    //获取节点中含有value的所有key集合
    getkeyList(value, tree, keyList) {
      //遍历所有同一级的树
      for (let i = 0; i < tree.length; i++) {
        let node = tree[i];
        //如果该节点存在value值则push
        if (node.name.indexOf(value) > -1) {
          keyList.push(node.key);
        }
        //如果拥有孩子继续遍历
        if (node.children) {
          this.getkeyList(value, node.children, keyList);
        }
      }
      //因为是引用类型，所有每次递归操作的都是同一个数组
      return keyList;
    },
    //该递归主要用于获取key的父亲节点的key值
    getParentKey(key, tree) {
      let parentKey, temp;
      //遍历同级节点
      for (let i = 0; i < tree.length; i++) {
        const node = tree[i];
        if (node.children) {
          //如果该节点的孩子中存在该key值，则该节点就是我们要找的父亲节点
          //如果不存在，继续遍历其子节点
          if (node.children.some((item) => item.key === key)) {
            parentKey = node.key;
          } else if ((temp = this.getParentKey(key, node.children))) {
            //parentKey = this.getParentKey(key,node.children)
            //改进，避免二次遍历
            parentKey = temp;
          }
        }
      }
      return parentKey;
    },
    //获取该节点的所有祖先节点
    getAllParentKey(key, tree) {
      var parentKey;
      if (key) {
        //获得父亲节点
        parentKey = this.getParentKey(key, tree);
        if (parentKey) {
          //如果父亲节点存在，判断是否已经存在于展开列表里，不存在就进行push
          if (!this.backupsExpandedKeys.some((item) => item === parentKey)) {
            this.backupsExpandedKeys.push(parentKey);
          }
          //继续向上查找祖先节点
          this.getAllParentKey(parentKey, tree);
        }
      }
    },
    onExpand(expandedKeys) {
      this.expandedKeys = expandedKeys;
      this.autoExpandParent = false;
    },
    onSelect(selectedKeys, event) {
      let that = this;
      let node = event.selectedNodes[0].data.props;
      if (node.dataRef.isFile) {
        this.src = that.baseUrl + node.path;
      }
    },
    getCatalog(pipelineHistoryId) {
      let that = this;
      axios
        .get(that.baseUrl + catalogPath + pipelineHistoryId)
        .then(function (res) {
          //console.log(res.data);
          that.treeData = res.data;
          that.buildTreeData(that.treeData, "");
        })
        .catch(function (error) {
          console.log(error);
        });
    },
    buildTreeData(data, parentKey) {
      let that = this;
      data.forEach((item, index) => {
        if (parentKey == "") {
          item.key = "" + index;
        } else {
          item.key = parentKey + "-" + index;
        }

        if (!item.isFile) {
          item.scopedSlots = { title: "title", icon: "folder" };
          if (item.children.length > 0) {
            that.buildTreeData(item.children, item.key);
          }
        } else {
          let icon = "file";
          if (item.name.endsWith(".doc")) {
            icon = "word";
          } else if (
            item.name.endsWith(".xls") ||
            item.name.endsWith(".xlsx")
          ) {
            icon = "excel";
          } else if (item.name.endsWith(".xml")) {
            icon = "xml";
          } else if (item.name.endsWith(".json")) {
            icon = "json";
          } else if (item.name.endsWith(".c")) {
            icon = "c";
          } else if (item.name.endsWith(".html")) {
            icon = "html";
          } else if (item.name.endsWith(".js")) {
            icon = "js";
          } else if (item.name.endsWith(".css")) {
            icon = "css";
          }

          item.scopedSlots = { title: "title", icon: icon };
        }
      });
    },
    onRightClick({ event, node }) {
      if (node.dataRef.isFile) {
        const x =
          event.currentTarget.offsetLeft + event.currentTarget.clientWidth;
        const y = event.currentTarget.offsetTop;
        this.NodeTreeItem = {
          pageX: x,
          pageY: y,
          id: node._props.eventKey,
          title: node.dataRef.name,
          path: node.dataRef.downloadPath,
        };
        this.tmpStyle = {
          position: "absolute",
          maxHeight: 40,
          textAlign: "center",
          left: `${x + 10 - 0}px`,
          top: `${y + 3 - 0}px`,
          display: "flex",
          flexDirection: "row",
        };
      }
    },
    clearMenu() {
      this.NodeTreeItem = null;
    },
    downloadFile() {
      let url = this.baseUrl + this.NodeTreeItem.path;
      let filename = this.NodeTreeItem.title;
      axios({
        method: "get",
        url: url,
        responseType: "blob",
      })
        .then((res) => {
          if (res) {
            let url = window.URL.createObjectURL(
              new Blob([res.data], { type: "application/octet-stream" })
            );

            let link = document.createElement("a");
            link.style.display = "none";
            link.href = url;
            link.setAttribute("download", filename);

            document.body.appendChild(link);
            link.click();
            document.body.removeChild(link);
            URL.revokeObjectURL(url);
          }
        })
        .catch((err) => {
          console.log(err);
        });
    },
  },
};
</script>
<style scoped>
.treeDiv {
  height: calc(100% - 4px);
  overflow: auto;
}
::-webkit-scrollbar {
  width: 8px;
  height: 8px;
  /**/
}
::-webkit-scrollbar-track {
  background: rgb(239, 239, 239);
  border-radius: 2px;
}
::-webkit-scrollbar-thumb {
  background: #bfbfbf;
  border-radius: 10px;
}
::-webkit-scrollbar-thumb:hover {
  background: #333;
}
::-webkit-scrollbar-corner {
  background: #bfbfbf;
}
</style>
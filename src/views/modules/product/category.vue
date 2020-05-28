<template>
  <div>
    <el-switch v-model="draggable" active-color="#13ce66" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-button type="warning" v-if="draggable" @click="batchSave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete">批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedkey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="node.level <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >Append</el-button>
          <el-button
            v-if="node.childNodes==0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >Delete</el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">Edit</el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
// 这里可以导入其他文件：组就，工具js，第三方插件js，json文件，图片文件等等
//  例如 import 《组件名称》 from '《组件路径》'

export default {
  // import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data() {
    return {
      menus: [],
      defaultProps: {
        children: "children", // 子节点
        label: "name" // 这里是显示的字段名称
      },
      expandedkey: [], // 默认展开的菜单
      dialogVisible: false,
      category: {
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        catId: 0,
        icon: "",
        productUnit: ""
      }, // 表单提交内容
      dialogType: "", //区分是新增还是编辑 add edit
      title: "", // 弹窗标题
      maxLevel: 0, // 计算节点深度
      updateNodes: [], // 要更新的节点
      draggable: false, // 是否开启拖拽
      pCid: []
    };
  },
  // 方法集合
  methods: {
    // 批量删除
    batchDelete() {
      let catIds = []; // 要删除的菜单id
      let pCid = []; // 打开的菜单pCid
      let checkedNodes = this.$refs.menuTree.getCheckedNodes();
      console.log(checkedNodes);
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId);
        // 找到打开的菜单父节点id
        if (checkedNodes[i].parentCid) {
          pCid.push(checkedNodes[i].parentCid);
        }
      }
      // 确认删除
      this.$confirm(`是否批量删除${catIds}菜单?", "提示`, {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      }).then(() => {
        this.$http({
          url: this.$http.adornUrl("/product/product/category/delete"),
          method: "post",
          data: this.$http.adornData(catIds, false)
        }).then(({ data }) => {
          this.$message({
            type: "success",
            message: "删除成功"
          });
          // 重新获取菜单树
          this.getMenu();
          // 展开打开的菜单
          // 设置默认打开的菜单 [父菜单id]
          // 去重
          this.expandedkey = pCid.filter(
            (item, index, self) => self.indexOf(item) === index
          );
        });
      });
    },

    // 10. 批量保存
    batchSave() {
      // 请求更新
      this.$http({
        url: this.$http.adornUrl("/product/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({ data }) => {
        // 更新成功
        this.$message({
          type: "success",
          message: "菜单顺序更新成功"
        });
        // 刷新菜单
        this.getMenu();
        // 设置默认打开的菜单 [父菜单id]
        this.expandedkey = this.pCid;
        // 重置拖拽的参数
        this.maxLevel = 0;
        this.updateNodes = [];
      });
    },
    // 9. 拖拽结束事件
    // 被拖拽节点对应的 Node、结束拖拽时最后进入的节点、被拖拽节点的放置位置（before、after、inner）、event
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log("tree drop: ", draggingNode, dropNode, dropType);
      // 9.1 当前节点最小的父节点id
      let pCid = 0;
      let siblings = null; // 兄弟节点
      if (dropType == "inner") {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      } else {
        pCid = dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      }

      this.pCid.push(pCid);

      // 9.2 当前拖拽节点的最新顺序
      for (let i = 0; i < siblings.length; i++) {
        // 如果遍历的是当前拖拽的节点，要看其父节点是否发生变化
        if (siblings[i].data.catId == draggingNode.data.catId) {
          // 9.3 当前拖拽节点的最新层级
          let catLevel = draggingNode.level;
          if (siblings[i].catLevel != catLevel) {
            // 当前拖拽节点层级发生改变
            catLevel = siblings[i].level;
            // 修改其他子节点的层级
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      console.log(this.updateNodes);
    },
    // 9.3 修改子节点的层级
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },

    // 8. 菜单拖拽
    allowDrop(draggingNode, dropNode, type) {
      // 被拖动的当前节点一级所在的父节点总层数不能大于3
      // 被拖动的当前节点总层数
      // type : pre next inner
      console.log(draggingNode, dropNode, type);
      // 计算当前节点的
      this.countNodeLevel(draggingNode);
      // 当前拖到的节点 + 父节点的深度不大于3
      let deep = Math.abs(this.maxLevel - draggingNode.level + 1);
      console.log("深度：", deep);
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        return deep + dropNode.parent.level <= 3;
      }
    },
    countNodeLevel(node) {
      // 找到所有子节点，找出最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      }
    },
    // 6. 提交菜单内容
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    // 7. 提交修改
    editCategory() {
      // 获取内容
      var { catId, name, icon, productUnit } = this.category;
      // 构造提交内容
      var data = {
        catId,
        name,
        icon,
        productUnit
      };
      this.$http({
        url: this.$http.adornUrl("/product/product/category/update"),
        method: "post",
        data: this.$http.adornData(data, false)
      }).then(({ data }) => {
        // 更新成功
        this.$message({
          type: "success",
          message: "更新成功"
        });
        // 关闭对话框
        this.dialogVisible = false;
        // 刷新菜单
        this.getMenu();
        // 设置默认打开的菜单 [父菜单id]
        this.expandedkey = [this.category.parentCid];
      });
    },
    // 5. 编辑菜单 - 数据回显
    edit(data) {
      // 复用dialog
      this.dialogVisible = true;
      this.dialogType = "edit"; // 编辑标识
      this.title = "编辑分类";

      // 回显内容,发送http请求获取最新内容
      this.$http({
        url: this.$http.adornUrl(
          `/product/product/category/info/${data.catId}`
        ),
        method: "get",
        params: this.$http.adornData({})
      }).then(({ data }) => {
        // 请求成功
        console.log(data);
        this.category.name = data.data.name;
        this.category.catId = data.data.catId;
        this.category.icon = data.data.icon;
        this.category.productUnit = data.data.productUnit;
        this.category.parentCid = data.data.parentCid;
      });

      console.log(data);
    },
    // 4. 提交添加菜单
    addCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        this.$message({
          type: "success",
          message: "保存成功"
        });
        // 关闭对话框
        this.dialogVisible = false;
        // 刷新菜单
        this.getMenu();
        // 设置默认打开的菜单 [父菜单id]
        this.expandedkey = [this.category.parentCid];
      });
      console.log(this.category);
    },
    // 1. 初始化菜单tree
    getMenu() {
      this.$http({
        url: this.$http.adornUrl("/product/product/category/list/tree"),
        method: "get"
      }).then(({ data }) => {
        // 请求成功进入
        this.menus = data.data;
      });
    },
    // 3. 新增菜单 - 打开弹窗
    append(data) {
      this.dialogVisible = true;
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.dialogType = "add"; // 新增标识
      this.title = "添加分类";
      // 默认值设置 - 防止编辑的数据未清空
      this.category.catId = null;
      this.category.name = "";
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.sort = 0;
      this.category.showStatus = 1;

      console.log(data);
    },
    // 2. 删除菜单
    remove(node, data) {
      var ids = [data.catId];
      var parentCid_ = data.parentCid; // 父节点
      // 弹窗提示
      this.$confirm(`是否删除[${data.name}]?", "提示`, {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false)
          }).then(({ data }) => {
            this.$message({
              type: "success",
              message: "删除成功"
            });
            // 重新获取菜单树
            this.getMenu();
            // 展开原先打开的菜单
            this.expandedkey.push(parentCid_);
          });
        })
        .catch(() => {
          console.log("取消");
        });

      console.log(node, data);
    }
  },
  // 计算属性 类似于data概念
  computed: {},
  // 监控data中的数据变化
  watch: {},
  // 生命周期 - 创建完成（可以访问当前this实例）
  created() {
    // 1.
    this.getMenu();
  },
  // 生命周期 - 挂载完成（可以访问DOM元素）
  mounted() {},
  beforeCreate() {}, // 生命周期 - 创建之前
  beforeMount() {}, // 生命周期 - 挂载之前
  beforeUpdate() {}, // 生命周期 - 更新之前
  updated() {}, // 生命周期 - 更新之后
  beforeDestory() {}, // 生命周期 - 销毁之前
  destoryed() {}, // 生命周期 - 销毁完成
  activated() {} // 如果页面有keep-alive缓存功能，这个函数会触发
};
</script>
<style>
</style>
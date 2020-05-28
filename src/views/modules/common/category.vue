<template>
  <el-tree :data="menus" :props="defaultProps" node-key="catId" ref="menuTree" @node-click="clickNode"></el-tree>
</template>

<script>
// 这里可以导入其他文件：组就，工具js，第三方插件js，json文件，图片文件等等
//  例如 import 《组件名称》 from '《组件路径》'

export default {
  // import 引入的组件需要注入到对象中才能使用
  components: {},
  props: {},
  data() {
    // 这里存放数据
    return {
      menus: [],
      defaultProps: {
        children: "children", // 子节点
        label: "name" // 这里是显示的字段名称
      }
    };
  },
  // 计算属性 类似于data概念
  computed: {},
  // 监控data中的数据变化
  watch: {},
  //方法集合
  methods: {
    clickNode(data, node, component) {
      console.log("子组件category的节点被点击", data, node, component);
      // 向父组件发送事件
      this.$emit("tree-node-click",data, node, component);
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
    }
  },
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
  beforeDestory() {}, //生命周期 - 销毁之前
  destoryed() {}, //生命周期 - 销毁完成
  activated() {} // 如果页面有keep-alive缓存功能，这个函数会触发
};
</script>
<style>
</style>
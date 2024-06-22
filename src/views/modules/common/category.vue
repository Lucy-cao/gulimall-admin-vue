<template>
  <el-tree :data="menus" :props="defaultProps"
           node-key="catId"
           ref="menuTree"
           @node-click="nodeClick">
  </el-tree>
</template>

<script>
/* eslint-disable */
export default {
  data() {
    return {
      menus: [],
      expandKeys: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      },
    }
  },
  methods: {
    getDataList() {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({data}) => { // 通过大括号对响应进行结构，获取实际的数据data
        this.menus = data.data;
      })
    },
    nodeClick(data, node, component){
      console.log("子组件中点击了节点：", data, node, component);
      // 将子组件的数据通过事件传递出去
      this.$emit("tree-node-click", data, node, component)
    }
  },
  created() {
    this.getDataList();
  }
}
</script>

<style>

</style>

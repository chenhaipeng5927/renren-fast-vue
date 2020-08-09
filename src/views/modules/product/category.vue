<!--  -->
<template>
  <el-tree :data="data" :props="defaultProps" @node-click="handleNodeClick"></el-tree>
</template>
<script>
export default {
  components: {},
  data() {
    return {
      data: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },
  // 测试
  methods: {
    handleNodeClick(data) {
      console.log(data);
    },
    getMenus() {
      this.$http({
        url: this.$http.adornUrl("/product/category/list/tree"),
        method: "get",
      }).then(({ data }) => {
        const { data: menu } = data;
        console.log("成功！", menu);
        this.data = menu;
      });
    },
  },
  // 测试
  created() {
    this.getMenus();
  },
};
</script>
<style scoped>
</style>

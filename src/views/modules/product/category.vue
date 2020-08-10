<!--  -->
<template>
  <el-tree
    :data="data"
    :props="defaultProps"
    :expand-on-click-node="false"
    show-checkbox
    node-key="catId"
  >
    <span class="custom-tree-node" slot-scope="{ node, data }">
      <span>{{ node.label }}</span>
      <span>
        <el-button
          v-show="node.level <= 2"
          icon="el-icon-plus"
          plain
          size="mini"
          @click="() => append(data)"
        ></el-button>
        <el-button
          v-if="node.childNodes.length==0"
          icon="el-icon-delete"
          size="mini"
          plain
          @click="() => remove(node, data)"
        ></el-button>
      </span>
    </span>
  </el-tree>
</template>
<script>
export default {
  components: {},
  data() {
    return {
      data: [],
      defaultProps: {
        children: "children",
        label: "name"
      }
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
        method: "get"
      }).then(({ data }) => {
        const { data: menu } = data;
        console.log("成功！", menu);
        this.data = menu;
      });
    },
    append(data) {
      console.log("append", data);
    },

    remove(node, data) {
      console.log("remove", node);
      console.log("data", data);
    }
  },
  // 测试
  created() {
    this.getMenus();
  }
};
</script>
<style scoped>
</style>

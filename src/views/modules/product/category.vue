<!--  -->
<template>
  <el-tree
    :data="data"
    :props="defaultProps"
    :expand-on-click-node="false"
    show-checkbox
    node-key="catId"
    :default-expanded-keys="expandedKey"
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
      expandedKey: [],
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
    append(data) {
      console.log("append", data);
    },

    remove(node, data) {
      var ids = [data.catId];
      this.$confirm(`是否删除【${data.name}】菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            this.$message({
              message: "菜单删除成功",
              type: "success",
            });
            //刷新新的菜单
            this.getMenus();
            //设置默认需要展开的菜单
            this.expandedKey = [node.parent.data.catId];
          });
        })
        .catch(() => {});
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

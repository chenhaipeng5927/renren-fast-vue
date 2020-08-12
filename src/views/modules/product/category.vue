<!--  -->
<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-button v-if="updateNodes.length>=1" @click="batchsave">批量保存</el-button>
    <el-button type="danger" @click="batchDelete" size="mini">删除按钮</el-button>
    <el-tree
      :data="data"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-show="node.level <= 2"
            type="text"
            plain
            size="mini"
            @click="() => append(data)"
          >增加</el-button>
          <el-button
            v-if="node.childNodes.length==0"
            type="text"
            size="mini"
            plain
            @click="() => remove(node, data)"
          >删除</el-button>
          <el-button type="text" size="mini" plain @click="edit(data)">编辑</el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog :title="title" :visible.sync="dialogVisible" width="30%" :closeOnClickModal="false">
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
      <div slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitData">确 定</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
export default {
  components: {},
  data() {
    return {
      pCid: [],
      draggable: false,
      updateNodes: [],
      maxLevel: 0,
      title: "",
      dialogType: "", //edit add
      category: {
        catId: null,
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: "",
        productUnit: "",
      },
      dialogVisible: false,
      data: [],
      expandedKey: [],
      defaultProps: {
        children: "children",
        label: "name",
      },
    };
  },

  methods: {
    batchDelete() {
      let catIds = [];
      let names = [];
      let checkedNodes = this.$refs.menuTree.getCheckedNodes();
      checkedNodes.forEach(function (item) {
        catIds.push(item.catId);
        names.push(item.name);
      });
       this.$confirm(`是否批量删除删除【${names}】菜单?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(catIds, false),
          }).then(({ data }) => {
            this.$message({
              message: "菜单删除成功",
              type: "success",
            });
            //刷新新的菜单
            this.getMenus();
          });
        })
        .catch(() => {});
    },
    batchsave() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update/sort"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单顺序修改成功",
          type: "success",
        });
        //刷新菜单
        this.getMenus();
        this.expandedKey = this.pCid;
        this.updateNodes = [];
        this.pCid = [];
      });
    },
    //拖拽后数据更新
    handleDrop(draggingNode, dropNode, dropType) {
      console.log("tree drop: ", dropNode.label, dropType);
      //1.当前节点的最新父节点的id
      let pCid = 0;
      let siblings = null;
      console.log("--------------------------", dropNode);
      if (dropType == "before" || dropType == "after") {
        pCid =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      } else {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
      }
      this.pCid.push(pCid);
      //2.当前拖拽节点的最新顺序
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            //当前节点层级发生变化
            catLevel = siblings[i].level;
            //修改子节点中的层级
            this.updateChildNodeLevel(siblings[i]);
          }
          //如果遍历的是当前正在拖拽的节点
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel,
          });
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i });
        }
      }
      console.log("updateNodes", this.updateNodes);

      //3.当前拖拽节点的最新层级
    },
    //递归修改子节点level
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data;
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level,
          });
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }

      node.childNodes;
      this.updateNodes.push();
    },
    //是否拖拽
    allowDrop(draggingNode, dropNode, type) {
      //重置最大深度
      this.maxLevel = 0;
      //1.被拖动的当前节点以及所在的父节点总层数不能大于三

      //1）求出最大深度
      this.countNodeLevel(draggingNode);
      //2）求出当前节点的深度
      let deep = this.maxLevel - draggingNode.level + 1;
      console.log("maxLevel", this.maxLevel);
      console.log("deep", deep);
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      } else {
        console.log(type);
        return deep + dropNode.parent.level <= 3;
      }
    },
    countNodeLevel(node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      } else {
        this.maxLevel = node.level;
      }
    },
    //编辑栏提交
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    //修改三级分类
    editCategory() {
      //解构需要的内容
      var { catId, name, icon, productUnit } = this.category;
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        //k,v名称一样时，对象中:v可省略
        // var data = { catId, name, icon, productUnit };//直接作为参数传去
        data: this.$http.adornData({ catId, name, icon, productUnit }, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单修改成功",
          type: "success",
        });
        this.dialogVisible = false;
        this.getMenus();
        console.log(this.category);
        this.expandedKey = [this.category.parentCid];
      });
    },
    //展开编辑菜单 数据渲染
    edit(data) {
      this.dialogType = "edit";
      this.title = "修改分类";
      console.log(data);
      this.dialogVisible = true;
      //发送请求获取当前节点最新的数据
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "get",
      }).then(({ data }) => {
        const { data: categoryV } = data;
        this.category.name = categoryV.name;
        this.category.catId = categoryV.catId;
        this.category.icon = categoryV.icon;
        this.category.productUnit = categoryV.productUnit;
        this.category.parentCid = categoryV.parentCid;
        this.category.catLevel = categoryV.catLevel;
        this.category.showStatus = categoryV.showStatus;
        this.category.sort = categoryV.sort;
      });
    },
    //添加三级分类
    addCategory() {
      console.log(this.category);
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.category, false),
      }).then(({ data }) => {
        this.$message({
          message: "菜单保存成功",
          type: "success",
        });
        this.dialogVisible = false;
        this.getMenus();
        this.expandedKey = [this.category.parentCid];
      });
    },
    //获取数据
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
    //点击添加分类
    append(data) {
      this.dialogType = "add";
      this.dialogVisible = true;
      this.title = "添加分类";
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.name = "";
      this.category.catId = null;
      this.category.icon = "";
      this.category.productUnit = "";
      this.category.showStatus = 1;
      this.category.sort = 0;

      console.log("append", data);
    },
    //删除分类
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

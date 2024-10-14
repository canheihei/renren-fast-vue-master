<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽"
    >
    </el-switch>
    <el-button v-if="draggable" @click="updateSort" type="primary"
      >批量更新</el-button
    >
    <el-button @click="batchDelete" type="danger">批量删除</el-button>
    <el-tree
      :data="data"
      :props="defaultProps"
      :expand-on-click-node="false"
      show-checkbox
      node-key="catId"
      :default-expanded-keys="expandKeys"
      :allow-drop="allowDrop"
      :draggable="draggable"
      ref="tree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button
            v-if="data.catLevel <= 2"
            type="text"
            size="mini"
            @click="() => append(data)"
          >
            添加
          </el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">
            更新
          </el-button>
          <el-button
            v-if="data.childrens.length == 0"
            type="text"
            size="mini"
            @click="() => remove(node, data)"
          >
            删除
          </el-button>
        </span>
      </span>
    </el-tree>

    <el-dialog
      :title="dialogType ? '新增' : '更新'"
      :visible.sync="dialogVisible"
      width="30%"
      :close-on-click-modal="false"
    >
      <el-form :model="categoryForm">
        <el-form-item label="类别名称">
          <el-input v-model="categoryForm.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标">
          <el-input v-model="categoryForm.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input
            v-model="categoryForm.productUnit"
            autocomplete="off"
          ></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitForm">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
/* eslint-disable */
export default {
  data() {
    return {
      pCids: [],
      draggable: false,
      updateNodes: [],
      maxLevel: 0,
      dialogType: false,
      dialogVisible: false,
      categoryForm: {
        name: null,
        icon: null,
        productUnit: null,
        showStatus: 1,
        sort: 0,
        catId: null,
        catLevel: 1,
      },
      data: [],
      expandKeys: [],
      defaultProps: {
        children: "childrens",
        label: "name",
      },
    };
  },
  methods: {
    getCategory() {
      this.$http({
        url: this.$http.adornUrl("/product/category/listTree"),
        method: "get",
      }).then(({ data }) => {
        console.log("成功获取的类别数据：", data.data);
        this.data = data.data;
      });
    },

    handleNodeClick(data) {
      console.log(data);
    },

    append(data) {
      this.dialogVisible = true;
      //console.log("添加", data);
      this.categoryForm.parentCid = data.catId;
      this.categoryForm.catLevel = data.catLevel + 1;
      this.categoryForm.showStatus = 1;
      this.categoryForm.sort = 0;
      this.categoryForm.catId = null;
      this.categoryForm.name = "";
      this.categoryForm.icon = "";
      this.categoryForm.productUnit = "";
      this.dialogType = true;
    },

    batchDelete() {
      let catIds = [];
      let checkedNodes = this.$refs.tree.getCheckedNodes(false, false);
      //console.log("---->",checkedNodes);
      for (let i = 0; i < checkedNodes.length; i++) {
        catIds.push(checkedNodes[i].catId);
      }
      this.$confirm(`是否确认删除【${catIds}】记录?`, "提示", {
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
            if (data && data.code === 0) {
              this.$message({
                message: "操作成功",
                type: "success",
              });
              this.getCategory();
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });
    },

    handleDrop(draggingNode, dropNode, type, event) {
      let parentId = 0;
      let sibings = null;
      if (type == "inner") {
        parentId = dropNode.data.catId;
        siblings = dropNode.childNodes;
      } else {
        parentId =
          dropNode.parent.data.catId == undefined
            ? 0
            : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
      }
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          let catLevel = draggingNode.level;
          if (siblings[i].level != draggingNode.level) {
            catLevel = siblings[i].level;
            this.updateChildNodeLevel(siblings[i]);
          }
          this.updateNodes.push = {
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: parentId,
            catLevel: catLevel,
          };
        } else {
          this.updateNodes.push = { catId: siblings[i].data.catId, sort: i };
        }
      }
      //console.log("----->", this.updateNodes);
      this.updateSort(parentId);
    },

    updateSort(parentId) {
      this.$http({
        url: this.$http.adornUrl("/product/category/updateBatch"),
        method: "post",
        data: this.$http.adornData(this.updateNodes, false),
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "数据拖拽成功",
            type: "success",
          });
          this.getCategory();
          this.expandKeys = this.pCids;
          this.updateNodes = [];
          this.maxLevel = 0;
        } else {
          this.$message.error(data.msg);
        }
      });
    },

    updateChildNodeLevel(node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var childNode = node.childNodes[i].data;
          this.updateNodes.push = {
            catId: childNodes.catId,
            catLevel: node.childNodes[i].level,
          };
          this.updateChildNodeLevel(node.childNodes[i]);
        }
      }
    },

    allowDrop(draggingNode, dropNode, type) {
      this.countNodeLevel(draggingNode);
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1;
      if (type == "inner") {
        return deep + dropNode.level <= 3;
      }
      return deep + dropNode.parent.level <= 3;
    },

    countNodeLevel(node) {
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          this.countNodeLevel(node.childNodes[i]);
        }
      }
    },

    addDialog() {
      this.$http({
        url: this.$http.adornUrl("/product/category/save"),
        method: "post",
        data: this.$http.adornData(this.categoryForm, false),
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "数据添加成功",
            type: "success",
          });
          this.dialogVisible = false;
          this.getCategory();
          this.expandKeys = [this.categoryForm.parentCid];
        } else {
          this.$message.error(data.msg);
        }
      });
    },

    editDialog() {
      this.$http({
        url: this.$http.adornUrl("/product/category/update"),
        method: "post",
        data: this.$http.adornData(this.categoryForm, false),
      }).then(({ data }) => {
        if (data && data.code === 0) {
          this.$message({
            message: "数据更新成功",
            type: "success",
          });
          this.dialogVisible = false;
          this.getCategory();
          this.expandKeys = [this.categoryForm.parentCid];
        } else {
          this.$message.error(data.msg);
        }
      });
    },

    edit(data) {
      this.dialogType = false;
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: "post",
        data: this.$http.adornData(this.categoryForm, false),
      }).then(({ data }) => {
        this.categoryForm.name = data.category.name;
        this.categoryForm.productUnit = data.category.productUnit;
        this.categoryForm.icon = data.category.icon;
        this.categoryForm.catLevel = data.category.catLevel;
        this.categoryForm.parentCid = data.category.parentCid;

        this.categoryForm.catId = data.category.catId;
        this.categoryForm.showStatus = 1;
        this.categoryForm.sort = 0;

        this.dialogVisible = true;
      });
    },

    submitForm() {
      if (this.dialogType) {
        this.addDialog();
      } else {
        this.editDialog();
      }
    },

    remove(node, data) {
      this.$confirm(`是否确认删除【${data.name}】记录?`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(() => {
          let ids = [data.catId];
          //把删除的请求提交到后台服务
          this.$http({
            url: this.$http.adornUrl("/product/category/delete"),
            method: "post",
            data: this.$http.adornData(ids, false),
          }).then(({ data }) => {
            if (data && data.code === 0) {
              this.$message({
                message: "操作成功",
                type: "success",
              });
              this.getCategory();
              this.expandKeys = [node.parent.data.catId];
            } else {
              this.$message.error(data.msg);
            }
          });
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "已取消删除",
          });
        });
      //console.log("删除", data, node);
    },
  },

  created() {
    this.getCategory();
  },
};
</script>
<style>
</style>
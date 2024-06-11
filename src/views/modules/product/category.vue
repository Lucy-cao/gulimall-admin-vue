<template>
  <div>
    <el-switch
      v-model="draggable"
      active-text="开启拖拽"
      inactive-text="关闭拖拽">
    </el-switch>
    <el-button type="primary" @click="batchSave">批量保存</el-button>
    <el-button @click="getDataList">刷新</el-button>
    <el-tree :data="menus" :props="defaultProps"
             :expand-on-click-node="false"
             show-checkbox
             node-key="catId"
             :default-expanded-keys="expandKeys"
             :draggable="draggable"
             :allow-drop="allowDrop"
             @node-drop="handleDrop">
    <span class="custom-tree-node" slot-scope="{ node, data }">
      <span>{{ node.label }}</span>
      <span>
        <el-button v-if="node.level<=2" type="text" size="mini" @click="() => append(data)">Append</el-button>
        <el-button type="text" size="mini" @click="edit(data)">Edit</el-button>
        <el-button v-if="node.childNodes.length==0" type="text" size="mini"
                   @click="() => remove(node, data)">Delete</el-button>
      </span>
    </span>
    </el-tree>
    <el-dialog
      :title="title"
      :visible.sync="dialogVisible"
      width="30%">
      <el-form :model="category">
        <el-form-item label="分类名称" :label-width="formLabelWidth">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图表地址" :label-width="formLabelWidth">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位" :label-width="formLabelWidth">
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
export default {
  name: 'category',
  /* eslint-disable */
  data() {
    return {
      menus: [],
      expandKeys: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      },
      dialogVisible: false,
      dialogType: "",//add:新增，edit:编辑
      category: {
        catId: null,
        name: "",
        parentCid: 0,
        catLevel: 0,
        showStatus: 1,
        sort: 0,
        icon: "",
        productUnit: ""
      },
      formLabelWidth: '120px',
      title: "",//弹窗的标题
      maxLevel: 1,
      updatedNodes: [],
      draggable: false,
      pCid: []
    };
  },
  methods: {
    getDataList() {
      this.$http({
        url: this.$http.adornUrl('/product/category/list/tree'),
        method: 'get'
      }).then(({data}) => { // 通过大括号对响应进行结构，获取实际的数据data
        console.log('获取到菜单数据', data.data);
        this.menus = data.data;
      })
    },
    batchSave() {
      console.log("batchSave");
      //批量保存拖动的结果，弹框确认
      this.$confirm('确定保存拖拽后的结果?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        //向后端发送请求保存拖拽后的数据
        this.$http({
          url: this.$http.adornUrl('/product/category/batchUpdate'),
          method: 'post',
          data: this.$http.adornData(this.updatedNodes, false)
        }).then(({data}) => {
          if (data && data.code === 0) {
            this.$message({
              message: '菜单拖拽成功',
              type: 'success'
            });
            //刷新节点数据
            this.getDataList();
            //清空待更新的节点信息
            this.updatedNodes = [];
            this.expandKeys = this.pCid;
          } else {
            this.$message.error(data.msg)
          }
        })
      });
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log('handleDrop', draggingNode, dropNode, dropType);
      //拖拽后可能产生变化的字段有：父节点id、节点的顺序、节点的层级
      let originLevel = draggingNode.level;
      //1、获取拖拽后的父节点
      let pCid = 0;
      let siblings = null;
      let newCatLevel = 0;
      if (dropType == "inner") {
        pCid = dropNode.data.catId;
        siblings = dropNode.childNodes;
        newCatLevel = dropNode.level + 1;
      } else {
        pCid = dropNode.parent == null ? 0 : dropNode.parent.data.catId;
        siblings = dropNode.parent.childNodes;
        newCatLevel = dropNode.level;
      }
      this.pCid.push(pCid);

      //2、获取拖拽后节点最新的顺序，可能影响拖拽的节点和其兄弟节点的顺序
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          //如果为拖拽的节点，则需要更新父节点id，排序，层级
          this.updatedNodes.push({
            catId: siblings[i].data.catId, parentCid: pCid, sort: i, catLevel: newCatLevel
          })
        } else {
          this.updatedNodes.push({
            catId: siblings[i].data.catId, sort: i
          })
        }
      }

      //3、获取拖拽节点的层级，以及其子节点的层级也可能产生变化
      if (originLevel != newCatLevel) {
        //拖拽的节点的子节点层级也需要变化，并且是递归变化，因为子节点可能还有子节点
        this.updateChildLevel(draggingNode, newCatLevel);
      }
      console.log("this.updatedNodes", this.updatedNodes);
    },
    updateChildLevel(node, newCatLevel) {
      //递归更新子节点的层级
      for (let i = 0; i < node.childNodes.length; i++) {
        this.updatedNodes.push({
          catId: node.childNodes[i].data.catId,
          catLevel: newCatLevel + 1
        })
        this.updateChildLevel(node.childNodes[i], newCatLevel + 1);
      }
    },
    append(data) {
      console.log("append:", data);
      //打开对话框表单，输入要添加的分类名称
      this.dialogVisible = true;
      this.dialogType = "add";
      this.title = "新增分类";
      //给相关的父节点id和等级等字段赋值
      this.category.parentCid = data.catId;
      this.category.catLevel = data.catLevel * 1 + 1;
      this.category.showStatus = 1;
      this.category.sort = 0;
      //以下的字段值都要初始化，否则再次打开弹窗还是之前的内容
      this.category.catId = null;
      this.category.name = "";
      this.category.icon = "";
      this.category.productUnit = "";
    },
    edit(data) {
      console.log("edit:", data);
      //设置弹框相关信息
      this.dialogVisible = true;
      this.dialogType = "edit";
      this.title = "编辑分类";
      //赋值渲染到弹框内
      this.category.catId = data.catId;
      this.category.name = data.name;
      this.category.icon = data.icon;
      this.category.productUnit = data.productUnit;
      this.category.parentCid = data.parentCid;
    },
    submitData() {
      if (this.dialogType == "add") {
        this.addCategory();
      }
      if (this.dialogType == "edit") {
        this.editCategory();
      }
    },
    allowDrop(draggingNode, dropNode, type) {
      //判断节点拖拽到某个位置后是否允许放置
      console.log("拖拽的数据", draggingNode, dropNode, type);
      //判断能否拖拽的关键在于：拖拽某个位置之后，层级是否超过3。如果超过，则不允许拖动，否则允许拖动。
      //获取拖动的节点的最大层级数
      this.maxLevel = draggingNode.level;
      this.countDraggingNodeLevel(draggingNode)
      //计算深度
      let deep = (this.maxLevel - draggingNode.level) + 1;
      //根据位置判断是否有父节点，如果有，加上父节点的层级数
      if (type == "inner") {
        return (deep + dropNode.level) <= 3;
      } else {
        return (deep + dropNode.parent.level) <= 3;
      }
    },
    countDraggingNodeLevel(node) {
      if (node.childNodes != null && node.childNodes.length !== 0) {
        //有子节点，需要判断子节点的中是否嵌套有子节点
        for (let i = 0; i < node.childNodes.length; i++) {
          //当前子节点是否已经超过最大层级数
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level;
          }
          //还需要判断子节点的子节点是否有超过层级数的，是一个递归
          this.countDraggingNodeLevel(node.childNodes[i])
        }
      }
    },
    editCategory() {
      let {catId, name, icon, productUnit} = this.category;
      //调用接口修改分类
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({catId, name, icon, productUnit}, false)
      }).then(({data}) => {
        if (data && data.code === 0) {
          this.$message({
            message: '菜单编辑成功',
            type: 'success'
          });
          //关闭弹框
          this.dialogVisible = false;
          //刷新菜单的数据
          this.getDataList();
          //删除成功之后点击的父节点自动展开
          this.expandKeys = [this.category.parentCid];
        } else {
          this.$message.error(data.msg)
        }
      })
    },
    addCategory() {
      //调用接口新增分类
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({data}) => {
        if (data && data.code === 0) {
          this.$message({
            message: '菜单新增成功',
            type: 'success'
          });
          //关闭弹框
          this.dialogVisible = false;
          //刷新菜单的数据
          this.getDataList();
          //删除成功之后点击的父节点自动展开
          this.expandKeys = [this.category.parentCid];
        } else {
          this.$message.error(data.msg)
        }
      })
    },
    remove(node, data) {
      console.log("remove:", node, data);
      let ids = [data.catId];
      //弹出确认框，让用户二次确认
      this.$confirm(`是否删除【${data.name}】菜单?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        //点击确定执行then里面的代码
        //调用删除接口
        this.$http({
          url: this.$http.adornUrl('/product/category/delete'),
          method: 'post',
          data: this.$http.adornData(ids, false)
        }).then(({data}) => {
          if (data && data.code === 0) {
            this.$message({
              message: '删除成功',
              type: 'success'
            });
            //刷新菜单的数据
            this.getDataList();
            //删除成功之后当前节点的父节点不自动收起。
            // 因为auto-expand-parent展开子节点的时候是否自动展开父节点默认设置为true，所以只需要展示当前节点的父节点，然后父节点的父节点也会展开
            this.expandKeys = [node.parent.data.catId];
          } else {
            this.$message.error(data.msg)
          }
        })
      }).catch(() => {
        //点击取消执行catch里面的代码
      });
    }
  },
  created() {
    this.getDataList();
  }
}
</script>

<style scoped>

</style>

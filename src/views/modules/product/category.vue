<template>
  <div>
    <el-tree :data="menus" :props="defaultProps" :expand-on-click-node="false" show-checkbox node-key="catId"
             :default-expanded-keys="expandKeys">
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
      title: ""//弹窗的标题
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
    editCategory(){
      let {catId, name, icon, productUnit}=this.category;
      //调用接口修改分类
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({catId,name,icon,productUnit}, false)
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
      console.log("remove:", node, data);
    }
  },
  created() {
    this.getDataList();
  }
}
</script>

<style scoped>

</style>

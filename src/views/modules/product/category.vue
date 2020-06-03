<!--  -->
<template>
  <div>
    <el-switch v-model="draggable" active-text="开启拖拽" inactive-text="关闭拖拽"></el-switch>
    <el-button v-if="draggable" type="primary">批量修改</el-button>
    <el-button type="danger" @click="batchDelete" :disabled="disabled">批量删除</el-button>
    <el-tree
      :data="menus"
      :props="defaultProps"
      show-checkbox
      :expand-on-click-node="false"
      node-key="catId"
      :default-expanded-keys="expandedKey"
      :draggable="draggable"
      :allow-drop="allowDrop"
      @node-drop="handleDrop"
      @check-change="handleCheckChange"
      ref="menuTree"
    >
      <span class="custom-tree-node" slot-scope="{ node, data }">
        <span>{{ node.label }}</span>
        <span>
          <el-button type="text" v-if="node.level<=2" size="mini" @click="() => append(data)">Append</el-button>
          <el-button type="text" size="mini" @click="() => edit(data)">Edit</el-button>
          <el-button
            type="text"
            v-if="node.childNodes.length==0"
            size="mini"
            @click="() => remove(node, data)"
          >Delete</el-button>
        </span>
      </span>
    </el-tree>
    <el-dialog title="提示" :visible="dialogVisible" width="30%">
      <el-form :model="category">
        <el-form-item label="分类名称">
          <el-input v-model="category.name" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="图标地址">
          <el-input v-model="category.icon" autocomplete="off"></el-input>
        </el-form-item>
        <el-form-item label="计量单位">
          <el-input v-model="category.productUnit" autocomplete="off"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="submitCategory()">确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>
 
<script>
//这里可以导入其他文件（比如：组件，工具js，第三方插件js，json文件，图片文件等等）
//例如：import 《组件名称》 from '《组件路径》';

export default {
  //import引入的组件需要注入到对象中才能使用
  components: {},
  data() {
    return {
      disabled: true,
      pCid: [],
      draggable: false,
      updateNodes: [],
      maxLevel: 0,
      dialogType: '',
      category: { name: '', catId: null, parentCid: 0, showStatus: 1, sort: 0, icon: null, productUnit: '', catLevel: 0 },
      dialogVisible: false,
      menus: [],
      expandedKey: [],
      defaultProps: {
        children: 'children',
        label: 'name'
      }
    }
  },
  //监听属性 类似于data概念
  computed: {},
  //监控data中的数据变化
  watch: {
  },
  //方法集合
  methods: {
    batchDelete() {
      //被选中的节点 返回的是一个数组
      let checkedNodes = this.$refs.menuTree.getCheckedNodes()
      let checkedNodeName = []
      let showNodeName = []
      let ids = []
      for (let i = 0; i < checkedNodes.length; i++) {
        console.log('checkedNodes', checkedNodes[i])
        checkedNodeName.push(checkedNodes[i].name)
        ids.push(checkedNodes[i].catId)
      }
      for (let i = 0; i < checkedNodeName.length; i++) {
        if (i < 6) {
          showNodeName.push(checkedNodes[i].name)
        } else {
          showNodeName.push('......')
          break
        }
      }
      this.$confirm(`此操作将删除【${showNodeName}】该文件, 是否继续?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() => {
          this.$http({
            url: this.$http.adornUrl('/product/category/delete'),
            method: 'post',
            data: this.$http.adornData(ids, false)
          }).then(({ data }) => {
            this.$message({
              type: 'success',
              message: '删除成功!'
            })
            this.getCategory()
          })
        })
        .catch(() => {})
    },
    batchSave() {
      this.$http({
        url: this.$http.adornUrl('/product/category/update/sort'),
        method: 'post',
        data: this.$http.adornData(this.updateNodes, false)
      }).then(({ data }) => {
        this.$message({
          message: '菜单顺序等修改成功',
          type: 'success'
        })
        //刷新出新的菜单
        this.getMenus()
        //设置需要默认展开的菜单
        this.expandedKey = this.pCid
        this.updateNodes = []
        this.maxLevel = 0
        // this.pCid = 0;
      })
    },
    handleDrop(draggingNode, dropNode, dropType, ev) {
      console.log('handleDrop: ', draggingNode, dropNode, dropType)
      //1、当前节点最新的父节点id
      let pCid = 0
      let siblings = null
      if (dropType == 'before' || dropType == 'after') {
        pCid = dropNode.parent.data.catId == undefined ? 0 : dropNode.parent.data.catId
        siblings = dropNode.parent.childNodes
      } else {
        pCid = dropNode.data.catId
        siblings = dropNode.childNodes
      }
      this.pCid.push(pCid)

      //2、当前拖拽节点的最新顺序，
      for (let i = 0; i < siblings.length; i++) {
        if (siblings[i].data.catId == draggingNode.data.catId) {
          //如果遍历的是当前正在拖拽的节点
          let catLevel = draggingNode.level
          if (siblings[i].level != draggingNode.level) {
            //当前节点的层级发生变化
            catLevel = siblings[i].level
            //修改他子节点的层级
            this.updateChildNodeLevel(siblings[i])
          }
          this.updateNodes.push({
            catId: siblings[i].data.catId,
            sort: i,
            parentCid: pCid,
            catLevel: catLevel
          })
        } else {
          this.updateNodes.push({ catId: siblings[i].data.catId, sort: i })
        }
      }

      //3、当前拖拽节点的最新层级
      console.log('updateNodes', this.updateNodes)
    },
    updateChildNodeLevel(node) {
      if (node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          var cNode = node.childNodes[i].data
          this.updateNodes.push({
            catId: cNode.catId,
            catLevel: node.childNodes[i].level
          })
          this.updateChildNodeLevel(node.childNodes[i])
        }
      }
    },
    handleCheckChange() {
      if (this.$refs.menuTree.getCheckedNodes().length != 0) {
        this.disabled = false
      }else{
        this.disabled = true
      }
    },
    allowDrop(draggingNode, dropNode, type) {
      //1、被拖动的当前节点以及所在的父节点总层数不能大于3
      //1）、被拖动的当前节点总层数
      console.log('allowDrop:', draggingNode, dropNode, type)
      //
      this.countNodeLevel(draggingNode)
      //当前正在拖动的节点+父节点所在的深度不大于3即可
      let deep = Math.abs(this.maxLevel - draggingNode.level) + 1
      console.log('深度：', deep)
      //   this.maxLevel
      if (type == 'inner') {
        // console.log(
        //   `this.maxLevel：${this.maxLevel}；draggingNode.data.catLevel：${draggingNode.data.catLevel}；dropNode.level：${dropNode.level}`
        // );
        return deep + dropNode.level <= 3
      } else {
        return deep + dropNode.parent.level <= 3
      }
    },
    countNodeLevel(node) {
      q
      //找到所有子节点，求出最大深度
      if (node.childNodes != null && node.childNodes.length > 0) {
        for (let i = 0; i < node.childNodes.length; i++) {
          if (node.childNodes[i].level > this.maxLevel) {
            this.maxLevel = node.childNodes[i].level
          }
          this.countNodeLevel(node.childNodes[i])
        }
      }
    },
    getCategory() {
      this.$http({
        url: this.$http.adornUrl('/product/category/list'),
        method: 'get'
      }).then(({ data }) => {
        console.log(data.page)
        this.menus = data.page
      })
    },
    submitCategory() {
      console.log(this.dialogType)
      if (this.dialogType === 'add') {
        console.log('add category')
        this.addCategory()
      } else if (this.dialogType === 'edit') {
        console.log('edit category')
        this.editCategory()
      }
    },
    addCategory() {
      this.$http({
        url: this.$http.adornUrl('/product/category/save'),
        method: 'post',
        data: this.$http.adornData(this.category, false)
      }).then(({ data }) => {
        this.dialogVisible = false
        this.$message({
          message: '添加成功',
          type: 'success'
        })
        this.getCategory()
        this.expandedKey = [this.category.parentCid]
      })
    },
    edit(data) {
      this.dialogType = 'edit'
      console.log(this.dialogType)
      this.dialogVisible = true
      //查询出当前的分类
      this.$http({
        url: this.$http.adornUrl(`/product/category/info/${data.catId}`),
        method: 'get'
      }).then(({ data }) => {
        this.category = data.data
      })
      this.category.catId = data.catId
    },
    editCategory() {
      var { name, productUnit, icon, catId } = this.category
      console.log({ name, productUnit, icon, catId })
      this.$http({
        url: this.$http.adornUrl('/product/category/update'),
        method: 'post',
        data: this.$http.adornData({ name, productUnit, icon, catId }, false)
      }).then(({ data }) => {
        this.dialogVisible = false
        this.$message({
          message: '修改成功',
          type: 'success'
        })
        this.getCategory()
        this.expandedKey = [this.category.parentCid]
        this.category = {}
      })
    },
    append(data) {
      console.log(this.dialogType)
      this.dialogType = 'add'
      this.dialogVisible = true
      this.category.parentCid = data.catId
      this.category.catLevel = data.catLevel * 1 + 1
      this.category.showStatus = 1
      this.category.sort = 0
      console.log('category' + this.category)
    },
    remove(node, data) {
      console.log(node, data)
      var ids = [data.catId]
      this.$confirm(`此操作将删除【${data.name}】该文件, 是否继续?`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      })
        .then(() => {
          this.$message(
            {
              type: 'success',
              message: '删除成功!'
            },
            this.$http({
              url: this.$http.adornUrl('/product/category/delete'),
              method: 'post',
              data: this.$http.adornData(ids, false)
            }).then(({ data }) => {
              this.getCategory()
              this.expandedKey = [node.parent.data.catId]
            })
          )
        })
        .catch(() => {
          this.$message({
            type: 'info',
            message: '已取消删除'
          })
        })
    }
  },
  //生命周期 - 创建完成（可以访问当前this实例）
  created() {
    this.getCategory()
  },
  //生命周期 - 挂载完成（可以访问DOM元素）
  mounted() {},
  beforeCreate() {}, //生命周期 - 创建之前
  beforeMount() {}, //生命周期 - 挂载之前
  beforeUpdate() {}, //生命周期 - 更新之前
  updated() {}, //生命周期 - 更新之后
  beforeDestroy() {}, //生命周期 - 销毁之前
  destroyed() {}, //生命周期 - 销毁完成
  activated() {} //如果页面有keep-alive缓存功能，这个函数会触发
}
</script>
<style scoped>
</style>
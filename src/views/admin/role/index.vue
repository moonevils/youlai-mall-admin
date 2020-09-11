<template>
  <div class="app-container">
    <el-form ref="queryForm" :model="queryParams" size="small" :inline="true">
      <el-form-item>
        <el-button type="primary" icon="el-icon-plus" @click="handleAdd">新增</el-button>
        <el-button type="success" icon="el-icon-edit" :disabled="single" @click="handleUpdate">修改</el-button>
        <el-button type="danger" icon="el-icon-delete" :disabled="multiple" @click="handleDelete">删除</el-button>
      </el-form-item>
      <el-form-item label="角色名称" prop="name">
        <el-input
          v-model="queryParams.name"
          placeholder="角色名称"
          clearable
          style="width: 240px"
          @keyup.enter.native="handleQuery"
        />
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="el-icon-search" @click="handleQuery">搜索</el-button>
        <el-button icon="el-icon-refresh" @click="handleResetQuery">重置</el-button>
      </el-form-item>
    </el-form>

    <el-table v-loading="loading" :data="pageList" border @selection-change="handleSelectionChange">
      <el-table-column type="selection" width="55" align="center"/>
      <el-table-column label="角色编号" prop="id" width="280"/>
      <el-table-column label="角色名称" prop="name" :show-overflow-tooltip="true" width="200"/>
      <el-table-column label="权限字符" prop="perms" :show-overflow-tooltip="true" width="200"/>
      <el-table-column label="显示顺序" prop="sort" width="100"/>
      <el-table-column label="状态" align="center" width="100">
        <template slot-scope="scope">
          <el-switch
            v-model="scope.row.status"
            :active-value="0"
            :inactive-value="1"
            @change="handleStatusChange(scope.row)"
          />
        </template>
      </el-table-column>
      <el-table-column label="备注" prop="remark"/>
      <el-table-column label="操作" align="center" width="180" class-name="small-padding fixed-width">
        <template slot-scope="scope">
          <el-button
            type="text"
            icon="el-icon-edit"
            @click="handleUpdate(scope.row)"
          >修改
          </el-button>
          <el-button
            type="text"
            icon="el-icon-delete"
            @click="handleDelete(scope.row)"
          >删除
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <pagination
      v-show="pagination.total>0"
      :total="pagination.total"
      :page.sync="pagination.page"
      :limit.sync="pagination.limit"
      @pagination="handleQuery"
    />

    <!-- 添加或修改角色配置对话框 -->
    <el-dialog :title="dialog.title" :visible.sync="dialog.visible" width="800px">
      <el-form ref="form" :model="form" :rules="rules" label-width="80px">
        <el-form-item label="角色名称" prop="name">
          <el-input v-model="form.name" placeholder="请输入角色名称"/>
        </el-form-item>
        <el-form-item label="权限字符" prop="perms">
          <el-input v-model="form.perms" placeholder="请输入权限字符"/>
        </el-form-item>
        <el-form-item label="角色顺序" prop="sort">
          <el-input-number v-model="form.sort" controls-position="right" :min="0"/>
        </el-form-item>
        <el-form-item label="状态">
          <el-radio-group v-model="form.status">
            <el-radio :label="1">正常</el-radio>
            <el-radio :label="0">停用</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="菜单权限">
          <el-tree
            ref="menu"
            :data="menuOptions"
            show-checkbox
            node-key="id"
            empty-text="加载中，请稍后"
          />
        </el-form-item>
        <el-form-item label="备注">
          <el-input v-model="form.remark" type="textarea" placeholder="请输入内容"/>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="handleSubmit">确 定</el-button>
        <el-button @click="dialog.visible=false">取 消</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
  import {list, detail, update, add, del, patch} from '@/api/admin/role'
  import {list as menuList} from '@/api/admin/menu'

  import store from '@/store'

  export default {
    data() {
      return {
        // 遮罩层
        loading: true,
        // 选中数组
        ids: [],
        // 非单个禁用
        single: true,
        // 非多个禁用
        multiple: true,
        queryParams: {
          name: undefined
        },
        pagination: {
          page: 1,
          limit: 10,
          total: 0
        },
        pageList: [],
        dialog: {
          title: undefined,
          visible: false
        },
        // 菜单列表
        menuOptions: [],
        // 表单参数
        form: {},
        // 表单校验
        rules: {
          name: [
            {required: true, message: '角色名称不能为空', trigger: 'blur'}
          ],
          perms: [
            {required: true, message: '权限字符不能为空', trigger: 'blur'}
          ]
        }
      }
    },
    created() {
      this.handleQuery()
    },
    methods: {
      handleQuery() {
        this.queryParams.page = this.pagination.page
        this.queryParams.limit = this.pagination.limit
        list(this.queryParams).then(response => {
          this.pageList = response.data
          this.pagination.total = response.total
          this.loading = false
        })
      },
      handleResetQuery() {
        this.pagination = {
          page: 1,
          limit: 10,
          total: 0
        }
        this.queryParams = {
          name: undefined
        }
        this.handleQuery()
      },
      handleSelectionChange(selection) {
        this.ids = selection.map(item => item.id)
        this.single = selection.length != 1
        this.multiple = !selection.length
      },
      handleStatusChange(row) {
        const text = row.status === 0 ? '启用' : '停用'
        this.$confirm('确认要"' + text + '""' + row.name + '"角色吗?', '警告', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(function () {
          return patch(row.id, {status: row.status})
        }).then(() => {
          this.$message.success(text + '成功')
        }).catch(function () {
          row.status = row.status === 0 ? 1 : 0
        })
      },
      handleAdd() {
        this.resetForm()
        this.dialog = {
          title: '新增角色',
          visible: true
        }
        this.loadMenuOptions()
      },
      async handleUpdate(row) {
        this.resetForm()
        this.dialog = {
          title: '修改角色',
          visible: true
        }
        const id = row.id || this.ids
        await this.loadRoleMenuOptions(id)
        detail(id).then(response => {
          this.form = response.data
        })
      },
      handleDelete(row) {
        const ids = row.id || this.ids
        this.$confirm('是否确认删除名称为"' + row.name + '"的数据项?', '警告', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          del(ids).then(() => {
            this.$message.success('删除成功')
            this.handleQuery()
          })
        }).catch(() =>
          this.$message.info('已取消删除')
        )
      },
      handleSubmit: function () {
        this.$refs['form'].validate(valid => {
          if (valid) {
            const id = this.form.id
            this.form.menuIds = this.getMenuAllCheckedKeys()
            if (id != undefined) {
              update(this.form.id, this.form).then(() => {
                this.$message.success('修改成功')
                this.dialog.visible = false
                this.handleQuery()
              })
            } else {
              this.form.menuIds = this.getMenuAllCheckedKeys()
              add(this.form).then(() => {
                this.$message.success('新增成功')
                this.dialog.visible = false
                this.handleQuery()
              })
            }
          }
        })
      },
      // 表单重置
      resetForm() {
        if (this.$refs.tree != undefined) {
          this.$refs.tree.setCheckedKeys([])
        }
        this.form = {
          id: undefined,
          name: undefined,
          perms: undefined,
          sort: 0,
          status: 0,
          menuIds: [],
          remark: undefined
        }
        if (this.$refs['form']) {
          this.$refs['form'].resetFields()
        }
      },
      loadMenuOptions() {
        menuList({mode: 2}).then(response => {
          this.menuOptions = response.data
        })
      },
      loadRoleMenuOptions(roleId) {
        menuList({mode: 2, roleId: roleId}).then(response => {
          this.menuOptions = response.data.menus
          this.$refs.menu.setCheckedKeys(response.data.checkedKeys);
        })
      },
      // 所有菜单节点数据
      getMenuAllCheckedKeys() {
        // 目前被选中的菜单节点
        const checkedKeys = this.$refs.menu.getHalfCheckedKeys()
        // 半选中的菜单节点
        const halfCheckedKeys = this.$refs.menu.getCheckedKeys()
        checkedKeys.unshift.apply(checkedKeys, halfCheckedKeys)
        return checkedKeys
      }
    }
  }
</script>
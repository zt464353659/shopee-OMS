<template>
  <div>
    <!-- 搜索 -->
    <div class="header-box">
      <el-form
        ref="listQuery"
        :inline="true"
        :model="listQuery"
        class="demo-form-inline"
        size="mini"
      >
        <el-form-item label="站点" prop="site_code">
          <el-select v-model="listQuery.site_code" clearable placeholder="选择站点">
            <el-option v-for="(item, index) in siteCode" :label="item" :value="item" :key="index" />
          </el-select>
        </el-form-item>
        <el-form-item label="账号" prop="account">
          <el-input v-model="listQuery.account" clearable placeholder="请输入账号" />
        </el-form-item>
        <el-form-item label="状态" prop="active">
          <el-select v-model="listQuery.active" clearable placeholder="请选择状态">
            <el-option label="启用" :value="1" />
            <el-option label="禁用" :value="0" />
          </el-select>
        </el-form-item>
        <el-form-item label="负责人" prop="duly_name">
          <el-input v-model="listQuery.duly_name" clearable placeholder="请输入负责人" />
        </el-form-item>
        <el-form-item label="创建人" prop="create_user_name">
          <el-input v-model="listQuery.create_user_name" clearable placeholder="请输入用户" />
        </el-form-item>
        <el-form-item>
          <el-button type="primary" @click="handleFilter">搜索</el-button>
          <el-button data-type="clear" @click="clearSearch">清空</el-button>
        </el-form-item>
      </el-form>
      <!--<label class="title">计划列表</label>-->
      <el-row class="right-row">
        <el-button
          type="primary"
          size="mini"
          icon="el-icon-circle-plus-outline"
          @click="addSite('add')"
          v-permission="permission.add"
        >
          添加账号
        </el-button>
      </el-row>
    </div>
    <!-- 列表 -->
    <div class="content-box">
      <el-table
        :key="newDate"
        :data="listData"
        v-loading="listLoading"
        border
        :default-sort="{prop: 'create_time', order: 'descending'}"
        empty-text="暂无相关结果"
        :max-height="maxHeight"
        style="width: 100%"
        @filter-change="handleFilterChange"
      >
        <el-table-column prop="id" label="ID" width="80" />
        <el-table-column prop="site_code" label="站点" width="80" />
        <el-table-column prop="account" label="账号" min-width="100" />
        <el-table-column
          prop="active"
          label="状态"
          width="100"
          column-key="active"
        >
          <template slot-scope="scope">
            <el-tag
              :type="scope.row.active === '1' ? 'success' : 'info'"
              size="small"
              disable-transitions
            >
              {{ scope.row.active === '1' ? '启用' : '禁用' }}
            </el-tag>
          </template>
        </el-table-column>
        <el-table-column prop="duly_name" label="负责人" min-width="100" />
        <el-table-column prop="create_user_name" label="创建人" />
        <el-table-column label="创建时间">
          <template slot-scope="scope">
            <div>{{ fattTime(scope.row.create_time) }}</div>
          </template>
        </el-table-column>
        <el-table-column label="操作" width="120" align="center">
          <template slot-scope="scope">
            <el-button
              type="text"
              size="mini"
              v-permission="permission.active"
              @click="handleChangeStatus(scope.row)"
            >
              {{ scope.row.active === '1' ? '禁用': '启用' }}
            </el-button>
            <el-button
              type="text"
              size="mini"
              @click="editSite(scope.row, 'edit')"
              v-permission="permission.edit"
            >
              编辑
            </el-button>
            <el-button
              type="text"
              size="mini"
              @click="deleteUser(scope.row)"
              v-permission="permission.delete"
            >
              删除
            </el-button>
          </template>
        </el-table-column>
      </el-table>
      <!--分页-->
      <div class="pagination-container">
        <el-pagination
          background
          layout="total, sizes, prev, pager, next, jumper"
          small
          @size-change="handleSizeChange"
          @current-change="handleCurrentChange"
          :current-page="listQuery.page"
          :page-sizes="[10, 20, 30, 50]"
          :page-size="listQuery.per_page"
          :total="pagination ? pagination.total : 0"
        />
      </div>
    </div>
    <!--添加计划弹窗dialog-->
    <add-form v-bind.sync="detailsSiteDialogOption" @reload="renderList" />
  </div>
</template>

<script>
  import { fetchPlanList, delAccount, apiUpdateAccountStatus } from '@/api/account'
  import addForm from './manage/addForm'
  import { filterQueryParams, fattCreateTime } from '@/utils'

  export default {
    name: 'UserManage',
    components: { addForm },
    data() {
      return {
        permission: {
          add: 'operation.account.account.add',
          delete: 'operation.account.account.delete',
          edit: 'operation.account.account.set',
          active: 'operation.account.account.active'
        },
        newDate: new Date().getTime(),
        status: '状态 ',
        title: '类型 ',
        filterTypeIcon: 'filter',
        filterStatusIcon: 'filter',
        maxHeight: document.documentElement.clientHeight - 200,
        detailsSiteDialogOption: {
          formData: {},
          siteCode: [],
          open: false
        },
        listData: [],
        listLoading: true,
        listQuery: {
          page: 1,
          per_page: 10,
          active: undefined,
          account: undefined,
          create_user_name: undefined,
          site_code: undefined,
          duly_name: undefined
        },
        siteCode: [],
        pagination: null
      }
    },
    created() {
      this.renderList()
      this.maxHeight = this.maxHeight < 200 ? 200 : this.maxHeight
    },
    mounted() {
      const that = this
      window.onresize = () => {
        return (() => {
          window.maxHeight = document.documentElement.clientHeight - 200
          that.maxHeight = window.maxHeight < 200 ? 200 : window.maxHeight
        })()
      }
    },
    methods: {
      renderList() {
        this.listLoading = true
        this.listData = null
        const queryParams = filterQueryParams(this.listQuery)
        this._.forEach(queryParams, (v, i) => {
          if (queryParams[i] || queryParams[i] === 0) {
            queryParams[i] = this.trim(v)
          }
        })
        if (queryParams.type === 'all') {
          delete queryParams.type
        }
        fetchPlanList(queryParams)
          .then(response => {
            this.listLoading = false
            this.listData = response.data.list
            this.siteCode = response.data.site_code
            this.pagination = response.data.pagination
            document.querySelector('.el-table__body-wrapper').scrollTop = 0
          })
          .catch(() => {
            this.listLoading = false
          })
      },
      handleSizeChange(val) {
        this.listQuery.page = 1
        this.listQuery.per_page = val
        this.renderList()
      },
      handleCurrentChange(val) {
        this.listQuery.page = val
        this.renderList()
      },
      editSite(data, operation) {
        const d = this._.cloneDeep(data)
        this.detailsSiteDialogOption = {
          open: true,
          siteCode: this.siteCode,
          formData: d,
          operation: operation
        }
      },
      addSite(operation) {
        this.detailsSiteDialogOption = {
          open: true,
          siteCode: this.siteCode,
          formData: {},
          operation: operation
        }
      },
      clearSearch() {
        this.$refs.listQuery.resetFields()
        this.newDate = new Date().getTime()
        this.filterTypeIcon = 'filter'
        this.filterStatusIcon = 'filter'
        this.renderList()
      },
      handleFilter() {
        this.listQuery.page = 1
        this.newDate = new Date().getTime()
        this.filterTypeIcon = 'filter'
        this.filterStatusIcon = 'filter'
        this.renderList()
      },
      // 筛选图标处理
      handleFilterChange(filters) {
        if (filters.status instanceof Array) {
          if (filters.status.length === 0) {
            this.filter = 'filter'
            document.querySelector(' .el-table th>.cell>span>svg').style.color =
              '#c0c4cc'
          } else {
            this.filter = 'filter-in'
            document.querySelector(
              ' .el-table th>.cell.highlight>span>svg'
            ).style.color = '#f00'
          }
        }
      },
      /* 用户删除 */
      deleteUser(d) {
        const data = {
          id: d.id,
          update_user_id: this.$store.getters.userInfo.id,
          update_user_name: this.$store.getters.userInfo.name
        }
        this.$confirm('确认删除？', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          closeOnClickModal: false,
          closeOnPressEscape: false,
          type: 'warning'
        })
          .then(_ => {
            delAccount(data).then(response => {
              this.renderList()
            })
          })
          .catch(_ => {})
      },
      handleChangeStatus(row) {
        const tips = row.active === '1' ? '禁用' : '启用'
        const active = row.active === '1' ? '0' : '1'
        this.$confirm(`确定${tips}该账号吗?`, '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.listLoading = true
          apiUpdateAccountStatus({ id: row.id, active })
            .then(res => {
              this.renderList()
              this.listLoading = true
            })
            .catch(() => { this.listLoading = false })
        })
      },
      // 格式化时间
      fattTime(time) {
        return fattCreateTime(Number(time) * 1000)
      },
      // 去除前后空格
      trim(str) {
        return String(str).replace(/^\s+|\s+$/g, '')
      }
    },
    watch: {
      maxHeight(val) {
        if (!this.timer) {
          this.maxHeight = val
          this.timer = true
          const that = this
          setTimeout(function() {
            that.timer = false
          }, 400)
        }
      }
    }
  }
</script>
<style rel="stylesheet/scss" lang="scss" scoped>
.el-popover {
  max-height: 400px !important;
  overflow-y: auto !important;
}
</style>

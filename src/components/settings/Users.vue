<template>
  <div class="users">
    <div class="box-tools pull-right">
      <div style="display: inline-block;margin-right:15px;" @mouseenter="toggleSearch(true)" @mouseleave="toggleSearch(false)">
        <Select v-model="searchBy" class="table-search" :class="{on: tableSearchOn||searchMode||searchBy}">
          <Option value="">不限</Option>
          <Option :value="col.subKey ? col.key +'.' + col.subKey : col.key" v-for="col in columns" :key="col.key" v-if="col.title&&col.searchable">{{col.title}}</Option>
        </Select>
        <Input v-model="keyword" ref="tableSearch" placeholder="搜索关键字" @on-focus="toggleSearchMode" @on-blur="toggleSearchMode" class="table-search" :class="{on: tableSearchOn||searchMode||keyword}" clearable></Input>
        <Button type="ghost" icon="ios-search" @click="search"></Button>
      </div>
      <Button-group>
        <Button type="ghost" icon="ios-refresh-empty" @click="refreshList">刷新</Button>
      </Button-group>
    </div>
    <Table stripe :loading="loading" :height="tableHeight" :columns="columns" :data="dataViewPage" ref="table"></Table>
    <div style="margin: 10px;overflow: hidden">
      <!-- <Button type="primary" size="large" @click="exportData" :disabled="!rowSelected"><Icon type="ios-download-outline"></Icon> 导出原始数据</Button> -->
      <div style="float: right;">
        <Page :total="data.length" :page-size-opts="[10,20,50,100]" @on-change="changePage" @on-page-size-change="changePageSize" :current="pageCurrent" show-sizer show-total show-elevator></Page>
      </div>
    </div>
    <Modal
      v-model="editModal"
      title="修改用户"
      :mask-closable="false"
      :transfer="false">
      <div class="modal-content">
        <Form ref="editForm" :model="editModel" :rules="ruleValidate" :label-width="120">
          <FormItem label="名称" prop="name">
            <Input v-model="editModel.name"></Input>
          </FormItem>
          <FormItem label="店群ID" prop="group">
            <Input v-model="editModel.group"></Input>
          </FormItem>
        </Form>
      </div>
      <div slot="footer">
        <Button size="large" @click="editModal=false">关闭</Button>
        <Button type="success" size="large" @click="submitEdit">提交</Button>
      </div>
    </Modal>
    <Modal
      v-model="deleteModal"
      title="删除用户"
      :mask-closable="false"
      :transfer="false">
      <div class="modal-content">
        确认删除用户"{{deleteModel.name}}"吗？
      </div>
      <div slot="footer">
        <Button size="large" @click="deleteModal=false">取消</Button>
        <Button type="error" size="large" @click="submitDelete">确认</Button>
      </div>
    </Modal>
  </div>
</template>

<script>
export default {
  name: 'users',
  data () {
    return {
      apiItem: {
        apiHost: '',
        apiService: 'users',
        apiAction: 'listall',
        apiQuery: {}
      },
      apiData: {},
      columns: [
        // { type: 'selection', width: 70, align: 'center' },
        {
          title: 'ID',
          key: 'id',
          ellipsis: true,
          sortable: true,
          searchable: true,
          render: (h, params) => {
            return h('span', {}, params.row.id)
          }
        },
        { title: '名称',
          key: 'name',
          ellipsis: true,
          sortable: true,
          searchable: true,
          sortMethod: (a, b, type) => {
            if (type === 'asc') {
              return a.localeCompare(b, 'zh-CN')
            } else {
              return b.localeCompare(a, 'zh-CN')
            }
          },
          render: (h, params) => {
            return h('span', {}, params.row.name)
          }
        },
        { title: '店群ID',
          key: 'group',
          sortable: true,
          searchable: true,
          sortMethod: (a, b, type) => {
            if (type === 'asc') {
              return a.localeCompare(b, 'zh-CN')
            } else {
              return b.localeCompare(a, 'zh-CN')
            }
          },
          render: (h, params) => {
            return h('span', {}, params.row.group)
          }
        },
        {
          title: '操作',
          key: 'action',
          fixed: 'right',
          render: (h, params) => {
            return h('Button-group', [
              h('Button', {
                props: {
                  type: 'ghost',
                  size: 'small'
                },
                style: {
                },
                on: {
                  click: async () => {
                    this.editModal = true
                    this.editModel = {
                      id: params.row.id,
                      name: params.row.name,
                      group: params.row.group
                    }
                  }
                }
              }, '修改'),
              h('Button', {
                props: {
                  type: 'ghost',
                  size: 'small'
                },
                style: {
                },
                on: {
                  click: async () => {
                    this.deleteModal = true
                    this.deleteModel = params.row
                  }
                }
              }, '删除')
            ])
          }
        }
      ],
      loading: false,
      dataRaw: [],
      data: [],
      dataViewPage: [],
      pageCurrent: 1,
      pageSize: 10,
      keyword: '',
      searchBy: '',
      searchMode: false,
      tableSearchOn: false,
      sort: null,
      searchModel: null,
      tableHeight: 500,
      detailed: false,
      detailedItem: {},
      editModal: false,
      editModel: {
        id: null,
        name: '',
        group: this.$store.getters.user.group
      },
      ruleValidate: {
        name: [
          { required: true, message: '请填写名称', trigger: 'blur' }
        ],
        group: [
          { required: true, message: '请填写店铺号码', trigger: 'blur' }
        ]
      },
      deleteModal: false,
      deleteModel: {
        id: null,
        name: ''
      }
    }
  },
  created () {
    // 如果页面刷新，则emit到Layout组件获取最新session
    if (!this.$store.getters.session) {
      this.$emit('on-checkauth', this.$route.path)
    }
  },
  mounted () {
    this.initDataTable().then(async (result) => {
      this.dataRaw = result
      this.data = result
      this.pageSize = 10
      this.pageCurrent = 1
      this.loading = false
      this.calcTableHeight()
    })
    // 监听window.resize事件
    const that = this
    window.onresize = () => {
      return (() => {
        that.calcTableHeight()
      })()
    }
  },
  watch: {
    'data': async function (newVal) {
      let vmInstance = this
      this.dataViewPage = this.data.filter(function (item, index, source) {
        return index < vmInstance.pageSize * vmInstance.pageCurrent && index >= vmInstance.pageSize * (vmInstance.pageCurrent - 1)
      })
    },
    'pageCurrent': async function (newVal) {
      let vmInstance = this
      this.dataViewPage = this.data.filter(function (item, index, source) {
        return index < vmInstance.pageSize * vmInstance.pageCurrent && index >= vmInstance.pageSize * (vmInstance.pageCurrent - 1)
      })
    },
    'pageSize': async function (newVal) {
      let vmInstance = this
      this.dataViewPage = this.data.filter(function (item, index, source) {
        return index < vmInstance.pageSize
      })
    },
    'sort': function (newVal) {
      this.refreshList()
    }
  },
  methods: {
    refreshList () {
      // refreshList()
      this.initDataTable().then(async (result) => {
        this.loading = false
        // this.searchBy = ''
        // this.keyword = ''
        // this.searchMode = false
        this.tableSearchOn = false
      })
    },
    async initDataTable () {
      this.loading = true
      this.apiItem = {
        apiHost: '',
        apiService: 'users',
        apiAction: 'listall',
        apiQuery: {}
      }
      this.apiData = {}
      if (this.sort) {
        this.apiData.sort = this.sort
      }
      if (this.searchModel) {
        this.apiData.search = this.searchModel
      }
      // TODO: 分页
      // this.apiData.limit$ = 2
      // this.apiData.skip$ = 2 * (this.pageCurrent - 1)
      this.$store.dispatch('setAPIStore', this.apiItem)
      var apiUrl = this.$store.getters.apiUrl
      return new Promise(async (resolve, reject) => {
        await this.$http.post(apiUrl, this.apiData).then(async (response) => {
          var respBody = response.data
          if (respBody.status === 'fail') {
            this.$Message.error('列表获取失败！(' + respBody.message + ')')
            reject(new Error('列表获取失败！(' + respBody.message + ')'))
          } else {
            // this.$Message.success('列表载入成功!')
            this.$store.dispatch('setAPILastResponse', respBody)
            this.dataRaw = respBody.data
            this.data = respBody.data
            this.pageSize = 10
            this.pageCurrent = 1
            resolve(respBody.data)
          }
        }).catch(err => {
          this.$store.dispatch('setAPILastResponse', err)
          reject(err)
        })
      })
    },
    sortTable (sort) {
      this.sort = sort
    },
    calcTableHeight () {
      let eleMain = document.querySelector('#main')
      let eleMainPaddingTop = window.getComputedStyle(eleMain, null).paddingTop
      let eleMainPaddingBottom = window.getComputedStyle(eleMain, null).paddingBottom
      this.tableHeight = eleMain.clientHeight - parseFloat(eleMainPaddingTop) - parseFloat(eleMainPaddingBottom) - 80
    },
    changePage (page) {
      this.pageCurrent = page
    },
    changePageSize (pageSize) {
      this.pageSize = pageSize
    },
    toggleSearchMode () {
      this.searchMode = !this.searchMode
    },
    toggleSearch (val) {
      this.tableSearchOn = val
    },
    search () {
      let cols = this.columns.filter((col, idx, ss) => {
        return col.searchable
      })
      this.data = this.dataRaw.filter((item, index, source) => {
        if (this.searchBy) {
          if (this.searchBy.indexOf('.')) {
            let thing = eval('item.' + this.searchBy)
            return thing.indexOf(this.keyword) >= 0
          } else {
            return item[this.searchBy].indexOf(this.keyword) >= 0
          }
        } else {
          let found = false
          for (let i = 0; i < cols.length; i++) {
            let col = cols[i]
            if (col.subKey) {
              let thing = eval('item.' + col.key + '.' + col.subKey)
              if (thing.indexOf(this.keyword) >= 0) {
                found = true
              }
            } else if (col.key && item[col.key] && (item[col.key].indexOf(this.keyword) >= 0)) {
              found = true
            }
          }
          return found
        }
      })
      this.pageCurrent = 0
      this.pageCurrent = 1
    },
    searchRemote () {
      let cols = this.columns.filter((col, idx, ss) => {
        return col.searchable
      })
      if (this.searchBy) {
        if (this.searchBy.indexOf('.') > -1) {
          cols = cols.filter((item) => {
            return item.key === this.searchBy.split('.')[0] && item.subKey === this.searchBy.substr(this.searchBy.indexOf('.') + 1)
          })
        } else {
          cols = cols.filter((item) => {
            return item.key === this.searchBy
          })
        }
      }
      this.searchModel = []
      cols.forEach((item) => {
        let by = {}
        let key = item.subKey ? item.key + '.' + item.subKey : item.key
        by[key] = this.keyword
        this.searchModel.push(by)
      })
      this.pageCurrent = 0
      this.pageCurrent = 1
      this.refreshList()
    },
    submitEdit () {
      this.$refs['editForm'].validate((valid) => {
        if (valid) {
          this.apiItem = {
            apiHost: '',
            apiService: 'users',
            apiAction: 'edit',
            apiQuery: {}
          }
          this.apiData = {
            id: this.editModel.id,
            name: this.editModel.name,
            group: this.editModel.group
          }
          this.$store.dispatch('setAPIStore', this.apiItem)
          var apiUrl = this.$store.getters.apiUrl
          return new Promise(async (resolve, reject) => {
            await this.$http.post(apiUrl, this.apiData).then(async (response) => {
              var respBody = response.data
              if (respBody.status === 'fail') {
                this.$Message.error('修改失败！(' + respBody.message + ')')
                reject(new Error('修改失败！(' + respBody.message + ')'))
              } else {
                this.$Message.success('店铺修改成功!')
                this.$store.dispatch('setAPILastResponse', respBody)
                this.editModal = false
                resolve(respBody.data)
              }
            }).catch(err => {
              this.$store.dispatch('setAPILastResponse', err)
              reject(err)
            })
          })
        } else {
          this.$Message.error('表单验证失败!')
        }
      })
    },
    submitDelete () {
      this.apiItem = {
        apiHost: '',
        apiService: 'users',
        apiAction: 'delete',
        apiQuery: {}
      }
      this.apiData = {
        id: this.deleteModel.id
      }
      this.$store.dispatch('setAPIStore', this.apiItem)
      var apiUrl = this.$store.getters.apiUrl
      return new Promise(async (resolve, reject) => {
        await this.$http.post(apiUrl, this.apiData).then(async (response) => {
          var respBody = response.data
          if (respBody.status === 'fail') {
            this.$Message.error('删除失败！(' + respBody.message + ')')
            reject(new Error('删除失败！(' + respBody.message + ')'))
          } else {
            this.$Message.success('店铺删除成功!')
            this.$store.dispatch('setAPILastResponse', respBody)
            this.deleteModal = false
            resolve(respBody.data)
          }
        }).catch(err => {
          this.$store.dispatch('setAPILastResponse', err)
          reject(err)
        })
      })
    }
  }
}
</script>

<style lang="less" scoped>

</style>

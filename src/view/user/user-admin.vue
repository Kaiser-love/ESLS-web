<template>
    <div style="display: flex;flex-direction: column;flex-wrap: wrap;justify-content: flex-start; align-items: center;align-content: center;" ref="container">
        <Card :bordered="false" v-bind:style="{ width: windowWidth*0.98 + 'px' }">
          <div slot="title">
            <Row type="flex" justify="center" align="middle">
                <Col span="24"><p>用户列表</p></Col>
            </Row>
          </div>
          <super_table @onClick="onTableClick" :pageSize="countPerPage" :current.sync="currentPage" @onSearch="onTableSearch" :data="userData" :columns="tableColumns" :isLoading="isTableLoading" :dataNum="userDataCount"></super_table>
        </Card>
        <Card :bordered="false" v-bind:style="{ width: windowWidth*0.98 + 'px' ,marginTop:'10px'}">
          <div slot="title">
            <Row type="flex" justify="center" align="middle">
                <Col span="22"><p>角色列表</p></Col>
                <Col span="2"><Button type="primary" @click="addRole">添加角色</Button></Col>
            </Row>
          </div>
          <Collapse v-model="selectRole" accordion>
            <Panel v-for="(item,index) in roleData" :name="item.id+''" :key="item.id">
            {{item.name}}
                <div slot="content">
                  <Transfer
                    :titles="['未获得权限','角色权限']"
                    :list-style="listStyle"
                    :data="allPerData"
                    :target-keys="getRolePerKey(index)"
                    :render-format="renderPermission"
                    @on-change="onTransferChange(index,arguments)"></Transfer>
                    <Button type="primary" v-if="hasDeleteAccess" style="margin-top:10px;" @click="delRole">删除角色</Button>
                </div>
            </Panel>
          </Collapse>
        </Card>
    </div>
</template>
<script>
import { getAllUser, switchUserUsable, deleteUser, addUserRole, delUserRole } from '@/api/user'
import { getAllRole, addPerm, delPerm, addRole, delRole } from '@/api/role'
import { getAllPermissions } from '@/api/permission'
import super_table from '@/components/table/supertable.vue'
import store from '@/store'
export default {
  components: {
    super_table
  },
  data () {
    return {
      windowWidth: 0,
      isTableLoading: false,
      countPerPage: 10,
      currentPage: 1,
      userDataCount: 0,
      userData: [],
      tableColumns: [
        {
          title: '名字',
          key: 'name',
          filter: {
            type: 'Input'
          }
        },
        {
          title: '电话',
          key: 'telephone',
          filter: {
            type: 'Input'
          }
        },
        {
          title: '地址',
          key: 'address',
          filter: {
            type: 'Input'
          }
        },
        {
          title: '部门',
          key: 'department',
          filter: {
            type: 'Input'
          }
        },
        {
          title: '角色',
          key: 'roleList',
          width: '200',
          render: (h, params) => {
            let currentRole = []
            let EditAccess = store.getters.access.indexOf(2) === -1
            for (let i = 0; i < params.row.roleList.split(' ').length; ++i) {
              currentRole.push(parseInt(params.row.roleList.split(' ')[i]))
            }
            return h('Select', {
              props: {
                multiple: true,
                value: currentRole,
                disabled: EditAccess
              },
              attrs: {
                style: 'padding-left:10px;padding-right:10px;text-align:left;'
              },
              on: {
                'on-change': (val) => {
                  this.onUpdateRole(params.row, val, params.row.roleList.split(' ').length - 1)
                }
              }
            }, this.roleData.map((item) => {
              return h('Option', {
                props: {
                  value: item.id,
                  label: item.name
                }
              })
            })
            )
          }
        },
        {
          title: '商店',
          key: 'shop',
          filter: {
            type: 'Input'
          }
        },
        {
          title: '状态',
          key: 'status',
          render: (h, params) => {
            const row = params.row
            const isUsable = row.status === 1
            let StatusAccess = store.getters.access.indexOf(12) === -1
            return h('i-switch', {
              props: {
                value: isUsable,
                size: 'large',
                disabled: StatusAccess
              },
              on: {
                'on-change': (val) => {
                  if (val) {
                    params.row.status = 1
                  } else {
                    params.row.status = 0
                  }
                  switchUserUsable(params.row.id).then(res => {
                  })
                }
              }
            }, [
              h('span', {
                slot: 'open'
              }, '启用'),
              h('span', {
                slot: 'close'
              }, '禁用')
            ])
          }
        },
        {
          title: '操作',
          key: 'action',
          align: 'center',
          width: '150',
          render: (h, params) => {
            let DeleteAccess = store.getters.access.indexOf(10) === -1
            return h('div', [
              h('Button', {
                props: {
                  type: 'error',
                  size: 'small'
                },
                style: {
                  margin: '2px',
                  display: DeleteAccess ? 'none' : 'inline-block'
                },
                on: {
                  'click': (event) => {
                    event.stopPropagation()
                    this.remove(params.row.id)
                  }
                }
              }, '删除')
            ])
          }
        }
      ],
      roleData: [],
      selectRole: '',
      allPerData: [],
      listStyle: {
        width: '250px',
        height: '200px'
      },
      addRoleName: '',
      addRoleType: ''
    }
  },
  created () {
    this.getUserTableData({ page: 0, count: this.countPerPage })
    this.getRoleList()
    this.getPermissionData()
  },
  mounted () {
    var that = this
    this.$nextTick(() => {
      this.windowWidth = this.$refs.container.offsetWidth
    })
    window.onresize = function () {
      that.windowWidth = that.$refs.container.offsetWidth
    }
  },
  watch: {
    currentPage () {
      this.getUserTableData({ page: this.currentPage - 1, count: this.countPerPage })
    }
  },
  computed: {
    hasEditAccess: () => {
      return store.getters.access.indexOf(2) !== -1
    },
    hasDeleteAccess: () => {
      return store.getters.access.indexOf(10) !== -1
    }
  },
  methods: {
    onUpdateRole (row, val, curlength) {
      console.log(val)
      console.log(curlength)
      if (curlength < val.length) {
        addUserRole(row.id, val).then(res => {
          this.getUserTableData({ page: this.currentPage - 1, count: this.countPerPage })
          this.$Message.info('添加角色成功')
        })
      } else if (val.length < curlength) {
        let rmRole = row.roleList.split(' ').filter((item) => {
          if (item === '') return false
          return val.indexOf(parseInt(item)) === -1
        })
        rmRole = rmRole.map((item) => { return parseInt(item) })
        delUserRole(row.id, rmRole).then(res => {
          this.$Message.info('删除角色成功')
          this.getUserTableData({ page: this.currentPage - 1, count: this.countPerPage })
        })
      }
    },
    addRole () {
      this.$Modal.confirm({
        title: '新角色信息',
        render: (h, params) => {
          var that = this
          return h('span', [
            h('Input', {
              prop: {
                placeholder: '输入名字',
                value: this.addRoleName
              },
              on: {
                'on-change': (event) => {
                  that.addRoleName = event.target.value
                }
              }
            }),
            h('Select', {
              props: {
                value: this.addRoleType
              },
              attrs: {
                style: 'margin-top:10px'
              },
              on: {
                'on-change': function (val) {
                  that.addRoleType = val
                }
              }
            }, [
              h('Option', {
                props: {
                  value: '最高权限',
                  label: '最高权限'
                }
              }),
              h('Option', {
                props: {
                  value: '中级权限',
                  label: '中级权限'
                }
              }),
              h('Option', {
                props: {
                  value: '低级权限',
                  label: '低级权限'
                }
              })
            ])
          ])
        },
        onOk: () => {
          addRole(this.addRoleName, this.addRoleType).then(res => {
            this.$Message.info('添加角色成功')
            this.getRoleList()
          })
        }
      })
    },
    delRole () {
      delRole(this.selectRole).then(
        this.getRoleList()
      )
    },
    getPermissionData () {
      getAllPermissions().then(res => {
        const data = res.data.data
        for (let i = 0; i < data.length; ++i) {
          data[i].id = data[i].id + ''
          Object.assign(data[i], { key: data[i].id })
        }
        this.allPerData = data
      })
    },
    onTransferChange (index, argu) {
      let newTargetKeys = argu[2]
      let direction = argu[1]
      if (direction === 'left') {
        let params = { collectionIds: [], ids: [] }
        let cIds = []
        for (let i = 0; i < newTargetKeys.length; ++i) {
          cIds.push(parseInt(newTargetKeys[i]))
        }
        params.collectionIds.push(cIds)
        params.ids.push(parseInt(this.selectRole[0]))
        delPerm(params)
        this.roleData[index].permissions = this.roleData[index].permissions.filter((item) => {
          console.log(newTargetKeys)
          return newTargetKeys.indexOf(item.id + '') === -1
        })
      } else if (direction === 'right') {
        let params = { collectionIds: [], ids: [] }
        let cIds = []
        for (let i = 0; i < newTargetKeys.length; ++i) {
          if (this.roleData[index].permissions.indexOf(newTargetKeys[i]) === -1) {
            cIds.push(parseInt(newTargetKeys[i]))
          }
        }
        params.collectionIds.push(cIds)
        params.ids.push(parseInt(this.selectRole[0]))
        addPerm(params)
        let addPer = this.allPerData.filter((item) => {
          return newTargetKeys.indexOf(item.key) !== -1
        })
        this.roleData[index].permissions = this.roleData[index].permissions.concat(addPer)
      }
    },
    getRolePerKey (index) {
      let res = []
      this.roleData[index].permissions.map((item) => {
        res.push(item.id + '')
      })
      return res
    },
    renderPermission (item) {
      return item.name
    },
    onTableClick (currentRow) {

    },
    getRoleList () {
      getAllRole().then(res => {
        this.roleData = res.data.data
      })
    },
    onTableSearch (search) {
      var key = Object.keys(search)
      if (key.length === 0) {
        this.getUserTableData({ page: 0, count: this.countPerPage })
        this.currentTagPage = 1
        return
      }
      var value = search[key[0]]
      this.getUserTableData({ query: key[0], queryString: value })
    },
    getUserTableData ({ page, count, query, queryString }) {
      var that = this
      that.isTableLoading = true
      getAllUser({ page: page, count: count, query: query, queryString: queryString }).then(res => {
        that.userData = res.data.data
        that.userDataCount = res.data.code
        that.isTableLoading = false
      })
    },
    remove (id) {
      var that = this
      this.$Modal.confirm({
        title: '警告',
        content: '确定删除该用户吗？',
        onOk: function () {
          deleteUser(id)
            .then(() => {
              that.getUserTableData({ page: that.currentPage - 1, count: that.countPerPage })
            })
        }
      })
    }
  }

}
</script>

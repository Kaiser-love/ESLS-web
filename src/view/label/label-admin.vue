<template>
  <div class="container" ref="container" :style="{flexDirection: 'row',alignItems: 'flex-start'}">
      <div class="left" v-bind:style="{ width: windowWidth*0.6 + 'px' }">
        <Card :bordered="false" class="e-lable-table-card card">
            <div slot="title">
                <Row type="flex" justify="center" >
                    <Col span="18"><p>样式信息</p></Col>
                    <Col span="2" v-if="hasEditAccess"><Button type="primary" @click="addStyle">新建样式</Button></Col>
                    <Col span="2" v-if="hasEditAccess">
                      <Upload style="margin-left:4px" :show-upload-list="false" action="" :before-upload="inputStyle">
                        <Button icon="ios-cloud-upload-outline">上传样式</Button>
                    </Upload>
                  </Col>
                </Row>
              </div>
            <super_table :pageSize="countPerPage" :current.sync="currentPage" :dataNum="dataNum" class="e-label-table" @onSearch="onTableSearch" @onClick="onTableClick" :data="styleData" :columns="tableColumns" :isLoading="isTableLoading" :pageNum="dataNum"></super_table>
        </Card>
        <Modal
            :style="{flexDirection: 'row'}"
            v-model="isModal"
            @on-ok="onUpdate"
            width="auto"
            title="样式编辑器"
            class-name="modal-style-designer">
            <modal_style_designer ref="designer" @reloadTable="reload" @onSava="isModal=false"></modal_style_designer>
        </Modal>
      </div>
      <div class="right" v-bind:style="{ marginLeft:'10px'}" >
        <div v-bind:style="{ width: windowWidth*0.3 + 'px',display:'flex',flexDirection: 'column', alignItems: 'flex-start', justifyContent: 'flex-start'}">
          <Card :bordered="false" class="e-lable-card card">
            <p slot="title">价签样式预览</p>
            <div>
              <e_label class="e-label" v-bind="item" ref="label_canvas" >
                  <Spin size="large" fix v-if="isLabelLoading"></Spin>
              </e_label>
            </div>
          </Card>
          <!-- <Card :bordered="false" class="card input-card">
            <div slot="title">
                <Row type="flex" justify="start" align="middle">
                    <Col span="21"><p>信息修改</p></Col>
                    <Col span="3"><Button type="primary">保存</Button></Col>
                </Row>
            </div>
            <div>
              <Input type="text" v-model="item.itemName" />
              <Input type="text" v-model="item.itemUnit" />
              <Input type="text" v-model="item.itemNorm" />
              <Input type="text" v-model="item.itemCategory" />
              <Input type="text" v-model="item.itemOrigin" />
              <Input type="text" v-model="item.itemNo" />
              <Input type="text" v-model="item.itemQRCode" />
              <Input type="text" v-model="item.itemBarCode" />
              <Input type="text" v-model="item.itemStock" />
              <Input type="text" v-model="item.itemPrice" />
              <Input type="text" v-model="item.itemOnSalePrice" />
            </div>
          </Card> -->
        </div>
      </div>
  </div>
</template>

<script>
import e_label from '@/components/e-label/e-lable.vue'
import super_table from '@/components/table/supertable.vue'
import modal_style_designer from '@/components/modal/modal-style-designer.vue'
import { getStyle, getAllStyle, deleteStyle, updateStyle, newStyle } from '@/api/style'
import FileSaver from 'file-saver'
import store from '@/store'
export default {
  components: {
    e_label,
    super_table,
    modal_style_designer
  },
  data () {
    return {
      windowWidth: 0,
      item: {
        itemName: '测试商品名称1',
        itemUnit: '罐',
        itemNorm: '205g',
        itemCategory: '衣物',
        itemOrigin: '北京',
        itemNo: '00012',
        itemQRCode: '692226641428',
        itemBarCode: '692226641428',
        itemStock: '110',
        itemisOnSale: true,
        itemPrice: '10.19',
        itemOnSalePrice: '444.44',
        labelStyle: '1'
      },
      styleList: [
        '1',
        '2',
        '3',
        '4',
        '5',
        '6'
      ],
      isLabelLoading: false,
      isTableLoading: false,
      isModal: false,
      dataNum: 0,
      currentPage: 1,
      countPerPage: 20,
      styleData: [],
      tableColumns: [
        {
          type: 'index',
          width: 60,
          align: 'center',
          indexMethod: (row) => {
            return row._index + 1 + (this.currentPage - 1) * this.countPerPage
          }
        },
        {
          title: '样式id',
          key: 'styleNumber',
          filter: {
            type: 'Input'
          }
        },
        {
          title: '样式名称',
          key: 'styleType',
          filter: {
            type: 'Input'
          }
        },
        {
          title: '宽度',
          key: 'width',
          filter: {
            type: 'Input'
          }
        },
        {
          title: '高度',
          key: 'height',
          filter: {
            type: 'Input'
          }
        },
        {
          title: '促销',
          width: '80',
          render: (h, params) => {
            let temp = this.styleData.find(function (item) { return item.styleNumber === params.row.styleNumber })
            return h('i-switch', {
              props: {
                value: params.row.isPromote === 1
              },
              on: {
                'on-change': (val, $event) => {
                  event.stopPropagation()
                  if (val) {
                    temp.isPromote = 1
                  } else {
                    temp.isPromote = 0
                  }
                  this.onTableClick(temp)
                }
              }
            })
          }
        },
        {
          title: '操作',
          key: 'action',
          width: '200',
          align: 'center',
          render: (h, params) => {
            let isAccess = store.getters.access.indexOf(2) === -1
            let DeleteAccess = store.getters.access.indexOf(10) === -1
            let EditAccess = store.getters.access.indexOf(18) === -1
            return h('div', [
              h('Button', {
                props: {
                  type: 'primary',
                  size: 'small'
                },
                style: {
                  margin: '2px',
                  display: isAccess ? 'none' : 'inline-block'
                },
                on: {
                  'click': (event) => {
                    event.stopPropagation()
                    let temp = this.styleData.find(function (item) { return item.styleNumber === params.row.styleNumber })
                    if (params.row.isPromote === 0) {
                      this.editStyle(temp.id, temp.styleType, temp.width, temp.height)
                    } else {
                      this.editStyle(temp.id + 1, temp.styleType, temp.width, temp.height)
                    }
                  }
                }
              }, '修改'),
              h('Button', {
                props: {
                  type: 'primary',
                  size: 'small'
                },
                style: {
                  margin: '2px',
                  display: isAccess ? 'none' : 'inline-block'
                },
                on: {
                  'click': (event) => {
                    event.stopPropagation()
                    let temp = this.styleData.find(function (item) { return item.styleNumber === params.row.styleNumber })
                    this.exportStyle(temp.id, temp.styleType)
                  }
                }
              }, '导出'),
              h('Button', {
                props: {
                  type: 'error',
                  size: 'small'
                },
                style: {
                  margin: '2px',
                  display: DeleteAccess || EditAccess ? 'none' : 'inline-block'
                },
                on: {
                  'click': (event) => {
                    event.stopPropagation()
                    let temp = this.styleData.find(function (item) { return item.styleNumber === params.row.styleNumber })
                    this.remove(temp.id)
                  }
                }
              }, '删除')
            ])
          }
        }
      ],
      currentStyleID: 0
    }
  },
  created () {
    this.getStyleTableData({ page: this.currentPage - 1, count: this.countPerPage })
  },
  watch: {
    currentPage () {
      this.getStyleTableData({ page: this.currentPage - 1, count: this.countPerPage })
    }
  },
  computed: {
    hasEditAccess: () => {
      return store.getters.access.indexOf(2) !== -1
    }
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
  methods: {
    inputStyle (data) {
      let reader = new FileReader()
      let str = ''
      var that = this
      reader.addEventListener('load', function (e) {
        str = e.target.result
        let styledes = data.name.split('.json')[0]
        let styleDisp = JSON.parse(str)
        let index = 1
        for (let i = 0; i < styleDisp.length; ++i) {
          if (styleDisp[i].status === 1) {
            that.$set(styleDisp[i], 'regionId', index++)
          } else {
            that.$set(styleDisp[i], 'regionId', 0)
          }
          delete styleDisp[i].id
        }
        newStyle(styledes).then(res => {
          const newId = res.data.data.id
          updateStyle(newId, styleDisp, 0, 0).then(res => {
            that.$Message.info('导入样式成功')
            that.reload()
          })
        })
      })
      reader.readAsText(data)
      return false
    },
    exportStyle (id, name) {
      getStyle(id).then(res => {
        let str = JSON.stringify(res.data.data)
        FileSaver.saveAs(new Blob([str], { type: 'text/plain;charset=utf-8' }), name + '.json')
      })
    },
    getLabelData (id, w, h) {
      var that = this
      that.isLabelLoading = true
      getStyle(id).then(res => {
        const dispData = res.data.data
        var len = dispData.length // 循环变量
        for (var i = 0; i < len; ++i) {
          if (dispData[i].sourceColumn === 'name') {
            that.item.itemName = dispData[i].text
          } else if (dispData[i].sourceColumn === 'price') {
            that.item.itemPrice = dispData[i].text
          } else if (dispData[i].sourceColumn === 'promotePrice') {
            that.item.itemOnSalePrice = dispData[i].text
          } else if (dispData[i].sourceColumn === 'spec') {
            that.item.itemNorm = dispData[i].text
          } else if (dispData[i].sourceColumn === 'class') {
            that.item.itemCategory = dispData[i].text
          } else if (dispData[i].sourceColumn === 'unit') {
            that.item.itemUnit = dispData[i].text
          } else if (dispData[i].sourceColumn === 'origin') {
            that.item.itemOrigin = dispData[i].text
          } else if (dispData[i].sourceColumn === 'shelfNumber') {
            that.item.itemNo = dispData[i].text
          } else if (dispData[i].sourceColumn === 'qrCode') {
            that.item.itemQRCode = dispData[i].text
          } else if (dispData[i].sourceColumn === 'barCode') {
            that.item.itemBarCode = dispData[i].text
          } else if (dispData[i].sourceColumn === 'stock') {
            that.item.itemStock = dispData[i].text
          } else {

          }
        }
        that.isLabelLoading = false
        that.$refs.label_canvas.initData(dispData, w, h, null)
      })
    },
    getStyleTableData (page, count) {
      var that = this
      that.isTableLoading = true
      getAllStyle(page, count).then(res => {
        const data = res.data.data.filter((item) => {
          return item.id % 2 === 0
        })
        that.dataNum = res.data.code
        that.styleData = data
        that.isTableLoading = false
      })
    },
    reload () {
      this.getStyleTableData({ page: this.currentPage - 1, count: this.countPerPage })
    },
    onTableSearch (search) {
      var key = Object.keys(search)
      if (key.length === 0) {
        this.getStyleTableData({ page: 0, count: this.countPerPage })
        this.currentTagPage = 1
        return
      }
      var value = search[key]
      this.getStyleTableData({ queryId: key[0], queryString: value })
    },
    onTableClick (currentRow) {
      if (currentRow.isPromote === 0) {
        this.getLabelData(currentRow.id, currentRow.width, currentRow.height)
      } else {
        this.getLabelData(currentRow.id + 1, currentRow.width, currentRow.height)
      }
    },
    remove (id) {
      var that = this
      this.$Modal.confirm({
        title: '警告',
        content: '确定删除该样式吗？',
        onOk: function () {
          deleteStyle(id)
            .then(() => {
              deleteStyle(id + 1).then(() => {
                that.getStyleTableData({ page: that.currentPage - 1, count: that.countPerPage })
              })
            })
        }
      })
    },
    editStyle (styleid, styletype, w, h) {
      console.log(styleid)
      this.currentStyleID = styleid
      this.isModal = true
      this.$refs.designer.getStyleData(styleid, styletype, w, h)
    },
    onUpdate () {
      this.$refs.designer.update(this.currentStyleID)
    },
    addStyle () {
      this.$Modal.confirm({
        title: '新样式信息',
        render: (h, params) => {
          var that = this
          return h('span', [
            h('p', '样式大小:'),
            h('Select', {
              props: {
                value: this.newStyleSize
              },
              on: {
                'on-change': (val) => {
                  that.newStyleSize = val
                }
              }
            }, [
              h('Option', {
                props: {
                  value: '4.2寸',
                  label: '4.2寸'
                }
              }),
              h('Option', {
                props: {
                  value: '2.13寸',
                  label: '2.13寸'
                }
              }),
              h('Option', {
                props: {
                  value: '2.9寸',
                  label: '2.9寸'
                }
              })
            ]),
            h('p', {
              attrs: {
                style: 'margin-top:10px'
              }
            }, '样式名字:'),
            h('Input', {
              props: {
                placeholder: '输入名字',
                value: this.newStyleName
              },
              on: {
                'on-change': (event) => {
                  that.newStyleName = event.target.value
                }
              }
            }),
            h('p', {
              attrs: {
                style: 'margin-top:10px'
              }
            }, '样式类型:'),
            h('Select', {
              props: {
                value: this.newStyleType
              },
              on: {
                'on-change': function (val) {
                  that.newStyleType = val
                }
              }
            }, [
              h('Option', {
                props: {
                  value: '黑白',
                  label: '黑白'
                }
              }),
              h('Option', {
                props: {
                  value: '三色',
                  label: '三色'
                }
              })
            ])
          ])
        },
        onOk: () => {
          let w = 0
          let h = 0
          let styleid = 0
          let styleType = this.newStyleSize + '-' + this.newStyleName + '-' + this.newStyleType
          if (this.newStyleSize === '4.2寸') {
            w = 400
            h = 300
          } else if (this.newStyleSize === '2.13寸') {
            if (this.styleType === '黑白') {
              w = 250
              h = 122
            } else if (this.styleType === '三色') {
              w = 212
              h = 104
            }
          } else if (this.newStyleSize === '2.9寸') {
            w = 296
            h = 128
          }
          this.isModal = true
          this.$refs.designer.getStyleData(styleid, styleType, w, h, 'new')
        }

      })
    }
  }
}
</script>
<style lang="less">
Input{
  margin: 2px
}
.card{
  margin: 10px;
}
.container{
    display: flex;
    flex-wrap: nowrap;
    justify-content:center;
    align-items: flex-start;

}
.left{
  display: flex;
  flex-direction: column;
  flex-wrap: wrap;
  justify-content: flex-start;
  align-items: flex-start;
}
.right{
  display: flex;
  flex-direction: column;
  flex-wrap: wrap;
  justify-content: flex-start;
  align-items: flex-start;
}
.e-label{
  width: 100%;
  height: auto;
}
.e-label-table{
  width: 100%;
}
.e-lable-card{
    width: auto;
    height: auto;
}
.e-lable-table-card{
    height: auto;
}
.input-card{
  width: 432px;
  height: auto;
}
.modal-style-designer{
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: center;

  .ivu-modal{
      top: 0;
  }
}
</style>

<style lang="less">

</style>

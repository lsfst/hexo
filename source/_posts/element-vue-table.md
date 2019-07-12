---
title: ElementUI + vue 定制一个表格插件
date: 2018-11-011 10:01:22
tags: [vue]
categories: vue
---

&emsp;&emsp;公司后台管理系统有很多表格页面，分别对应数据库不同的表。整个后台UI使用的是Element UI，Element UI的表格功能很强大，但是如果拿来直接用，会产生大量的模板代码，再加上各种个性化功能和自定义配置，一个一个写起来既麻烦又难看。
&emsp;&emsp;正是基于此才需要对表格进行封装，希望达到的效果如下:
{% codeblock  %}
tableList: [
    {name: 'keyID', width: {length: 100},sort:true,show: true},
    {name: 'Type',show:true,width:{length:150},format:{comData:cdrTypeKeyValue}},
    {name: 'dataTraffic',format:{trafficFM:true}, sort:true,type:"tag",tagData:function(value){
        return value> 0 ? (value<1000000?"info":"warning") : "danger";
    },width: {length: 160},show: true},
    {name: 'crtTm', show: true,type:'date',width: {length: 200}},
]
{% endcodeblock %}

在创建一个新的表格页面时，只需要利用如上的表格字段配置就可以完成渲染，增删查改，排序，自定义样式等大部分工作。
当然，这种模板是比较特性化的，主要根据具体业务而定。

### 表格组件
{% codeblock simpleTableTemplate.vue  lang:js %}

<template xmlns:v-bind="http://www.w3.org/1999/xhtml">
    <div>
    <el-table
            v-loading="listLoading"
            element-loading-text="拼命加载中"
            element-loading-background="rgba(0, 0, 0, 0.8)"
            ref="elTable"
            :data="computedLists"
            @select="selectionChange"
            @select-all="selectionAll"
            @row-click="clickRow"
            max-height="700"
            highlight-current-row
            style="width: 100%;"
            :row-class-name="tableRowClassName"
    >

  <el-table-column
                v-if="check"
                fixed="left"
                :prop="tableKey"
                type="selection"
                width="35">
        </el-table-column>

        <template v-for="item in columnList">
        
            <el-table-column v-if="item.sort"  :width="item.width.length" align="center" :label="$t(prefix+item.name)" :render-header="(h,obj,index) => renderSort(h,obj,index,item.name)">
                <template slot-scope="scope">
                    <i v-if="item.type=='date'" class="el-icon-time"></i>
                    <span v-if="item.type=='html'" v-html="scope.row.showData[item.name]"></span>
                    <el-tooltip v-else-if="item.type=='tip'" placement="top">
                        <div slot="content" v-html="scope.row.showData.tips[item.name]"></div>
                        <span v-html="scope.row.showData[item.name]"></span>
                    </el-tooltip>
                    <span v-else-if="item.type=='date'" v-text="scope.row.showData[item.name]"></span>
                    <el-tag v-else-if="item.type=='tag'" :type="scope.row.showData.tags[item.name]" :style="item.style">{{ scope.row.showData[item.name]}}</el-tag>
                    <span v-else  v-text="scope.row.showData[item.name]"></span>
                </template>
            </el-table-column>
            <el-table-column v-else  :width="item.width.length" align="center" :label="$t(prefix+item.name)">
                <template slot-scope="scope">
                    <i v-if="item.type=='date'" class="el-icon-time"></i>
                    <span v-if="item.type=='html'" v-html="scope.row.showData[item.name]"></span>
                    <el-tooltip v-else-if="item.type=='tip'" placement="top">
                        <div slot="content" v-html="scope.row.showData.tips[item.name]"></div>
                        <span v-html="scope.row.showData[item.name]"></span>
                    </el-tooltip>
                    <span v-else-if="item.type=='date'" v-text="scope.row.showData[item.name]"></span>
                    <el-tag v-else-if="item.type=='tag'" :type="scope.row.showData.tags[item.name]" :style="item.style">{{ scope.row.showData[item.name]}}</el-tag>
                    <span v-else  v-text="scope.row.showData[item.name]"></span>
                </template>
            </el-table-column>
        </template>

        <el-table-column type="expand" fixed="right" v-if="expand">
            <template slot-scope="props">
                <el-form label-position="left" inline class="demo-table-expand">
                    <el-form-item v-for="item in expandList" :key="item.name" :label="$t(prefix+item.name)" :style="item.style">
                        <span v-html="props.row.showData[item.name]"></span>
                    </el-form-item>
                </el-form>
            </template>
        </el-table-column>
      <!--expand扩展框支持-->
      
      </el-table>
          </div>
</template>


<script>
    /**
     * @author lsf
     * @Date 2018/10/8 18:13
     * @Description 
     * expand 在点击展开的时候才会触发，之后值会缓存起来
     *
     * 一个完整的table的参数列表：
     *              param                        required           desc
     *   :list-loading="listLoading"
     *   :table-key="tableKey"                       Y              主键
     *   :expand="expand"                            N              扩展框
     *   :check="check"                              N              checkbox
     *   :prefix="prefix"                            Y
     *   :table-list="tableList"                     Y              表格配置列表
     *   :computed-lists="computedLists"             Y              处理后的查询数据
     *   @tableRowClassName="tableRowClassName"      Y              row样式
     *   @renderSort="renderSort"                    Y              搜索表头
     *   @selectionAll="selectionAll"                N              全选事件
     *   @selectionChange="selectionChange"          N              选择事件
     *
     */
    export default {
        name:"simpleTableTemplate",
        props:['tableKey','listLoading','expand','prefix','check','tableList','computedLists'],
        computed: {
            //从tableList提取出expandList（expand列表），columnList(表格主体)
            columnList: function () {
                let columnList=[];
                let tableList = this.tableList;
                for (let i = 0; i < tableList.length; i++) {
                    if(!tableList[i].expand && !tableList[i].hide){
                        columnList.push(tableList[i])
                    }
                }
                return columnList
            },
            expandList: function () {
                let expandList=[];
                let tableList = this.tableList;
                for (let i = 0; i < tableList.length; i++) {
                    if(tableList[i].expand){
                        expandList.push(tableList[i])
                    }
                }
                return expandList
            },
        },
        methods:{
            renderSort(h,obj,index,name){ //排序
                return this.$parent.renderSort(h,obj,index,name)
            },
            tableRowClassName({row, rowIndex}){ //row样式
                return this.$parent.tableRowClassName({row, rowIndex})
            },
            selectionAll(val,row){    //checkbox全选
                this.$emit('selectionAll',val, row)
            },
            selectionChange(val,row){
                this.$emit('selectionChange',val, row)
            },
            clickRow(row,event,column){  //点击行事件
                this.$refs.elTable.toggleRowSelection(row,true;
                if(this.expand){
                    this.$refs.elTable.toggleRowExpansion(row)
                }
                this.$emit('clickRow',row)
            }
        },
    };
</script> 

{% endcodeblock %}

### 操作栏组件  
&emsp;&emsp;除了表格主体外，还需要操作栏，比如增删查改下载等，所以还需要用按钮写一个简单的操作栏组件  
{% codeblock operationsTemplate.vue lang:js %} 
  <template>
      <div>
          <el-button class="filter-item" style="margin-left: 10px;" type="primary" icon="el-icon-refresh"
                     @click="handleRefresh">{{ ('refresh') }}
          </el-button>
          <template  v-for="item in operations">
              <el-button v-if="item=='add'" class="filter-item" style="margin-left: 10px;" type="primary" icon="el-icon-plus"
                         @click="handleCreate">{{ ('add') }}
              </el-button>
              <el-button v-if="item=='edit'" class="filter-item" style="margin-left: 10px;" type="primary" icon="el-icon-edit" :disabled="editDisable"
                          @click="handleUpdate">{{ ('edit') }}
              </el-button>
              <el-button v-if="item=='remove'" class="filter-item" style="margin-left: 10px;" type="danger" icon="el-icon-delete" :disabled="deleteDisable"
                          @click="handleDelete">{{ ('delete') }}
              </el-button>
              <el-button v-if="item=='download'" v-waves :loading="downloadLoading" class="filter-item" type="primary" icon="el-icon-download"
                         @click="handleDownload">{{ ('export') }}
              </el-button>
          </template>
      </div>
  </template>
  
  
  <script> 
      //操作栏：刷新查询（所有的表格都有），新增，编辑，修改，删除，导出
      export default {
          name:"operationsTemplate",
          props:['operations','downloadLoading','deleteDisable','editDisable'],
          methods:{
              handleRefresh(){
                  this.$emit('handleRefresh')
              },
              handleCreate(){
                  this.$emit('handleCreate')
              },
              handleUpdate(){
                  this.$emit('handleUpdate')
              },
              handleDelete(){
                  this.$emit('handleDelete')
              },
              handleDownload(){
                  this.$emit('handleDownload')
              }
          }
      }
  </script>

{% endcodeblock %}

### 组成实例
{% codeblock  lang:js %}
<template>
    <div class="app-container">

        <div class="filter-container">
                <el-col :span="6">
                    <el-input :placeholder="$t('table.vid')" v-model="listQuery.vid"
                              clearable
                              prefix-icon="el-icon-search" style="width: 90%;" class="filter-item"/>
                </el-col>

               <el-col :span="6">
                    <el-select v-model="listQuery.area" class="filter-item"
                               :placeholder="$t('table.area')" filterable remote
                               style="width:90%" clearable
                               @focus="switchArea" :remote-method="remoteAreaMethod"
                               :loading="selectLoading"
                               clearable>
                        <el-option v-for="item in areaOptions" :key="item.value" :label="item.label"
                                   :value="item.value"/>
                    </el-select>
                </el-col>

                <operationsTemplate
                        :operations="operations"
                        :download-loading="downloadLoading"
                        @handleDownload="handleDownload"
                        @handleRefresh="handleRefresh"
                ></operationsTemplate>
        </div>
        <div>

            <simpleTableTemplate
                    :list-loading="listLoading"
                    :table-key="tableKey"
                    :expand="expand"
                    :check="check"
                    :prefix="prefix"
                    :table-list="tableList"
                    :computed-lists="computedLists"
                    @tableRowClassName="tableRowClassName"
                    @renderSort="renderSort"
                    @selectionAll="selectionAll"
                    @selectionChange="selectionChange"
            ></simpleTableTemplate>

            <div class="pagination-container">
                <el-pagination :current-page="listQuery.page" :page-sizes="[10,20,30, 50]" :page-size="listQuery.limit"
                               :total="total" background layout="total, sizes, prev, pager, next, jumper"
                               @size-change="handleSizeChange" @current-change="handleCurrentChange"/>
            </div>
        </div>

    </div>
</template>


<script>
    import simpleTableTemplate from '@/views/table/simpleTableTemplate'
    import operationsTemplate from '@/views/table/operationsTemplate'

    //import具体方法时，必须{}包裹
    import {fetch} from '@/api/cdr'
    import {areaS2} from '@/api/select'
    import {renderHeadSort, getList, selectionChange, selectionAll,
    debounce,computedFmt,remoteQuery,remoteInitQuery,handleDownload
    } from '@/utils/tableCustom'

    const cdrTypeOptions = [
        {key: 'N', name: '在线呼叫'},
        {key: 'C', name: '落地呼叫'},
        {key: 'B', name: '落地回拨'}
    ]

    const cdrTypeKeyValue = cdrTypeOptions.reduce((acc, cur) => {
        acc[cur.key] = cur.name
        return acc
    }, {})

    export default {
        name: 'cdrTable',
        components: {simpleTableTemplate,operationsTemplate},
        data() {
            return {
                prefix:'table.tbCDR.',
                tableKey: 'keyCDRID',
                tableList: [
                    {name: 'keyCDRID', width: {length: 100},sort:true,show: true},
                    {name: 'cdrType',show:true,width:{length:150},format:{comData:cdrTypeKeyValue}},
                    {name: 'dataTraffic',format:{trafficFM:true}, sort:true,type:"tag",
                        tagData:function(value){
                            return value> 0 ? (value<1000000?"info":"warning") : "danger";
                        },width: {length: 160},show: true},
                    {name: 'area',  show: true,width: {length: 200},type:'html',format:{areaFM:true}},
                    {name: 'crtTm', show: true,type:'date',width: {length: 200}},
                    {name: 'crtBy',  expand: true},
                ],
                check: false,
                expand: true,
                operations: ['download'],    //操作栏操作配置
                list: [],                   //API接口获取的原始数据
                total: null,
                listLoading: true,
                listQuery: {                //查询参数：分页，范围，排序以及其他参数
                    page: 1,
                    offset: 0,
                    limit: 20,
                    orderList: []
                },
                areaOptions:[],              //用户输入条件实时查询的结果存进areaOptions（默认20条）
                areaInitOptions:[],          //为了减少element remote select的查询次数，选择将remote select初始（无条件）查询结果存进areaInitOptions
                downloadLoading: false,
                selectLoading: false,
                lastDebounce: undefined,     //节流函数：对查询频率进行限制
                lastModifyTm: undefined,
            }
        },
        created() {
            //查询并更新查询时间
            this.getList()
            this.lastModifyTm = (new Date()).getTime()
        },
        computed: {
            //对查询的原始数据进行处理成实际展示的文本
            computedLists: function () {
                return computedFmt.call(this,this.list,this.tableList)
            }
        },
        mounted(){
            //remote select初次查询
            remoteInitQuery.call(this, areaS2,"area","area")
        },
        watch: {
            //watch监听搜索区用户输入，只要有变化则自动触发查询，这样子比较简单，但是这样有个小问题：搜索区有input框，select，remote select以及时间栏，
            //对于input框，用户每输入一个字符就会触发一次查询，这样子会导致访问次数太高，浪费网络资源。所以需要使用节流函数限制访问频率：具体设置是两秒触发一次查询
            listQuery: {
                handler(curVal, oldVal){
                    debounce.call(this, 2000, function () {
                        this.getList()
                    })
                },
                deep: true
            },
        },
        methods: {
            getList() {
                getList.call(this, fetch)
            },
            handleSizeChange(val) {
                this.listQuery.limit = val
                this.listQuery.offset = (this.listQuery.page - 1) * this.listQuery.limit
                this.getList()
            },
            handleCurrentChange(val) {
                this.listQuery.page = val
                this.listQuery.offset = (this.listQuery.page - 1) * this.listQuery.limit
                this.getList()
            },
            //重置搜索条件
            handleRefresh() {
                this.listQuery = {
                    page: 1,
                    offset: 0,
                    limit: 20,
                    orderList: []
                }
            },
            //数据导出
            handleDownload() {
                const tHeader = this.tableList.map(function (value) {
                    return value.name;
                })
                handleDownload.call(this, tHeader, fetch)
            },
            renderSort(h, {column, $index}, index, name) {
                return renderHeadSort.call(this, h, {column, $index}, index, name)
            },
            //地区select框的remote查询
            remoteAreaMethod(query){
                remoteQuery.call(this, areaS2,query, "area","area")
            },
            //select原始查询结果切换
            switchArea(){
                this.areaOptions = this.areaInitOptions
            },
            selectionChange(val, row){
                this.checkedRows = val;
            },
            selectionAll(val, row){
                this.checkedRows = val;
            },
            tableRowClassName({row, rowIndex}) {
                    return '';
            },
        }
    }
</script>

{% endcodeblock %}


### 说明
## 1.API
&emsp;&emsp;表格操作和select remote查询的api我习惯分别写在不同的的js文件中，这样方便管理，在具体需要使用时单独引用。
&emsp;&emsp;查询的request使用的是axios，进行了单独封装，这里就不贴代码了。
{% codeblock cdr.js lang:js %}
import request from '@/utils/request'
import qs from 'qs'

export function fetch(query) {
    //直接对query操作会影响全局query
    let _query=JSON.parse(JSON.stringify(query));
    _query.orderList = JSON.stringify(_query.orderList)
    return request({
        url: '/api/cdr',
        method: 'get',
        headers: {'Authorization': "Basic YWRtaW46MTIzNDU2"},
        params: _query
    })

{% endcodeblock %}

{% codeblock select.js lang:js %}
import request from '@/utils/request'

export function areaS2(query) {
    query.pageSize=50
    return request({
        url: '/api/select2/areaS2',
        method: 'get',
        headers: {'Authorization': "Basic YWRtaW46MTIzNDU2"},
        params: query
    })
}
{% endcodeblock %}

{% codeblock tableCustom.js  lang:js %}
/**
 * Created on 2018/9/18.
 * 表格相关的通用方法封装
 * 需要引用vue对象，call/apply 更改上下文
 * 总结通用属性:
 * tableKey-主键
 * list-数据列表
 * total-总数
 * listLoading-loading标识
 * listQuery-参数列表：通用参数{page: 1,offset: 0,limit: 20,orderList: []}
 * editDisable-编辑model显示标识
 * deleteDisable-删除model显示标识
 * checkedRows-checkbox选中列表
 * downloadLoading -下载loading
 * selectLoading  -select loading
 * lastDebounce   -页面查询控制：作为定时任务标识作为节流控制
 * lastModifyTm   -页面查询控制：记录上次查询时间作为节流控制
*
* /
 
  /**
 * @Date 2018/9/18 17:40
 * @Description 查询：清空chckedRows
 * @param func :传入的是查询func
 * @return
 */
export function getList(func) {
    this.listLoading = true
    func(this.listQuery).then(response => {
        var data = response.data.data;
        this.list = data.contentList
        this.total = data.totalElements
        this.checkedRows=[]
        this.listLoading = false
    }).catch(() => {
        this.listLoading = false
        this.$notify({
            title: this.$t('table.selectError'),
            message: this.$t('table.selectError'),
            type: 'warning',
            duration: 2000
        })
    });
}

export function handleDelete(func) {
    let idList = this.checkedRows.map(item => {
        return item.keyID || item[this.tableKey] ;
    })
    this.$confirm(this.$t('table.deleteTips').replace('{}',idList.length ), this.$t('table.tips'), {
        confirmButtonText: this.$t('table.confirm'),
        cancelButtonText: this.$t('table.cancel'),
        type: 'warning'
    }).then(() => {
        func(idList).then(response => {
            if (response.data.code == 0) {
                this.$message({
                    type: 'success',
                    message: this.$t('table.deleteSuccess')
                });
                this.getList()
            } else {
                this.$message({
                    type: 'error',
                    message: this.$t('table.deleteFailure')+':'+response.data.reason
                });
            }
        })
    }).catch(() => {
        this.$message({
            type: 'info',
            message: this.$t('table.deleteCancel')
        });
    });
}

function formatJson(filterVal, jsonData) {
    return jsonData.map(v => filterVal.map(j => {
        return v[j]
    }))
}

export function handleDownload(tHeader,func) {
    this.downloadLoading = true
    var exportListQuery = JSON.parse(JSON.stringify(this.listQuery));
    exportListQuery.table_columns = tHeader.join(",")
    exportListQuery.offset = 0
    //暂时限制1000条，可自由配置
    exportListQuery.limit = 1000
    func(exportListQuery).then(response => {
        const exportData = formatJson(tHeader, response.data.data.contentList);
        import
        ('@/vendor/Export2Excel').then(excel => {
            excel.export_json_to_excel({
                header: tHeader,
                data: exportData,
                filename: 'table-list'
            })
            this.$notify({
                title: this.$t('table.success'),
                message: this.$t('table.downLoadSuccess'),
                type: 'success',
                duration: 2000
            })

            this.downloadLoading = false
        })
    })
}

/**
 * @Date 2018/9/29 14:35
 * @Param idle-间隔时间,action-方法
 * @Description  针对页面查询的节流函数，控制页面请求频率
 * queryList停止变化一段时间后才会开始更新
 * 改进：这个延迟时间应该仅限制连续事件的触发即可，如果距离上一次更新时间超过两秒，应该直接执行
 * @return
 */
export function debounce (idle, action) {
    let now=(new Date()).getTime()
    let ctx = this, args = arguments
    if(!this.lastModifyTm || now-this.lastModifyTm>2000){
        this.lastModifyTm=now
        action.apply(ctx, args)
    }else {
        this.lastModifyTm=now
        return function(){
            clearTimeout(this.lastDebounce)
            this.lastDebounce = setTimeout(function(){
                action.apply(ctx, args)
            }, idle)
        }.apply(this)
    }
}

/**
 * @Date 2018/9/18 16:53
 * @Description 自定义表头排序：render方法，给排序按钮绑定点击查询事件
 * @return
 */
export function renderHeadSort(h, {column, $index}, index, name) {
    return h('span', [
        h('span', column.label),
        h('span',
            [
                h('i', {
                    attrs: {
                        id: name + '_up'
                    },
                    class: 'up',
//                            style: 'margin-left: 5px;',
                    on: {
                        click: () => {
                            let input = document.getElementById(name + '_up')
                            let itemClass = input.className
                            if (itemClass.indexOf("onU") == -1) {
                                input.classList.add("onU")
                                document.getElementById(name + '_down').classList.remove("onU")
                                this.listQuery.orderList = this.listQuery.orderList.filter(function (item) {
                                    return item[0] != name;
                                });
                                this.listQuery.orderList.push([name, 1])
                            } else {
                                input.classList.remove("onU")
                                this.listQuery.orderList = this.listQuery.orderList.filter(function (item) {
                                    return item[0] != name;
                                });
                            }
                        }
                    },
                }),
                h('i', {
                    class: 'down',
                    attrs: {
                        id: name + '_down'
                    },
                    on: {
                        click: () => {
                            let input = document.getElementById(name + '_down')
                            let itemClass = input.className
                            if (itemClass.indexOf("onU") == -1) {
                                input.classList.add("onU")
                                document.getElementById(name + '_up').classList.remove("onU")
                                this.listQuery.orderList = this.listQuery.orderList.filter(function (item) {
                                    return item[0] != name;
                                });
                                this.listQuery.orderList.push([name, 0])
                            } else {
                                input.classList.remove("onU")
                                this.listQuery.orderList = this.listQuery.orderList.filter(function (item) {
                                    return item[0] != name;
                                });
                            }
                        }
                    },
                }),
            ]
        ),
    ]);
}

/**
 * @Date 2018/10/18 10:14
 * @Description  compute方法，对查询道德数据进行处理转化成可以显示的文本
 * 对于比较特殊和复杂的条件，允许直接在表格参数中定义成func传入，在这里调用
 * 同时，由于进行更新删除操作时需要的是原始数据，故不可将原始数据与处理过的数据混淆
 * @return
 */
export function computedFmt(computedList,tableList) {
    let format,name,obj;
    for(let i=computedList.length-1;i>=0;i--){
        let tags = {};
        let tips={};
        obj = JSON.parse(JSON.stringify(computedList[i]))
        for(let j=tableList.length-1;j>=0;j--){
            name=tableList[j].name
            if(tableList[j].type=="tag"){
                if(obj[name] || obj[name] ==0){
                    //方法或对象
                    if(typeof tableList[j].tagData==="function"){
                        tags[name]=tableList[j].tagData(obj[name],obj)
                    }else{
                        tags[name]=tableList[j].tagData[obj[name]]
                    }
                }
            }
            if(tableList[j].type=="tip"){
                try {
                    if(tableList[j].tips && typeof tableList[j].tips === "function") { //tips是函数
                        tips[name]=tableList[j].tips(obj[name],obj)  //value,rowData参数
                    } else { //不是函数
                        tips[name]=tableList[j][name]
                    }
                } catch(e) {
                    tips[name]="-"
                }
            }
            if(tableList[j].format){
                format=tableList[j].format

                //优先执行自定义function，注意空值判断
                if(format.fmt){
                    if(typeof format.fmt === "function") { //fmt是函数
                        let fmt=format.fmt.call(this,obj[name],obj)
                        if(fmt.fmt){
                            obj[name]=fmt.fmt
                        }else {
                            obj[name]=fmt
                        }
                    }
                }
                if(!obj[name] && obj[name]!=0){
                    continue
                }
                if(format.ratio){
                    obj[name]=obj[name]/format.ratio
                }
                if(format.decimals){
                    obj[name]=toDecimals(obj[name],format.decimals)
                }
                if(format.comData){
                    obj[name]=format.comData[obj[name]] || obj[name]
                }
            }
        }
        obj.tags=tags
        obj.tips=tips
        computedList[i].showData=obj
    }
    return computedList;
}

/**
 * 
 * @param func remote select方法
 * @param query 用户输入val
 * @param name  查询的column
 */
export function remoteQuery(func,query,name) {
    format=format || ""
    if (query !== '') {
        this.selectLoading = true;
            func({query:query,idxOwnerId:'eu.'}).then(response => {
                this.selectLoading = false;
                const S2Data = response.data.data.items;
                this[name+'Options'] = S2Data.map(item => {
                    return {value: item.id, label: item.text};
                });
            }).catch(r => {
                this.selectLoading = false
            })
    }
}

export function remoteInitQuery(func,name) {
    format=format || ""
    func({query:'',idxOwnerId:'eu.'}).then(response => {
               const S2Data = response.data.data.items;
               this[name+'InitOptions'] = S2Data.map(item => {
                   return {value: item.id, label: item.text};
               });
               this[name+'Options'] = this[name+'InitOptions'];
           })
}
{% endcodeblock %}

### 遇到的问题
## 1.表格组件化的目的
&emsp;&emsp;一为减少模板代码量，二为方便对代码进行管理，同时保持可扩展性。就本人使用了一段时间vue的体验来讲，使用组件写业务代码并不会使代码简洁，甚至显得更啰嗦，但是代码层次清晰明了，便于理解，而且vue的组件的生命周期能提供粒度更细的控制。

## 2.原始数据与展示数据的单独保存
&emsp;&emsp;大致上将业务上使用的表格分成了只提供查询功能的简单表格和提供了增删查改等的复杂表格。对于简单表格来讲，只需要进行查询和数据的展示，但对复杂表格来说，进行修改删除操作时，只有原始数据才是有意义的，所以将原始数据存进data的list中，方便进行其他操作时直接引用，而展示数据则在computed中处理后存进computedLists，computedLists直接与表格绑定

## 3.在computed中转化数据
&emsp;&emsp;这里可以使用Element UI自带的expressions和vue的filters模块，不得不说Element UI的功能很强大，一个简单的input组件都能封装上百行代码。。。
&emsp;&emsp;expressions也可用作文本转换，但一来它不支持复杂的逻辑，二来expressions是直接混在组件template中的，大量使用会让代码结构显得混乱。filters相对expressions来说就规范点，但仍是嵌在template中，而且给每个vue实例写重复的filters也很烦。既然这样，不如直接在computed中一次性完成所有文本转换，这样代码量也少了很多。
&emsp;&emsp;computed的第二个好处是性能好。就使用效果来看，computed与method没有区别，但性能上差距很大。method中，每次页面渲染时都会重新执行一次。而computed的特点是依赖收集、动态计算，依赖的值发生变化才会修改，否则取缓存值，因此对于非响应式依赖（如Date.now()），计算一次后将不会发生改变。多数情况下computed够用，但如果要在数据变化响应时，执行异步操作或开销较大的操作，可以使用watct。

## 4.watch监听搜索并使用节流函数
&emsp;&emsp;一开始我还傻乎乎的给每个输入框绑定事件，后来发现完全没必要弄得这么麻烦。直接将所有相关参数绑定到listQuery中，监听listQuery的变化即可。同时，为了防止文本输入框查询频率过高，加入了节流函数进行限制。

## 5.renderHeadSort
&emsp;&emsp;Element Table其实有提供现成的排序功能，无论是前端排序还是访问后端查询接口，但我不喜欢它的样式，所以自己写了一个(任性

## 6.call/apply的使用
&emsp;&emsp;tableCustom.js和table.vue实例中存在大量的call/apply方法，因为表格封装后只剩下少部分参数和方法在实例中，使用call/apply可以在外部代码中引用到这些方法，方便将这部分代码与通用的部分分离开来。而且由于项目中引入了vue-i18n国际化插件，而i18n又跟vue实例绑定，所以需要用到i18n做文本转换的部分不得不使用call/apply来获取vue实例。

## 7.父子组件的两种通信方式
{% codeblock %}
this.$emit('func',params) 子组件触发func事件，父组件监听，这种模式无法获取返回值；
this.parent.func(params)  子组件直接调用父组件方法，可直接获取返回值，不过父组件中必须注册对应方法，否则报错。
{% endcodeblock %}

## 8.table参数中的format
&emsp;&emsp;format即文本转换规则，是一个Object，里面可以存储多个规则，如:
{% codeblock %}
decimals   小数点设置
areaFM     地区样式
comData    语义化，比如val=-1在页面显示失败，value=1显示成功,value=0显示进行中
{% endcodeblock %}

&emsp;&emsp;对于用的比较少的特殊规则，则可以直接传递对应function：
{% codeblock lang:js %}
if(format.fmt){
    if(typeof format.fmt === "function") { //fmt是函数
        let fmt=format.fmt.call(this,obj[name],obj)
        if(fmt.fmt){
            obj[name]=fmt.fmt
        }else {
            obj[name]=fmt
        }
    }
}
{% endcodeblock %}

## 9.mixin整合重复代码
&emsp;&emsp;实际上这个表格仍有一部分(大概一百来行)可以压缩到mixin里面，这样子实际调用起来会显得更简洁，有时间再整理出来吧。


### 效果
<div align=center>{% asset_img 1.gif  %}</div>
&emsp;&emsp;这样子一个简单的表格组件就差不多完成了，个人感觉写的并不够好，不过既然能满足日常业务需求，也就不吹毛求疵了。另外这篇博客字有点多，可以拆开来的，不过本人懒癌发作，就不整理了<font face="黑体" color=red size=5>（逃</font>

<template>
    <div class="container-wrap">
        <div style="display:flex">
            <div style="flex:1">
                <label>科目名称</label>
                <Input v-model="subjectName" size="large" placeholder="请输入科目名称" style="width: 200px;margin-left:10px" />
                <label style="margin-left:10px">ID</label>
                <Input v-model="subjectId" size="large" placeholder="请输入科目ID" style="width: 200px;margin-left:10px" />
                <Button type="primary" class="search-btn" @click="search">搜索</Button></Col>
            </div>
            <div style="width:100px;text-align:right">
                <Button type="success" @click="addSubject">新增科目</Button>
            </div>
        </div>
        <Row class="table-wrap">
            <Table  :loading="loading" :columns="columns" :data="subjects_datas">
                <template slot-scope="{ row }" slot="price">
                    <strong v-if="row.price&&row.price>0">¥{{ row.price }}元</strong>
                    <strong v-else>免费</strong>
                </template>
                <template slot-scope="{ row }" slot="content">
                    <strong>{{ row.content?'已添加':'---' }}</strong>
                </template>
                <template slot-scope="{ row }" slot="action">
                    <Button v-if="row.level > 0" type="info" size="small" style="margin-right:5px" @click="ShowParents(row)">查看上一级</Button>
                    <Button v-if="!row.has_down_level" type="primary" size="small" style="margin-right:5px" @click="AddChildrens(row)">添加下一级</Button>
                    <Button v-else type="info" size="small" style="margin-right:5px" @click="ShowChildrens(row)">查看下一级</Button>
                    <Button type="warning" size="small" style="margin-right:5px" @click="ContentFormShow(row)">编辑内容</Button>
                    <Button type="warning" size="small" style="margin-right:5px" @click="EditFormShow(row)">修改</Button>
                    <Button type="error" size="small" @click="DelConfirmShow(row)">删除</Button>
                </template>
            </Table>
            <div class="page-wrap">
                <Page :total="total" @on-change="page_list"  v-if="total!=0" />
            </div>
        </Row>
         <Modal v-model="isShowAddForm"
            :title="window_title"
            width="400"
            mask scrollable
            :loading='modalLoading'
            @on-ok="add_subjects">
            <Form ref="subjectForm" :model="subject_form" label-position="right" :label-width="80" :rules="ruleValidate">
                <FormItem label="科目名称" prop='subject_name'>
                    <Input v-model="subject_form.subject_name" placeholder="请输入科目名称"></Input>
                </FormItem>
                <FormItem label="是否收费" prop='price'>
                    <div style="display:flex">
                        <div style="width:80px">
                            <i-switch v-model="subject_form.free" size="large">
                                <span slot="open">收费</span>
                                <span slot="close">免费</span>
                            </i-switch>
                        </div>
                        <div style="flex:1">
                            <Input v-if="subject_form.free" v-model="subject_form.price" placeholder="请输入收费金额">
                                <span slot="append">元</span>
                            </Input>
                        </div>
                    </div>
                </FormItem>
            </Form>
         </Modal>
         <Modal v-model="isShowContentForm" :title="window_title" width="800" @on-ok="saveContent()">
            <Editor :value="subject_form.content" @on-change="change_value"></Editor>
         </Modal>
    </div>
</template> 

<script>
import Editor from "@/components/editor"
import { verification } from '@/api/verification'
export default {
    name: "subjectsmanageindex",
    data() {
        var self = this
        return {
            page:1,
            pageSize: 10,
            total:0,
            subjectName:'',
            subjectId:'',
            currLevel:0,
            currParent: {
                id:'0',
                price:''
            },
            route:[{
                id:'0',
                price:''
            }],
            columns: [
                    {
                        title: '科目名称',
                        key: 'subject_name'
                    },
                    {
                        title: 'ID',
                        key: 'id'
                    },
                    {
                        title: "内容添加",
                        key: 'content',
                        slot:'content'
                    },
                    {
                        title: "收费状态",
                        key: 'price',
                        slot:'price'
                    },
                    {
                        title: '操作',
                        key: 'action',
                        width:380,
                        slot:'action'
                    }
                ],
            subjects_datas: [],
            subject_form:{
                subject_name:'',
                subject_ID:'',
                free:false,
                price: '0',
                content:'',
                has_down_level:false,
                parent_ID:'0'
            },
            ruleValidate: {
                subject_name: [{ required: true, message: '请输入科目/章节名称', trigger: 'blur' }],
                price: [{ trigger: 'blur', 
                        validator: (rule, value, callback) => {
                            if(self.subject_form.free){
                                if (value === '' ) {
                                    callback(new Error('请输入价格'))
                                } else if (!/(^[1-9]([0-9]+)?(\.[0-9]{1,2})?$)|(^(0){1}$)|(^[0-9]\.[0-9]([0-9])?$)/.test(value)) {
                                    return callback(new Error('价格由整数、小数点和最多两个小数组成！'))
                                } else {
                                    callback()
                                }
                            } else {
                                callback()
                            }
                        }
                    }
                ]
            },
            old_price:0,
            init_data:"",
            parentid:"0",
            subjectid:"",
            isShowAddForm: false,
            isShowContentForm: false,
            loading: true,
            modalLoading: true,
            window_title:"新增科目"
        }
    },
      components:{
        Editor
    },
    computed:{
        show_price(){
            return this.subject_form.free
        }
    },
      mounted() {
        this.init_data=JSON.stringify(this.subject_form)
        this.page_list(this.page)
    },
    methods: {
        cancel(){
            this.parentid=""
            this.subjectid=""
            this.subject_form=JSON.parse(this.init_data)
            this.isShowAddForm=false,
            this.window_title="新增科目"
        },

         /*
        *获取科目实体
        *作者：gzt
        *时间：2020-11-22 09:21:48
        */
        get_entity(){
            var query = new this.ParseServer.Query("Subjects")
            query.get(this.subjectid).then(res=>{
                Object.keys(this.subject_form).forEach(key => {
                    this.subject_form[key]=res.get(key)
                    
                })
                this.subject_form.free = res.get('price') && parseFloat(res.get('price'))>0
                this.old_price=res.get("price")==null?0:res.get("price")
            })
        },
        /** 弹出添加科目弹框 */
        addSubject(){
            this.window_title="添加科目"
            this.isShowAddForm=true
        },
        /**
         * 弹出编辑窗口
         */
        EditFormShow(row) {
            this.subjectid=row.id
            this.isShowAddForm=true
            this.window_title="编辑科目"
            this.get_entity()
        },
        /**
         * 弹出编辑内容窗体
         */
        ContentFormShow(row){
            this.subjectid=row.id
            this.subject_form.content = row.content
            this.isShowContentForm=true
            this.window_title="编辑内容"
        },
        /** 查看上一级 */
        ShowParents(row){
            this.route.pop()
            this.currParent = this.route[this.route.length-1]
            this.page_list(1)
        },
        /** 查看下一级 */
        ShowChildrens(row){
            this.route.push(row)
            this.currLevel += 1
            this.currParent = row
            // this.old_price = row.price
            this.page_list(1)
        },
        /** 添加下一级 */
        AddChildrens(row) {
            var self = this
            this.$Modal.confirm({
                title: '操作提示',
                content: '<p>是否添加下级目录？</p>',
                onOk: () => {
                    self.route.push(row.id)
                    self.currLevel += 1
                    self.currParent = row
                    self.page_list(1)
                }
            })
        },
          /*
        *新增科目
        *作者：gzt
        *时间：2020-11-21 23:41:37
        */
        add_subjects(){
            var self = this
            var subjects = this.ParseServer.Object.extend("Subjects")
            var subject = new subjects()
            if(this.subjectid) {
                subject.set('id', this.subjectid)
            }
            this.subject_form.parent_ID = this.currParent.id
            this.$refs['subjectForm'].validate(valid => {
                if (!valid) {
                    self.$Message.error('请检查表单项')
                    self.modalLoading = false
                    setTimeout(() => {
                        self.modalLoading = true
                    }, 100)
                    return false
                } else {
                    subject.set('subject_name', self.subject_form.subject_name)
                    subject.set("content", '')
                    subject.set("price", self.subject_form.price)
                    subject.set("level", self.currLevel)
                    subject.set("parent_ID",self.currParent.id)
                    subject.set("has_down_level",self.subject_form.has_down_level)
                    subject.save().then((subject)=>{
                        if (self.currParent.id!="0"||self.currParent.id!=0){
                            self.updateParentPrice(self.currParent.id)
                        }else{
                            this.$Message.success('保存成功')
                            this.page_list(1)
                            this.cancel()
                        }

                    },(error)=>{
                        console.log(error)
                        this.$Message.error('保存失败')
                    })
                }
            })
        },
        /** 更新父级科目 */
        updateParentPrice(parentId){
            var self = this
            var query = new self.ParseServer.Query("Subjects")
            query.equalTo("parent_ID", parentId)
            query.find().then(childrens => {
                let hasChildren = false
                let price = 0
                childrens.forEach((_item, _index)=>{
                    hasChildren = true
                    price += parseFloat(_item.get('price'))
                })
                var query1 = new self.ParseServer.Query("Subjects")
                query1.get(parentId).then((parent)=>{
                    parent.set("has_down_level", hasChildren)
                    parent.set("price", '' + price.toFixed(2))
                    parent.save().then((result)=>{
                        if(parent.get('parent_ID')!='0'){
                            self.updateParentPrice(parent.get('parent_ID'))
                        } else {
                            self.$Message.success('保存成功')
                            self.page_list(1)
                            self.cancel()
                        }
                    })
                })
            })
            
        },
        /*
        *富文本编辑框的内容发生变化
        *作者：gzt
        *时间：2020-11-22 00:37:46
        */
        change_value(html,text){
            this.subject_form.content=html
        },
        saveContent(){
            var self = this
            var subjects = this.ParseServer.Object.extend("Subjects")
            var subject = new subjects()
            subject.set('id', this.subjectid)
            subject.set("content", this.subject_form.content)
            subject.save().then((res)=>{
                self.page_list(self.page)
                self.$Message.success('保存成功')
            },(error)=>{
                console.log(error)
                self.$Message.error('保存失败')
            })
        },
        /*
        *搜索数据
        *作者：gzt
        *时间：2020-11-21 23:30:19
        */
        search(){
            this.page=1
           this.page_list(this.page) 
        },
        /*
        *分页加载数据
        *作者：gzt
        *时间：2020-11-21 23:30:27
        */
        page_list(page_index){
            let _this=this
            let query = new this.ParseServer.Query('Subjects')
            query.equalTo('parent_ID', this.currParent.id)
            query.count().then(count=>{
                _this.total=count
            })
            query.skip((this.page-1)*this.pageSize)
            query.limit(this.pageSize)
            query.find().then(list => {
                _this.subjects_datas=[]
                if (list && list.length>0){
                    list.forEach((item)=>{
                        _this.subjects_datas.push({
                            id:item.id,
                            subject_name:item.get("subject_name"),
                            parent_ID:item.get("parent_ID"),
                            subject_ID:item.get("subject_ID"),
                            price:item.get("price"),
                            level: item.get('level'),
                            content: item.get('content'),
                            has_down_level: item.get('has_down_level')
                        })
                    })
                }
                this.loading=false
            })
        },

        /*
        *递归删除科目
        *作者：gzt
        *时间：2020-11-22 11:57:11
        */
        recursive_delete(parent_id){
            var query_deletes=new this.ParseServer.Query("Subjects")
            query_deletes.equalTo("parent_ID",parent_id)
            query_deletes.find().then((response)=>{
                if(response&&response.length>0){
                    response.forEach((data)=>{
                        data.destroy().then((result)=>{
                            this.recursive_delete(data.id)
                        })
                    })
                }
            })
        },


        /*
        * 删除科目
        *作者：gzt
        *时间：2020-11-22 08:41:53
        */
        DelConfirmShow(row){
            var subject_id = row.id
            let _this=this
            this.$Modal.confirm({
                    title: '删除提示',
                    content: '<p>删除当前科目后，包含的子科目都会被删除，确定要删除吗？</p>',
                    onOk: () => {
                        var query = new this.ParseServer.Query("Subjects")
                        query.get(subject_id).then((response)=>{

                            // 删除所有的子组件
                           this.recursive_delete(response.id)
                            // 删除当前组件
                            response.destroy().then((delete_result)=>{
                                // 更新当前的父级组件是否还存在其他的子组件
                                    query.equalTo("parent_ID",response.get("parent_ID"))
                                    query.find().then((result)=>{
                                        if(result.length==0){
                                            var query_get = new this.ParseServer.Query("Subjects")
                                            if(response.get("parent_ID")=="0"||response.get("parent_ID")==""){
                                                this.$Message.success('删除成功');
                                                _this.page_list(this.page)
                                                return
                                            }
                                            query_get.get(response.get("parent_ID")).then((subject)=>{
                                                subject.set('has_down_level',false)
                                                subject.save().then((r)=>{
                                                     this.$Message.success('删除成功');
                                                    _this.page_list(this.page)
                                                })
                                            })
                                        }else{
                                            this.$Message.success('删除成功');
                                            _this.page_list(this.page)
                                        }
                                    })
                                })
                        })
                    },
                    onCancel: () => {
                        this.$Message.info('取消了操作');
                    }
                });
        }
    }
}
</script>

<style lang="less" scoped>
</style>

<template>
	<section>
		<!--工具条-->
		<el-col :span="24" class="toolbar" style="padding-bottom: 0px;">
			<el-form :inline="true" :model="filters">
				<el-form-item>
					<el-input v-model="filters.keyWord" placeholder="关键字"></el-input>
				</el-form-item>
				<el-form-item>
					<el-button type="primary" v-on:click="getBrands">查询</el-button>
				</el-form-item>
				<el-form-item>
					<el-button type="primary" @click="handleAdd">新增</el-button>
				</el-form-item>
			</el-form>
		</el-col>

		<!--列表-->
		<el-table :data="brands" highlight-current-row v-loading="listLoading" @selection-change="selsChange" style="width: 100%;">
			<el-table-column type="selection" width="60%">
			</el-table-column>
			<el-table-column type="index" width="150%">
			</el-table-column>
			<el-table-column prop="name" label="品牌名称" width="150%" sortable>
			</el-table-column>
			<el-table-column prop="englishName" label="英文名称" width="250%" sortable>
			</el-table-column>
			<el-table-column prop="logo" label="logo" width="350%" sortable>
			</el-table-column>
			<el-table-column prop="productType.name" label="类型" width="150%" sortable>
			</el-table-column>
			<el-table-column prop="description" label="描述" min-width="150%" sortable>
			</el-table-column>
			<el-table-column label="操作" width="150">
				<!--自定义列显示的模板-->
				<template scope="scope">
					<el-button size="small" @click="handleEdit(scope.$index, scope.row)">编辑</el-button>
					<!--  scope.$index 行索引      scope.row  该行数据  -->
					<el-button type="danger" size="small" @click="handleDel(scope.$index, scope.row)">删除</el-button>
				</template>
			</el-table-column>
		</el-table>

		<!--工具条-->
		<el-col :span="24" class="toolbar">
			<el-button type="danger" @click="batchRemove" :disabled="this.sels.length===0">批量删除</el-button>
			<el-pagination layout="prev, pager, next" @current-change="handleCurrentChange" :page-size="pageSize" :total="total" style="float:right;">
			</el-pagination>
		</el-col>

		<!--编辑界面-->
		<el-dialog title="编辑" :visible="editFormVisible" :close-on-click-modal="false">
			<el-form :model="editForm" label-width="80px" :rules="editFormRules" ref="editForm">
				<el-form-item label="姓名" prop="name">
					<el-input v-model="editForm.name" auto-complete="off"></el-input>
				</el-form-item>
				<el-form-item label="性别">
					<el-radio-group v-model="editForm.sex">
						<el-radio class="radio" :label="1">男</el-radio>
						<el-radio class="radio" :label="0">女</el-radio>
					</el-radio-group>
				</el-form-item>
				<el-form-item label="年龄">
					<el-input-number v-model="editForm.age" :min="0" :max="200"></el-input-number>
				</el-form-item>
				<el-form-item label="生日">
					<el-date-picker type="date" placeholder="选择日期" v-model="editForm.birth"></el-date-picker>
				</el-form-item>
				<el-form-item label="地址">
					<el-input type="textarea" v-model="editForm.addr"></el-input>
				</el-form-item>
			</el-form>
			<div slot="footer" class="dialog-footer">
				<el-button @click.native="editFormVisible = false">取消</el-button>
				<el-button type="primary" @click.native="editSubmit" :loading="editLoading">提交</el-button>
			</div>
		</el-dialog>

		<!--新增界面-->
		<!--
		v-model  true/false 模态框是否显示
		elementui组件有方法，我们只有获取到组件对象后才能调用他的方法
		如何去获取到组件对象
		(1)组件上添加 ref属性
		(2)this.$refs.addForm 获取到组件对象
		  -->
		<el-dialog title="新增" :visible.sync="addFormVisible" size="tiny">
			<el-form :model="addForm" label-width="80px" :rules="addFormRules" ref="addForm">
				<!--label 表单组件的信息  prop  验证使用-->
				<el-form-item label="品牌名称" prop="name">
					<el-input v-model="addForm.name" auto-complete="off"></el-input>
				</el-form-item>
				<el-form-item label="英文名称" prop="englishName">
					<el-input v-model="addForm.englishName" auto-complete="off"></el-input>
				</el-form-item>
				<el-form-item label="类型" prop="productTypeId">
					<el-cascader
							v-model="props.value"
							:options="productType"
							:props="props"
							:show-all-levels="false"
							@change="handleChange"></el-cascader>
				</el-form-item>
                <el-form-item label="Logo">
                    <el-upload
                            class="upload-demo"
                            action="http://localhost:3000/services/common/fastdfs"
                            :on-preview="handlePreview"
                            :on-remove="handleRemove"
                            :on-success="handleSuccess"
                            :on-exceed="handleExceed"
                            :file-list="fileList"
                            :multiple="true"
                            :limit="1"
                            list-type="picture">
                        <el-button size="small" type="primary">点击上传</el-button>
                        <div slot="tip" class="el-upload__tip">只能上传jpg/png文件，且不超过500kb</div>
                    </el-upload>
                </el-form-item>
				<el-form-item label="描述">
					<el-input type="textarea" v-model="addForm.description"></el-input>
				</el-form-item>
			</el-form>
			<div slot="footer" class="dialog-footer">
				<el-button @click.native="addFormVisible = false">取消</el-button>
				<el-button type="primary" @click.native="addSubmit" :loading="addLoading">提交</el-button>
			</div>
		</el-dialog>
	</section>
</template>

<script>
	import util from '../../common/js/util'
	//import NProgress from 'nprogress'
	import { getUserListPage, removeUser, batchRemoveUser, editUser, addUser } from '../../api/api';

	export default {
		data() {
			return {
                fileList: [
                    {
                        name: 'food.jpeg',
                        url: 'https://fuss10.elemecdn.com/3/63/4e7f3a15429bfda99bce42a18cdd1jpeg.jpeg?imageMogr2/thumbnail/360x360/format/webp/quality/100'
                    }
                    ],
                props:{
                    label:"name",
                    value:"id",
					expandTrigger: 'hover'
                },
                filters: {
                    keyWord: ''
                },
                pageSize:10,
                brands: [],
                total: 0,
                page: 1,
                listLoading: false,
                sels: [],//列表选中列
				productType:[],
                editFormVisible: false,//编辑界面是否显示
                editLoading: false,
                editFormRules: {
                    name: [
                        { required: true, message: '请输入姓名', trigger: 'blur' }
                    ]
                },
                //编辑界面数据
                editForm: {
                    id: 0,
                    name: '',
                    sex: -1,
                    age: 0,
                    birth: '',
                    addr: ''
                },

                addFormVisible: false,//新增界面是否显示
                addLoading: false,
                addFormRules: {
                    name: [
                        { required: true, message: '请输入姓名', trigger: 'blur' }
                    ]
                },
                //新增界面数据
                addForm: {
                    name: '',
                    englishName:'',
                    productTypeId:null,
                    description:''
                }

            }
		},
		methods: {
            handleExceed(files, fileList){
                this.$message({
                    message: '只能上传一张图片哦！！',
                    type: 'warning'
                });
            },
            handleSuccess(response, file, fileList){
                console.log("response:",response)
                console.log("file:",file)
                console.log("fileList:",fileList)
            },
            handleRemove(file, fileList) {
                console.log(file, fileList);
            },
            handlePreview(file) {
                console.log(file);
            },
			//性别显示转换
			formatSex(row, column) {
				return row.sex == 1 ? '男' : row.sex == 0 ? '女' : '未知';
			},
			handleCurrentChange(val) {
				this.pageNum = val;
				this.getBrands();
			},
			//获取品牌列表
			getBrands() {
				this.listLoading = true;
				this.$http.post("/product/brand/json" , {
				    	"pageSize":this.pageSize,
						"pageNum":this.pageNum,
						"keyWord":this.filters.keyWord
					})
					.then((res)=>{
                        this.listLoading = false;
						let pageList = res.data;
						this.brands = pageList.rows;
						this.total = pageList.total;
					})
			},
            //获取品牌类型列表
            getProuductType(){
			    this.$http.get("/product/productType/list")
					.then((res)=>{
					    this.productType = res.data;
					})
			},
			//删除
			handleDel(index, row) {
                this.$confirm('确认删除该记录吗?', '提示', {
                    type: 'warning'
                }).then(() => {
                    //确定删除
                    this.listLoading = true;
                    //NProgress.start();
                    this.$http.delete("/product/brand/delete/"+row.id)
                        .then((res)=>{
                            this.listLoading = false;
                            if(res.data.success){
                                this.$message({
                                    message: '删除成功',
                                    type: 'success'
                                });
                                this.getBrands();
                            }else{
                                //失败的提示
                                this.$message({
                                    message: res.data.message,
                                    type: 'error'
                                });
                            }
                        })
                })
			},
            //显示编辑界面
            handleEdit(index, row) {
                this.editFormVisible = true;
                this.editForm = Object.assign({}, row);
            },
            //显示新增界面
            handleAdd() {
                this.addFormVisible = true;
                this.addForm = {
                    name: '',
                    englishName:'',
                    productTypeId:null,
                    description:''
                };
            },
			//编辑
			editSubmit() {
				this.$refs.editForm.validate((valid) => {
					if (valid) {
						this.$confirm('确认提交吗？', '提示', {}).then(() => {
							this.editLoading = true;
							//NProgress.start();
							let para = Object.assign({}, this.editForm);
							para.birth = (!para.birth || para.birth == '') ? '' : util.formatDate.format(new Date(para.birth), 'yyyy-MM-dd');
							editUser(para).then((res) => {
								this.editLoading = false;
								//NProgress.done();
								this.$message({
									message: '提交成功',
									type: 'success'
								});
								this.$refs['editForm'].resetFields();
								this.editFormVisible = false;
								this.getBrands();
							});
						});
					}
				});
			},
			//新增
			addSubmit() {
                this.$refs.addForm.validate((valid) => {
                    if (valid) {
                        this.$confirm('确认提交吗？', '提示', {}).then(() => {
                            this.addLoading = true;
                            //NProgress.start();
                            let para = Object.assign({}, this.addForm);//对象的复制
                            //级联选择器的结果是一个数组
                            para.productTypeId = para.productTypeId[para.productTypeId.length-1];
                            this.$http.post("/product/brand/add",para)
                                .then(res=>{
                                    this.addLoading = false;
                                    let data = res.data;
                                    if(data.success){
                                        this.$message({
                                            message: '提交成功',
                                            type: 'success'
                                        });
                                        this.$refs['addForm'].resetFields();
                                        this.addFormVisible = false;
                                        this.getBrands();
                                    }else{
                                        this.$message({
                                            message: data.message,
                                            type: 'error'
                                        });
                                    }
                                })
                        });
                    }
                });
			},
			//新增界面类型级联
            handleChange(value){
                this.addForm.productTypeId = value
            },
			selsChange(sels) {
				this.sels = sels;
			},
			//批量删除
			batchRemove() {
				var ids = this.sels.map(item => item.id).toString();
				this.$confirm('确认删除选中记录吗？', '提示', {
					type: 'warning'
				}).then(() => {
					this.listLoading = true;
					//NProgress.start();
					let para = { ids: ids };
					batchRemoveUser(para).then((res) => {
						this.listLoading = false;
						//NProgress.done();
						this.$message({
							message: '删除成功',
							type: 'success'
						});
						this.getBrands();
					});
				}).catch(() => {

				});
			}
		},
		//相当于jquery的$(function(){})
		mounted() {
			this.getBrands();
			this.getProuductType();
		}
	}

</script>

<style scoped>

</style>
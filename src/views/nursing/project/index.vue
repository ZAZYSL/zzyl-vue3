<template>
  <div class="app-container">
    <!-- 搜索筛选表单 -->
    <el-form :model="queryParams" ref="queryRef" :inline="true" v-show="showSearch" label-width="68px">
      <el-form-item label="名称" prop="name">
        <el-input
          v-model="queryParams.name"
          placeholder="请输入名称"
          clearable
          @keyup.enter="handleQuery"
        />
      </el-form-item>
      <el-form-item label="状态" prop="status">
        <el-select v-model="queryParams.status" placeholder="请选择" clearable style="width: 220px">
          <el-option
              v-for="item in nursing_project_status"
              :key="item.value"
              :label="item.label"
              :value="item.value" />
          <el-option v-for="item in options" :key="item.value" :label="item.label" :value="item.value" />
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="Search" @click="handleQuery">搜索</el-button>
        <el-button icon="Refresh" @click="resetQuery">重置</el-button>
      </el-form-item>

    </el-form>

    <!-- 操作按钮工具栏 -->
    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button
          type="primary"
          plain
          icon="Plus"
          @click="handleAdd"
          v-hasPermi="['nursing:project:add']"
        >新增</el-button>
      </el-col>

    </el-row>

    <!-- 护理项目数据表格 -->
    <el-table v-loading="loading" :data="projectList" @selection-change="handleSelectionChange">
      <!-- 护理项目序号 -->
      <el-table-column label="序号" type="index" align="center" />
      <!-- 护理项目名称 -->
      <el-table-column label="名称" align="center" prop="name" />
      <!-- 排序号 -->
      <el-table-column label="排序号" align="center" prop="orderNo" />
      <!-- 单位 -->
      <el-table-column label="单位" align="center" prop="unit" />
      <!-- 价格 -->
      <el-table-column label="价格" align="center" prop="price" />
      <!-- 图片预览列 -->
      <el-table-column label="图片" align="center" prop="image" width="100">
        <template #default="scope">
          <image-preview :src="scope.row.image" :width="50" :height="50"/>
        </template>
      </el-table-column>
      <!-- 护理要求 -->
      <el-table-column label="护理要求" align="center" prop="nursingRequirement" />
      <!-- 状态 -->
      <el-table-column label="状态" align="center" prop="status" >
        <template #default="scope">
          <el-tag :type="scope.row.status == 1 ? 'success' : 'danger'">{{ scope.row.status == 1 ? '启用' : '禁用' }}</el-tag>
        </template>
      </el-table-column>
      <!-- 创建时间 -->
      <el-table-column label="创建时间" align="center" prop="createTime" width="180">
        <template #default="scope">
          <span>{{ parseTime(scope.row.createTime, '{y}-{m}-{d} {h}:{i}:{s}') }}</span>
        </template>
      </el-table-column>
      <!-- 操作列（行级修改、删除） -->
      <el-table-column label="操作" fixed="right" width="200" align="center" class-name="small-padding fixed-width">
        <template #default="scope">
          <el-button link type="primary" icon="Edit" @click="handleUpdate(scope.row)" v-hasPermi="['nursing:project:edit']">修改</el-button>
          <el-button link type="primary" icon="Delete" @click="handleDelete(scope.row)" v-hasPermi="['nursing:project:remove']">删除</el-button>
          <el-button link type="primary" :icon="scope.row.status == 0 ? 'Lock' : 'Unlock'" @click="handleEnable(scope.row)" >{{ scope.row.status == 0 ? '启用' : '禁用' }}</el-button>
        </template>
      </el-table-column>
    </el-table>
    
    <!-- 分页组件 -->
    <pagination
      v-show="total>0"
      :total="total"
      v-model:page="queryParams.pageNum"
      v-model:limit="queryParams.pageSize"
      @pagination="getList"
    />

    <!-- 添加或修改护理项目对话框 -->
    <el-dialog :title="title" v-model="open" width="500px" append-to-body>
      <el-form ref="projectRef" :model="form" :rules="rules" label-width="80px">
        <el-form-item label="名称" prop="name">
          <el-input v-model="form.name" placeholder="请输入名称" />
        </el-form-item>
        <el-form-item label="排序号" prop="orderNo">
          <el-input-number v-model="form.orderNo"
                           placeholder="请输入"
                           :min="1"
                           :max="20" />
        </el-form-item>
        <el-form-item label="单位" prop="unit">
          <el-input v-model="form.unit" placeholder="请输入单位" />
        </el-form-item>
        <el-form-item label="价格" prop="price">
          <el-input-number v-model="form.price"
                           placeholder="请输入"
                           :min="5"
                           :max="100"
                           :step="5" />
        </el-form-item>
        <el-form-item label="状态" prop="status">
          <el-radio-group v-model="form.status">
            <el-radio v-for="dict in nursing_project_status"
                      :key="dict.value"
                      :label="dict.value"
                      :value="dict.value">
              {{ dict.label }}
            </el-radio>
          </el-radio-group>
        </el-form-item>
        <!-- 项目图片上传组件 -->
        <el-form-item label="图片" prop="image">
          <image-upload v-model="form.image"/>
        </el-form-item>
        <el-form-item label="护理要求" prop="nursingRequirement">
          <el-input v-model="form.nursingRequirement" placeholder="请输入护理要求" type="textarea"/>
        </el-form-item>
      </el-form>
      <template #footer>
        <div class="dialog-footer">
          <el-button type="primary" @click="submitForm">确 定</el-button>
          <el-button @click="cancel">取 消</el-button>
        </div>
      </template>
    </el-dialog>
  </div>
</template>

<script setup name="Project">


// 引入护理项目相关的 API 请求方法
import { listProject, getProject, delProject, addProject, updateProject } from "@/api/nursing/project"

// 获取当前组件实例上下文，用于访问全局绑定的工具方法（如 $modal、download、resetForm）
const { proxy } = getCurrentInstance()

//引用数据字典
const { nursing_project_status } = proxy.useDict("nursing_project_status");
// 护理项目表格数据列表
const projectList = ref([])
// 控制添加/修改弹窗的显示与隐藏
const open = ref(false)
// 表格加载遮罩层状态（true: 显示加载中，false: 加载完成）
const loading = ref(true)
// 控制搜索区域的显示与隐藏
const showSearch = ref(true)
// 表格多选选中的项目 ID 数组
const ids = ref([])
// 非单个选中状态（为 true 时禁用修改按钮，只有恰好选中1条数据时才为 false 允许修改）
const single = ref(true)
// 非多选状态（为 true 时禁用删除按钮，至少选中1条数据时为 false 允许删除）
const multiple = ref(true)
// 数据总条数（用于分页）
const total = ref(0)
// 弹窗对话框标题（"添加护理项目" 或 "修改护理项目"）
const title = ref("")

// 定义响应式数据集合（包含表单对象、查询参数、表单校验规则）
const data = reactive({
  // 添加或修改时提交的表单数据对象
  form: {},
  // 查询参数对象（分页及过滤条件）
  queryParams: {
    pageNum: 1,
    pageSize: 10,
    name: null,
    status: null,
  },
  // 表单字段验证规则
  rules: {
    name: [
      { required: true, message: "名称不能为空", trigger: "blur" }
    ],
    price: [
      { required: true, message: "价格不能为空", trigger: "blur" }
    ],
    image: [
      { required: true, message: "图片不能为空", trigger: "blur" }
    ],
    nursingRequirement: [
      { required: true, message: "护理要求不能为空", trigger: "blur" }
    ],
  }
})

// 将 reactive 对象的内部属性转换为 ref 响应式引用，方便在模板中直接使用
const { queryParams, form, rules } = toRefs(data)

/** 
 * 查询护理项目列表
 * 开启 loading，调用接口获取分页列表数据并更新状态
 */
function getList() {
  loading.value = true
  listProject(queryParams.value).then(response => {
    projectList.value = response.rows
    total.value = response.total
    loading.value = false
  })
}

/** 
 * 取消按钮操作
 * 关闭添加/修改弹窗并重置表单数据
 */
function cancel() {
  open.value = false
  reset()
}

/** 
 * 表单重置
 * 初始化表单字段默认值，并清空 Element Plus 表单校验提示
 */
function reset() {
  form.value = {
    id: null,
    name: null,
    orderNo: null,
    unit: null,
    price: null,
    image: null,
    nursingRequirement: null,
    status: null,
    createBy: null,
    updateBy: null,
    remark: null,
    createTime: null,
    updateTime: null
  }
  proxy.resetForm("projectRef")
}

/** 
 * 搜索按钮操作
 * 将当前页码重置为第 1 页并重新查询列表
 */
function handleQuery() {
  queryParams.value.pageNum = 1
  getList()
}

/** 
 * 重置按钮操作
 * 清空搜索表单中的输入项并重新执行查询
 */
function resetQuery() {
  proxy.resetForm("queryRef")
  handleQuery()
}

/** 
 * 多选框选中数据变化事件
 * @param {Array} selection 选中的行数据集合
 * 更新选中的 ID 数组，并根据选中数量控制修改/删除按钮的禁用状态
 */
function handleSelectionChange(selection) {
  ids.value = selection.map(item => item.id)
  single.value = selection.length != 1
  multiple.value = !selection.length
}

/** 
 * 新增按钮操作
 * 重置表单字段，设置对话框标题为“添加护理项目”，并打开弹窗
 */
function handleAdd() {
  reset()
  open.value = true
  title.value = "添加护理项目"
}

/** 
 * 修改按钮操作
 * @param {Object} row 点击当前行的行数据（若从顶部按钮点击，则从 ids 获取选中 ID）
 * 查询选中项目的详情并回显到表单中，打开修改弹窗
 */
function handleUpdate(row) {
  reset()
  const _id = row.id || ids.value
  getProject(_id).then(response => {
    form.value = response.data
    form.value.status = String(form.value.status)
    open.value = true
    title.value = "修改护理项目"
  })
}

/** 
 * 提交按钮操作（保存新增或修改）
 * 对表单进行校验，若校验通过则根据是否包含 id 区分调用更新或新增接口
 */
function submitForm() {
  proxy.$refs["projectRef"].validate(valid => {
    if (valid) {
      if (form.value.id != null) {
        // 存在 ID 时调用修改接口
        updateProject(form.value).then(response => {
          proxy.$modal.msgSuccess("修改成功")
          open.value = false
          getList()
        })
      } else {
        // 不存在 ID 时调用新增接口
        addProject(form.value).then(response => {
          proxy.$modal.msgSuccess("新增成功")
          open.value = false
          getList()
        })
      }
    }
  })
}

/** 
 * 删除按钮操作
 * @param {Object} row 当前行数据（若从顶部操作栏批量删除，则使用 ids.value）
 * 弹出确认框确认后调用删除接口，删除成功后重新拉取列表
 */
function handleDelete(row) {
  const _ids = row.id || ids.value
  proxy.$modal.confirm('是否确认删除护理项目编号为"' + _ids + '"的数据项？').then(function() {
    return delProject(_ids)
  }).then(() => {
    getList()
    proxy.$modal.msgSuccess("删除成功")
  }).catch(() => {})
}


/* 启用禁用按钮操作 */
function handleEnable(row) {
  // 获取当前状态
  const status = row.status;
  // 提示信息
  const msg = status === 1 ? '禁用' : '启用';

  // 构建参数
  const params = {
    id: row.id,
    status: status === 1 ? 0 : 1
  };

  proxy.$modal.confirm(`是否确认${msg}该护理项目的数据项？`).then(function() {
    return updateProject(params);
  }).then(() => {
    getList();
    proxy.$modal.msgSuccess(`${msg}成功`);
  }).catch(() => {});
}

// 页面初始化时自动调用查询列表数据
getList()
</script>

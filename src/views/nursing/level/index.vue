<template>
  <div class="app-container">
    <!-- ==================== 1. 顶部搜索筛选表单区 ==================== -->
    <el-form :model="queryParams" ref="queryRef" :inline="true" v-show="showSearch" label-width="68px">
      <!-- 等级名称搜索框（支持回车快捷搜索） -->
      <el-form-item label="等级名称" prop="name">
        <el-input
            v-model="queryParams.name"
            placeholder="请输入等级名称"
            clearable
            @keyup.enter="handleQuery"
        />
      </el-form-item>

      <!-- 护理计划搜索框（【🌟微实战改造点】：后续可优化为按计划下拉筛选或按状态筛选） -->
      <el-form-item label="状态" prop="status">
        <el-select
            v-model="queryParams.status"
            placeholder="请选择状态"
            clearable
            style="width: 200px"
        >
          <el-option
              v-for="item in nursing_level_status"
              :key="item.value"
              :label="item.label"
              :value="item.value"
          />
        </el-select>
      </el-form-item>

      <!-- 搜索与重置操作按钮 -->
      <el-form-item>
        <el-button type="primary" icon="Search" @click="handleQuery">搜索</el-button>
        <el-button icon="Refresh" @click="resetQuery">重置</el-button>
      </el-form-item>
    </el-form>

    <!-- ==================== 2. 顶部工具栏操作按钮区 ==================== -->
    <el-row :gutter="10" class="mb8">
      <!-- 新增按钮（带权限控制：nursing:level:add） -->
      <el-col :span="1.5">
        <el-button
            type="primary"
            plain
            icon="Plus"
            @click="handleAdd"
            v-hasPermi="['nursing:level:add']"
        >新增</el-button>
      </el-col>








    </el-row>

    <!-- ==================== 3. 核心数据表格区 ==================== -->
    <el-table v-loading="loading" :data="levelList" @selection-change="handleSelectionChange">
      <!-- 多选复选框列 -->
      <el-table-column label="序号" align="center" type="index" width="55" />
      <el-table-column label="等级名称" align="center" prop="name" />

      <!-- 【🌟微实战改造点】：目前显示的是冷冰冰的数字 ID，后续需改成显示“护理计划名称” (prop="planName") -->
      <el-table-column label="护理计划名称" align="center" prop="planId" >
        <template #default="scope">
          <span>{{ formatPlanName(scope.row.planId) }}</span>
        </template>
      </el-table-column>

      <el-table-column label="护理费用" align="center" prop="fee" />

      <!-- 【🌟微实战改造点】：目前直接显示数字 0/1，后续可用 <el-tag> 渲染为绿色的“启用”或红色的“禁用” -->
      <el-table-column label="状态" align="center" prop="status" >
        <template #default="scope">
          <el-tag v-if="scope.row.status === 1" type="success">启用</el-tag>
          <el-tag v-else type="danger">禁用</el-tag>
          </template>
        </el-table-column>



<!--      <el-table-column label="等级说明" align="center" prop="description" />-->

      <!-- 创建时间列（利用若依内置的 parseTime 格式化为 yyyy-MM-dd） -->
      <el-table-column label="创建时间" align="center" prop="createTime" width="180">
        <template #default="scope">
          <span>{{ parseTime(scope.row.createTime, '{y}-{m}-{d} {h}:{i}:{s}') }}</span>
        </template>
      </el-table-column>

      <!-- 行操作列 -->
      <el-table-column label="操作" fixed="right" width="200" align="center" class-name="small-padding fixed-width">
        <template #default="scope">
          <el-button link type="primary" icon="Edit" @click="handleUpdate(scope.row)" v-hasPermi="['nursing:level:edit']">修改</el-button>
          <el-button link type="primary" icon="Delete" @click="handleDelete(scope.row)" v-hasPermi="['nursing:level:remove']">删除</el-button>
          <!-- 【🌟微实战改造点】：在此处可增加一个“启用/禁用”快捷按钮，点击调用状态切换逻辑 -->
          <el-button link type="primary" :icon="scope.row.status == 0 ? 'Lock' : 'Unlock'" @click="handleEnable(scope.row)" >{{ scope.row.status == 0 ? '启用' : '禁用' }}</el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- ==================== 4. 分页组件区 ==================== -->
    <pagination
        v-show="total>0"
        :total="total"
        v-model:page="queryParams.pageNum"
        v-model:limit="queryParams.pageSize"
        @pagination="getList"
    />

    <!-- ==================== 5. 添加或修改护理等级对话框 ==================== -->
    <el-dialog :title="title" v-model="open" width="500px" append-to-body>
      <el-form ref="levelRef" :model="form" :rules="rules" label-width="80px">
        <el-form-item label="等级名称" prop="name">
          <el-input v-model="form.name" placeholder="请输入等级名称" />
        </el-form-item>

        <!-- 【🌟微实战重点改造项】：将这个 input 文本框替换为 <el-select> 下拉框，供用户选择护理计划 -->
        <el-form-item label="护理计划" prop="planId">
          <el-select v-model="form.planId" placeholder="请选择护理计划" style="width: 240px">
            <el-option
                v-for="item in planOptions"
                :key="item.id"
                :label="item.planName"
                :value="item.id"
            />
          </el-select>
        </el-form-item>

        <el-form-item label="护理费用" prop="fee">
          <el-input v-model="form.fee" placeholder="请输入护理费用" />
        </el-form-item>

        <el-form-item label="状态" prop="status">
          <el-radio-group v-model="form.status">
            <el-radio v-for="dict in nursing_level_status"
                      :key="dict.value"
                      :label="dict.value"
                      :value="dict.value">
              {{ dict.label }}
            </el-radio>
          </el-radio-group>
        </el-form-item>

        <el-form-item label="等级说明" prop="description">
          <el-input v-model="form.description" placeholder="请输入等级说明" />
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

<script setup name="Level">
// 导入护理等级模块相关的 API 请求方法
import { listLevel, getLevel, delLevel, addLevel, updateLevel } from "@/api/nursing/level"
// 获取护理计划模块相关的 API 请求方法

import { listPlan } from "@/api/nursing/plan"

// 获取当前组件实例上下文代理（用于访问 $modal, $refs, resetForm, parseTime 等若依全局挂载方法）
const { proxy } = getCurrentInstance()

const { nursing_level_status } = proxy.useDict("nursing_level_status");

const planOptions = ref([]) // 存放从后端拉取的护理计划选项

// ----------------- 响应式状态定义 -----------------
const levelList = ref([])          // 表格数据列表
// 状态下拉选项（0: 禁用, 1: 启用）
const statusOptions = ref([
  { label: '启用', value: 1 },
  { label: '禁用', value: 0 }
])
const open = ref(false)            // 弹窗显隐状态（true: 打开, false: 关闭）
const loading = ref(true)          // 表格遮罩层加载动画状态
const showSearch = ref(true)       // 控制搜索表单区域的显隐
const ids = ref([])                // 表格中勾选选中的数据主键 ID 数组
const single = ref(true)           // “修改”按钮禁用状态（非单选时为 true 禁用）
const multiple = ref(true)         // “删除”按钮禁用状态（未选中任何项时为 true 禁用）
const total = ref(0)               // 数据总条数（用于分页计算）
const title = ref("")              // 弹窗标题（动态显示为“添加护理等级”或“修改护理等级”）

// 表单对象与查询参数响应式容器
const data = reactive({
  form: {}, // 弹窗表单绑定的数据对象
  queryParams: { // 分页与搜索请求参数
    pageNum: 1,
    pageSize: 10,
    name: null,
    planId: null,
  },
  rules: { // 表单项必填与格式校验规则
    name: [
      { required: true, message: "等级名称不能为空", trigger: "blur" }
    ],
    planId: [
      { required: true, message: "护理计划ID不能为空", trigger: "blur" }
    ],
    fee: [
      { required: true, message: "护理费用不能为空", trigger: "blur" }
    ],
    status: [
      { required: true, message: "状态不能为空", trigger: "change" }
    ],
    description: [
      { required: true, message: "等级说明不能为空", trigger: "blur" }
    ],
  }
})

// 将响应式对象转换为解构 ref 引用
const { queryParams, form, rules } = toRefs(data)

// ----------------- 业务逻辑函数 -----------------

/**
 * 功能：查询护理等级列表
 * 逻辑：开启 loading 动画，调用后端分页接口，回显表格数据并绑定总条数
 */
function getList() {
  loading.value = true
  listLevel(queryParams.value).then(response => {
    levelList.value = response.rows
    total.value = response.total
    loading.value = false
  })
}

/**
 * 功能：弹窗“取消”按钮触发
 * 逻辑：关闭弹窗，并清空重置表单数据
 */
function cancel() {
  open.value = false
  reset()
}

/**
 * 功能：重置弹窗表单数据与校验报错
 * 逻辑：将 form 里的各个属性恢复为初始 null 状态，并调用 resetForm 清除红字校验提示
 */
function reset() {
  form.value = {
    id: null,
    name: null,
    planId: null,
    fee: null,
    status: null,
    description: null,
    remark: null,
    createTime: null,
    updateTime: null,
    createBy: null,
    updateBy: null
  }
  proxy.resetForm("levelRef")
}

/**
 * 功能：搜索按钮触发
 * 逻辑：将当前页码强制重置回第一页（避免在靠后页数搜不出结果），并执行查询
 */
function handleQuery() {
  queryParams.value.pageNum = 1
  getList()
}

/**
 * 功能：重置按钮触发
 * 逻辑：清空搜索框中的输入条件，恢复初始状态并重新查询第一页
 */
function resetQuery() {
  proxy.resetForm("queryRef")
  handleQuery()
}

/**
 * 功能：表格多选框勾选状态变更监听
 * 逻辑：
 * 1. 提取所有选中行的 id 存入 ids 数组；
 * 2. single 控制：只有勾选数量刚好等于 1 时，“修改”按钮才可用；
 * 3. multiple 控制：只要有勾选项，“删除”按钮就可用。
 */
function handleSelectionChange(selection) {
  ids.value = selection.map(item => item.id)
  single.value = selection.length != 1
  multiple.value = !selection.length
}

/**
 * 功能：点击顶部“新增”按钮
 * 逻辑：重置表单 -> 设定弹窗标题为“添加护理等级” -> 打开弹窗
 */
function handleAdd() {
  reset()
  open.value = true
  title.value = "添加护理等级"
}

/**
 * 功能：点击“修改”按钮（支持行内按钮或顶部批量修改按钮）
 * 逻辑：获取目标行的 ID -> 调用接口获取该行最新详情数据回显至 form -> 打开弹窗
 */
function handleUpdate(row) {
  reset()
  const _id = row.id || ids.value
  getLevel(_id).then(response => {
    form.value = response.data
    form.value.status = String(form.value.status)
    open.value = true
    title.value = "修改护理等级"
  })
}

/**
 * 功能：弹窗“确定”按钮触发（提交表单）
 * 逻辑：
 * 1. 触发前端校验 rules；
 * 2. 校验通过后，判断是否有 id：有 id 则调用修改接口 updateLevel，无 id 则调用新增接口 addLevel；
 * 3. 提交成功后提示成功消息、关闭弹窗、重新拉取列表刷新页面。
 */
function submitForm() {
  proxy.$refs["levelRef"].validate(valid => {
    if (valid) {
      if (form.value.id != null) {
        updateLevel(form.value).then(response => {
          proxy.$modal.msgSuccess("修改成功")
          open.value = false
          getList()
        })
      } else {
        addLevel(form.value).then(response => {
          proxy.$modal.msgSuccess("新增成功")
          open.value = false
          getList()
        })
      }
    }
  })
}

/**
 * 功能：点击“删除”按钮（支持行内单个删除或顶部批量删除）
 * 逻辑：弹出二次确认提示框 -> 用户确认后调用后端 delLevel 接口 -> 成功后重新拉取列表
 */
function handleDelete(row) {
  const _ids = row.id || ids.value
  proxy.$modal.confirm('是否确认删除护理等级编号为"' + _ids + '"的数据项？').then(function() {
    return delLevel(_ids)
  }).then(() => {
    getList()
    proxy.$modal.msgSuccess("删除成功")
  }).catch(() => {})
}
/**
 * 功能：行内“启用 / 禁用”状态快捷切换
 * 逻辑：
 * 1. 判断当前状态：1 -> 准备禁用为 0；0 -> 准备启用为 1；
 * 2. 弹出二次确认框，防止用户误触；
 * 3. 确认后调用 updateLevel 仅传递 id 和 status，MyBatis-Plus 的 updateById 会自动只更新状态；
 * 4. 成功后弹出轻提示，并重新调用 getList() 刷新表格。
 */
function handleEnable(row) {
  const status = row.status;
  const msg = status === 1 ? '禁用' : '启用';

  // 构建待更新的参数（仅需传递 id 和新的 status）
  const params = {
    id: row.id,
    status: status === 1 ? 0 : 1
  };

  proxy.$modal.confirm(`是否确认${msg}该护理等级？`).then(function() {
    return updateLevel(params);
  }).then(() => {
    getList();
    proxy.$modal.msgSuccess(`${msg}成功`);
  }).catch(() => {});
}
/**
 * 功能：点击“导出”按钮
 * 逻辑：调用若依封装的通用下载工具，将当前查询条件传给后端导出 Excel 文件
 */
function handleExport() {
  proxy.download('nursing/level/export', {
    ...queryParams.value
  }, `level_${new Date().getTime()}.xlsx`)
}

/** 查询所有启用的护理计划 */
function getPlanOptions() {
  // status: 1 只拉取启用的护理计划
  listPlan({ status: 1 }).then(response => {
    planOptions.value = response.rows
  })
}

/** 根据 planId 从下拉列表中匹配出计划名称 */
function formatPlanName(planId) {
  if (!planId || !planOptions.value.length) return planId || '--';
  const target = planOptions.value.find(item => item.id === planId);
  return target ? target.planName : planId;
}

// 页面加载时自动执行
getPlanOptions()
// 页面初次加载时，自动触发一次查询，渲染表格首屏数据
getList()
</script>
<template>
  <div class="app-container">
    <!-- 搜索表单 -->
    <el-form :model="queryParams" ref="queryRef" :inline="true" v-show="showSearch" label-width="68px">
      <el-form-item label="等级名称" prop="name">
        <el-input
          v-model="queryParams.name"
          placeholder="请输入等级名称"
          clearable
          @keyup.enter="handleQuery"
        />
      </el-form-item>
      <el-form-item label="状态" prop="status">
        <el-select v-model="queryParams.status" placeholder="请选择状态" clearable>
          <el-option
            v-for="dict in nursing_level_status"
            :key="dict.value"
            :label="dict.label"
            :value="dict.value"
          />
        </el-select>
      </el-form-item>
      <el-form-item>
        <el-button type="primary" icon="Search" @click="handleQuery">搜索</el-button>
        <el-button icon="Refresh" @click="resetQuery">重置</el-button>
      </el-form-item>
    </el-form>

    <!-- 按钮：新增、修改、删除、导出 -->
    <el-row :gutter="10" class="mb8">
      <el-col :span="1.5">
        <el-button
          type="primary"
          plain
          icon="Plus"
          @click="handleAdd"
          v-hasPermi="['serve:level:add']"
        >新增</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="success"
          plain
          icon="Edit"
          :disabled="single"
          @click="handleUpdate"
          v-hasPermi="['serve:level:edit']"
        >修改</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="danger"
          plain
          icon="Delete"
          :disabled="multiple"
          @click="handleDelete"
          v-hasPermi="['serve:level:remove']"
        >删除</el-button>
      </el-col>
      <el-col :span="1.5">
        <el-button
          type="warning"
          plain
          icon="Download"
          @click="handleExport"
          v-hasPermi="['serve:level:export']"
        >导出</el-button>
      </el-col>
      <right-toolbar v-model:showSearch="showSearch" @queryTable="getList"></right-toolbar>
    </el-row>

    <!-- 数据列表 -->
    <el-table
        v-loading="loading"
        :data="levelList"
        @selection-change="handleSelectionChange"
        show-overflow-tooltip
        height="calc(100vh - 320px)">
      <el-table-column type="selection" fixed="left" align="center"  width="55" />
      <el-table-column label="序号" align="center" fixed="left" type="index" width="55" />
      <el-table-column label="等级名称" align="center" fixed="left" prop="name" width="150" />
      <el-table-column label="护理费用" align="center" prop="fee" />
      <el-table-column label="状态" align="center" prop="status">
        <template #default="scope">
          <dict-tag :options="nursing_level_status" :value="scope.row.status"/>
        </template>
      </el-table-column>
      <el-table-column label="等级说明" align="center" prop="description" />
      <el-table-column label="更新时间" align="center" prop="updateTime" width="180">
        <template #default="scope">
          <span>{{ parseTime(scope.row.updateTime, '{y}-{m}-{d} {h}:{i}:{s}') }}</span>
        </template>
      </el-table-column>
      <el-table-column label="备注" align="center" prop="remark" />
      <el-table-column label="操作" align="center" class-name="small-padding fixed-width" width="250" fixed="right">
        <template #default="scope">
          <el-button link type="primary" icon="Edit" @click="handleUpdate(scope.row)" v-hasPermi="['serve:level:edit']">修改</el-button>
          <el-button link type="danger" icon="Delete" @click="handleDelete(scope.row)" v-hasPermi="['serve:level:remove']">删除</el-button>
          <!-- 状态启禁用按钮 -->
          <el-button 
            link :type="scope.row.status == 0 ? 'success' : 'warning'" 
            :icon="scope.row.status == 0 ? 'SuccessFilled' : 'CircleCloseFilled'" 
            @click="handleEnable(scope.row)">
            {{ scope.row.status == 0 ? '启用' : '禁用' }}
          </el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- 分页栏 -->
    <pagination
      v-show="total>0"
      :total="total"
      v-model:page="queryParams.pageNum"
      v-model:limit="queryParams.pageSize"
      @pagination="getList"
    />

    <!-- 添加或修改护理等级对话框 -->
    <el-dialog :title="title" v-model="open" width="500px" append-to-body>
      <el-form ref="levelRef" :model="form" :rules="rules" label-width="95px">
        <el-form-item label="等级名称" prop="name">
          <el-input v-model="form.name" placeholder="请输入等级名称" />
        </el-form-item>
            <el-form-item label="护理计划" prop="lplanId">
              <el-select v-model="form.lplanId" placeholder="请选择护理计划" filterable clearable>
                <el-option
                  v-for="plan in planList"
                  :key="plan.value"
                  :label="plan.label"
                  :value="plan.value"
                />
              </el-select>
            </el-form-item>
        <el-form-item label="护理费用" prop="fee">
          <!-- <el-input v-model="form.fee" placeholder="请输入护理费用" /> -->
          <el-input-number v-model="form.fee" :min="0" :max="10000"></el-input-number>
        </el-form-item>
        <el-form-item label="状态" prop="status">
          <el-radio-group v-model="form.status">
            <el-radio
            v-for="dict in nursing_level_status"
            :key="dict.value"
            :label="dict.value"
            size="large">
              {{ dict.label }}
            </el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="等级说明" prop="description">
          <el-input v-model="form.description" placeholder="请输入等级说明" />
        </el-form-item>
        <el-form-item label="备注" prop="remark">
          <el-input v-model="form.remark" placeholder="请输入备注" type="textarea" />
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
import { listLevel, getLevel, delLevel, addLevel, updateLevel, updateStatus } from "@/api/serve/level";
import { getAllPlan } from "@/api/serve/plan";

const { proxy } = getCurrentInstance();
const { nursing_level_status } = proxy.useDict('nursing_level_status');

const levelList = ref([]);
const planList = ref([]);
const open = ref(false);
const loading = ref(true);
const showSearch = ref(true);
const ids = ref([]);
const single = ref(true);
const multiple = ref(true);
const total = ref(0);
const title = ref("");

const data = reactive({
  form: {},
  queryParams: {
    pageNum: 1,
    pageSize: 10,
    name: null,
    status: null,
  },
  rules: {
    name: [
      { required: true, message: "等级名称不能为空", trigger: "blur" }
    ],
    lplanId: [
      { required: true, message: "护理计划ID不能为空", trigger: "blur" }
    ],
    fee: [
      { required: true, message: "护理费用不能为空", trigger: "blur" }
    ],
  }
});

const { queryParams, form, rules } = toRefs(data);

/** 查询护理等级列表 */
function getList() {
  loading.value = true;
  listLevel(queryParams.value).then(response => {
    levelList.value = response.rows;
    total.value = response.total;
    loading.value = false;
  });
}

// 取消按钮
function cancel() {
  open.value = false;
  reset();
}

// 表单重置
function reset() {
  form.value = {
    id: null,
    name: null,
    lplanId: null,
    fee: null,
    status: null,
    description: null,
    createTime: null,
    createBy: null,
    updateBy: null,
    remark: null,
    updateTime: null
  };
  proxy.resetForm("levelRef");
}

/** 搜索按钮操作 */
function handleQuery() {
  queryParams.value.pageNum = 1;
  getList();
}

/** 重置按钮操作 */
function resetQuery() {
  proxy.resetForm("queryRef");
  handleQuery();
}

// 多选框选中数据
function handleSelectionChange(selection) {
  ids.value = selection.map(item => item.id);
  single.value = selection.length != 1;
  multiple.value = !selection.length;
}

/** 新增按钮操作 */
function handleAdd() {
  reset();
  open.value = true;
  title.value = "添加护理等级";
}

/** 修改按钮操作 */
function handleUpdate(row) {
  reset();
  const _id = row.id || ids.value
  getLevel(_id).then(response => {
    form.value = response.data;
    form.value.status = form.value.status.toString();
    form.value.lplanId = form.value.lplanId.toString();
    open.value = true;
    title.value = "修改护理等级";
  });
}

/** 提交按钮 */
function submitForm() {
  proxy.$refs["levelRef"].validate(valid => {
    if (valid) {
      if (form.value.id != null) {
        updateLevel(form.value).then(response => {
          proxy.$modal.msgSuccess("修改成功");
          open.value = false;
          getList();
        });
      } else {
        addLevel(form.value).then(response => {
          proxy.$modal.msgSuccess("新增成功");
          open.value = false;
          getList();
        });
      }
    }
  });
}

/** 删除按钮操作 */
function handleDelete(row) {
  const _ids = row.id || ids.value;
  proxy.$modal.confirm('是否确认删除护理等级编号为"' + _ids + '"的数据项？').then(function() {
    return delLevel(_ids);
  }).then(() => {
    getList();
    proxy.$modal.msgSuccess("删除成功");
  }).catch(() => {});
}

/** 状态启禁用按钮操作 */
function handleEnable(row) {
  const _id = row.id || ids.value;
  const newStatus = row.status == 0 ? 1 : 0; // 切换状态
  const action = newStatus == 0 ? '禁用' : '启用';
  proxy.$modal.confirm(`是否确认${action}${row.name}？`).then(function() {
    return updateStatus(_id);
  }).then(() => {
    getList();
    proxy.$modal.msgSuccess(`${action}成功`);
  }).catch(() => {});
}

/** 导出按钮操作 */
function handleExport() {
  proxy.download('serve/level/export', {
    ...queryParams.value
  }, `level_${new Date().getTime()}.xlsx`)
}

/** 获取护理计划列表 */
function getPlanList() {
  getAllPlan().then(response => {
    const list = response.data || response.rows || [];
    planList.value = list.map(p => ({ label: p.label, value: String(p.value) }));
  }).catch(() => {
    planList.value = [];
  });
}

getList();
getPlanList();
</script>

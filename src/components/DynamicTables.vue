<template>
  <el-row :gutter="0" style="padding-bottom: 5px">
    <el-col :span="23">
      <LayoutSelector
        v-model="selectLayout"
        :options="optionsLayout"
        :max-visible="maxActions"
        @change="handleLayoutChange"
      />
    </el-col>
    <el-col :span="1" style="display: flex; justify-content: flex-end">
      <el-button @click="handleOpen" type="primary" circle>
        <el-icon><Setting /></el-icon>
      </el-button>
    </el-col>
  </el-row>

  <Table
    :fixed="props?.fixed"
    :height="props?.height"
    :columns="columns"
    :templates="[...vfFields, ...icons, ...actions, ...textFields]"
    :data="props.dataTable"
    @onCta="onCta"
  />

  <el-dialog
    v-model="dialogVisible"
    :title="props.settingsTitle || 'Cài đặt hiển thị'"
    width="calc(100vw - 100px)"
    top="25px"
    style="max-width: 1440px; height: 720px; overflow: hidden"
    @close="handleCloseDialog"
  >
    <el-scrollbar height="700px">
      <Table
        fixed
        :height="200"
        :columns="dialogColumns"
        :templates="[...vfFields, ...icons, ...actions, ...textFields]"
        :data="props.dataTable"
        @onCta="onCta"
      />
      <div class="action-box">
        <p style="font-weight: bold; font-size: 14px; color: #606266">
          Giao diện:
        </p>
        <el-select
          size="small"
          v-model="selectedTemplate"
          placeholder="Chọn giao diện"
          class="action-select"
          style="width: 230px"
          @change="handleTemplateChange"
        >
          <el-option
            v-for="item in props.optionsLayout"
            :key="item.value"
            :label="item.label"
            :value="item.value"
          />
        </el-select>

        <div class="button-group">
          <!-- Default Save button -->
          <el-button
            size="small"
            type="success"
            :disabled="!hasChanges"
            @click="handleSave"
            class="action-button"
          >
            <el-icon><Check /></el-icon>
            <span>Lưu</span>
          </el-button>
          <!-- Add Delete button -->
          <el-button
            size="small"
            type="danger"
            :disabled="!canDelete"
            @click="handleDelete"
            class="action-button"
          >
            <el-icon><Delete /></el-icon>
            <span>Xóa</span>
          </el-button>
        </div>
      </div>

      <EditorTable
        v-model="dialogColumns"
        :vfFields="vfFields"
        :actions="actions"
        :icons="icons"
        :height="360"
        :textFields="props.textFields"
        @onText="onTextChange"
      />
    </el-scrollbar>
  </el-dialog>

  <!-- Error dialog -->
  <el-dialog v-model="showError" title="Lỗi" width="400px" :show-close="false">
    <span>{{ errorMessage }}</span>
    <template #footer>
      <span class="dialog-footer">
        <el-button type="primary" @click="showError = false"> Đóng </el-button>
      </span>
    </template>
  </el-dialog>
</template>

<script setup lang="ts">
import { ref, computed } from "vue";
import { VfField, VfType, Column, OptionLayout } from "@/interfaces/table";
import { Check, Delete, SetUp, Setting } from "@element-plus/icons-vue";
import { cloneDeep } from "lodash";
import EditorTable from "./EditorTable.vue";
import Table from "./Table.vue";
import LayoutSelector from "./LayoutSelector.vue";

interface Props {
  optionsLayout: OptionLayout[];
  defaultValue?: string | number;
  maxActions?: number;
  dataTable: any[];
  columns: Column[];
  actions: VfField[];
  icons: VfField[];
  vfFields: VfField[];
  textFields: VfField[];
  dialogVisible: boolean;
  settingsTitle?: string;
  height?: number;
  fixed?: boolean;
}

const emit = defineEmits<{
  (e: "update:columns", columns: Column[]): void;
  (e: "handleSave", value: OptionLayout, columns: Column[]): void;
  (e: "handleDelete", value: OptionLayout): void;
  (e: "handleDefault", value: OptionLayout): void;
  (e: "changeLayout", value: OptionLayout): void;
  (e: "handleOpen", value: boolean): void;
  (e: "handleClose", value: boolean): void;
  (e: "handleTemplateChange", value: OptionLayout): void;
  (e: "onCta", action: string, row: any, index: number): void;
  (e: "onText", data: VfField[]): void;
}>();

const props = withDefaults(defineProps<Props>(), {
  maxActions: 6,
  height: 390,
  fixed: false,
});

// Track changes
const initialColumns = ref<Column[]>([]);
const selectedTemplate = ref<any>(props.defaultValue || props.optionsLayout[0]);
const selectLayout = ref<any>(props.defaultValue || props.optionsLayout[0]);
const dialogVisible = ref(props.dialogVisible);
const showError = ref(false);
const errorMessage = ref("");
const isNewLayout = ref(false);
let lastSavedTemplate: OptionLayout | null = null;
// Tách biệt state cho dialog
const dialogColumns = ref<Column[]>([]);

// Initialize change tracking when dialog opens
const initializeChangeTracking = () => {
  dialogColumns.value = cloneDeep(
    props.columns.map((column) => ({
      ...column,
      isDrag: false,
    }))
  );
  initialColumns.value = cloneDeep(dialogColumns.value);
  lastSavedTemplate = cloneDeep(selectedTemplate.value);
  isNewLayout.value = false;
};

// Modify hasChanges computed to include deletion state
const hasChanges = computed(() => {
  if (isNewLayout.value) return true;

  // Check if template has changed
  if (
    JSON.stringify(selectedTemplate.value) !== JSON.stringify(lastSavedTemplate)
  ) {
    return true;
  }

  // Check if columns have changed
  const columnsChanged =
    JSON.stringify(dialogColumns.value) !==
    JSON.stringify(initialColumns.value);

  return columnsChanged;
});

// Check if layout can be deleted
const canDelete = computed(() => {
  const currentTemplate = selectedTemplate.value;

  if (props.optionsLayout.length <= 1) {
    return false;
  }

  if (currentTemplate.isDefault) {
    return false;
  }

  return true;
});

// Simplified handleDelete
const handleDelete = () => {
  if (!canDelete.value) {
    errorMessage.value = selectedTemplate.value.isDefault
      ? "Không thể xóa giao diện mặc định"
      : "Phải có ít nhất một giao diện";
    showError.value = true;
    return;
  }

  emit("handleDelete", selectedTemplate.value);

  // After deletion, select the first available template
  const remainingTemplates = props.optionsLayout.filter(
    (template) => template.value !== selectedTemplate.value.value
  );

  if (remainingTemplates.length > 0) {
    selectedTemplate.value = remainingTemplates[0];
    selectLayout.value = remainingTemplates[0];
    handleTemplateChange(remainingTemplates[0].value);
  }

  isNewLayout.value = false; // Reset new layout flag
};

const handleSave = () => {
  if (!hasChanges.value) return;

  // 1. Emit sự kiện save với template và columns mới
  emit("update:columns", dialogColumns.value);
  emit("handleSave", selectedTemplate.value, dialogColumns.value);

  // 2. Cập nhật lại initialColumns để tracking thay đổi tiếp theo
  initialColumns.value = cloneDeep(dialogColumns.value);

  // 3. Cập nhật lastSavedTemplate
  lastSavedTemplate = cloneDeep(selectedTemplate.value);

  // 4. Reset trạng thái isNewLayout
  isNewLayout.value = false;
};

const handleDefault = () => {
  emit("handleDefault", selectedTemplate.value);
};

const onTextChange = (data: VfField[]) => {
  emit("onText", data);
};

const displayedButtons = computed(() => {
  return props.optionsLayout.slice(0, props.maxActions);
});

const handleOpen = () => {
  dialogVisible.value = true;
  dialogColumns.value = cloneDeep(
    props.columns.map((column) => ({
      ...column,
      isDrag: false,
    }))
  );
};
emit("handleOpen", true);
initializeChangeTracking();

const handleCloseDialog = () => {
  closeDialog();
};

const closeDialog = () => {
  dialogVisible.value = false;
  if (hasChanges.value) {
    dialogColumns.value = cloneDeep(initialColumns.value);
    selectedTemplate.value = cloneDeep(lastSavedTemplate);
  }
  emit("handleClose", false);
};

const isActive = computed(() => {
  return (value: string | number) => {
    return selectLayout.value.value === value;
  };
});

// Update handleTemplateChange to mark changes
const handleTemplateChange = (value: string | number) => {
  const selectedOption = props.optionsLayout.find(
    (item) => item.value === value
  );
  if (selectedOption) {
    // Check if this is a new template
    isNewLayout.value = !props.optionsLayout.some(
      (layout) => layout.value === selectedOption.value
    );
    selectedTemplate.value = selectedOption;
    emit("handleTemplateChange", selectedOption);
  }
};

const handleLayoutChange = (value: string | number) => {
  const selectedOption = props.optionsLayout.find(
    (item) => item.value === value
  );
  if (selectedOption) {
    selectLayout.value = selectedOption;
    selectedTemplate.value = selectedOption;
    emit("changeLayout", selectedOption);
  }
};

const onCta = (action: string, row: any, index: number) => {
  emit("onCta", action, row, index);
};
</script>

<style lang="scss" scoped>
.action-box {
  display: inline-flex;
  align-items: center;
  padding: 8px 16px;
  padding-left: 0;
  gap: 5px;
  flex-wrap: wrap;

  .button-group {
    display: flex;
    gap: 5px;
    align-items: center;
  }

  .el-icon {
    margin-right: 0px;
    font-size: 15px;
  }

  :deep(.el-button + .el-button) {
    margin-left: 5px;
  }

  span {
    font-size: 13px;
  }
}

.dialog-footer {
  display: flex;
  justify-content: flex-end;
  width: 100%;
}
</style>

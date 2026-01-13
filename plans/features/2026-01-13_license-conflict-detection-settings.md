# [功能] 深度许可证设置拆分与许可证冲突检测设置

## ✅ 关键确认事项

以下事项已与需求方确认：

1. **后端字段名称**: `excluded_paths` 为暂定字段名，后续对接后端时可能调整
2. **路径格式**: 与 `codetraceForm.scanning_files` 保持一致
   - 使用分号 `;` 分隔多个路径
   - 支持相对路径和绝对路径
   - 通过 `renderScanningFiles` 方法处理
3. **表单验证**: 与 `codetraceForm.scanning_files` 保持一致
   - `excluded_paths` 为非必填字段
   - 在 `validateAllForms` 中调用 `validate()` 以保持代码一致性
4. **显示/隐藏逻辑**: "深度许可证检测设置"和"许可证冲突检测设置"保持一致
   - 用户选择"深度许可证"扫描引擎时，两个区域都显示
   - 用户取消选择时，两个区域都隐藏
   - 不需要独立控制

## 📝 背景与问题描述

### 问题现象

当前扫描设置中，DeepLicense 引擎的设置区域标题为"深度许可证及冲突检测设置"，将两种不同的检测功能合并在一起，不够清晰。需要将其拆分为两个独立的设置区域：

1. 深度许可证检测设置（DeepLicense Settings）
2. 许可证冲突检测设置（License Conflict Detection Settings）

### 功能需求

1. 修改 DeepLicense 设置区域标题为"深度许可证检测设置"
2. 新增"许可证冲突检测设置"区域，显示与隐藏逻辑与 DeepLicense 一致
   - 用户选择"深度许可证"扫描引擎时，两个设置区域都显示
   - 用户取消选择时，两个设置区域都隐藏
3. 在"许可证冲突检测设置"中新增"文件级许可证路径屏蔽列表"设置项
4. 该设置项为非必填的文本输入框，样式和行为参考 `codetraceForm.scanning_files` 字段
   - 多行文本输入框（textarea）
   - 最大长度 500 字符
   - 显示字数统计
   - 使用分号 `;` 分隔多个路径
5. 输入框下方添加说明文字："该路径下的文件级许可证不参与许可证冲突检测，降低未被正式使用的许可证文本引起的误报。"
6. 使用 `renderScanningFiles` 方法将用户输入处理为数组，参与实际请求
7. 后端字段名暂定为 `excluded_paths`，后续对接后端时可能调整

### 影响范围

- 文件：`src/views/project/ScanModal.vue`
- 国际化文件（需手动编辑）：
  - `src/language/cn.js` (中文翻译)
  - `src/language/en.js` (英文翻译)
- 影响模块：DeepLicense 扫描设置界面

### 优先级

P1（高优先级，UI 和功能优化）

## 🔍 上下文信息

### 涉及文件详情

#### 1. src/views/project/ScanModal.vue

**文件路径**: `f:/Code/vue3-frontend/src/views/project/ScanModal.vue`

**当前 DeepLicense 设置区域位置**:

- 模板部分：第 217-272 行
  - 标题：第 222 行 `{{ t("DEEPLICENSE_SETTINGS") }}`
  - 表单引用：第 224 行 `deeplicenseFormRef`
  - 表单项：
    - 第 226-240 行：CWE 检查器提示信息
    - 第 242-245 行：扫描深度输入
    - 第 247-249 行：白名单规则输入
    - 第 251-255 行：项目许可证名称输入
    - 第 257-269 行：许可证文件上传

**相关代码位置**:

- **DeepLicense 表单定义**: 第 583-592 行

  ```javascript
  const deeplicenseFormRef = ref<FormInstance>();
  const deeplicenseForm = reactive({
    isCheckAll: true,
    deep: 999,
    white_list: "(license|copying|eula|patent|notice|legal|disclaimer|gpl|mit|apache|bsd)",
    project_license_name: "",
    fileList: [],
    license_text: "",
  });
  ```

- **表单验证规则**: 第 621-638 行

  ```javascript
  const deeplicenseRules = reactive<FormRules>({
    deep: [...],
    white_list: [...],
  });
  ```

- **扫描引擎选择状态**: 第 509-517 行

  ```javascript
  const scanEngineSelectionStatus = reactive({
    sca: false,
    sast: false,
    iac: false,
    fuzzing: false,
    codetrace: false,
    sast_binary: false,
    deeplicense: false, // DeepLicense 显示/隐藏控制
  });
  ```

- **引擎扫描类型映射**: 第 681-689 行

  ```javascript
  const engineScanTypes = reactive({
    sca: "source_code",
    sast: "source_code",
    iac: "source_code",
    fuzzing: "binary",
    codetrace: "source_code",
    sast_binary: "binary",
    deeplicense: "source_code", // DeepLicense 扫描类型
  });
  ```

- **生成扫描参数**: 第 989-1005 行
  ```javascript
  if (scanEngineSelectionStatus.deeplicense) {
    const payload = {
      project_id: project.id,
      source: providerToSource(project.provider),
      scan_type: engineScanTypes.deeplicense,
      engine_type: "DeepLicense",
      deep_license_scan_parameter: {
        ...createBaseScanParameter(project),
        max_deep: deeplicenseForm.deep,
        perform_deep_license: deeplicenseForm.isCheckAll,
        filename_white_list_regex: deeplicenseForm.white_list,
        project_license_name: deeplicenseForm.project_license_name,
        license_text: deeplicenseForm.license_text,
      },
    };
    payloadList.push(payload);
  }
  ```

**参考的样式代码**:

- **CodeTrace 文件路径设置**: 第 204-213 行

  ```vue
  <el-form-item :label="t('LIST_OF_FILES_OR_FOLDERS_FOR_SCANNING')">
    <el-radio-group v-model="codetraceForm.scanning_file_action">
      <el-radio label="inclusive">{{ t("INCLUSIVE") }}</el-radio>
      <el-radio label="exclusive">{{ t("EXCLUSIVE") }}</el-radio>
    </el-radio-group>
  </el-form-item>
  <el-form-item>
    <el-input v-model="codetraceForm.scanning_files" type="textarea" :rows="3" maxlength="500" show-word-limit
      :placeholder="t('LIST_FILES_OR_FOLDERS_PATH')" />
  </el-form-item>
  ```

- **警告提示样式**: 第 233-238 行
  ```vue
  <div class="flex flex-row" v-if="showCweCheckerDesc">
    <i class="fa-solid fa-triangle-exclamation"></i>
    <el-txt type="subtitle2" class="title-text">
      {{ t("SAST_BINARY_CWE_CHECKER_DESC") }}
    </el-txt>
  </div>
  ```

**工具方法位置**:

- **renderScanningFiles**: 第 879-883 行
  ```javascript
  function renderScanningFiles(filePath) {
    if (!filePath) return [];
    const file = filePath?.replace(/(\n|\r|\r\n|↵)/g, "");
    return file.split(";").filter((item) => item != "");
  }
  ```

**表单重置位置**:

- **resetForm**: 第 826-843 行
- **handleCancel**: 第 868-877 行
- **validateAllForms**: 第 1133-1177 行

#### 2. 国际化文件（需手动编辑）

**中文翻译文件**: `src/language/cn.js` (约 169 KB)
**英文翻译文件**: `src/language/en.js` (约 182 KB)

**需要添加的翻译键**:

- `DEEP_LICENSE_SETTINGS`: "深度许可证检测设置"
- `LICENSE_CONFLICT_SETTINGS`: "许可证冲突检测设置"
- `LICENSE_CONFLICT_EXCLUDED_PATHS`: "文件级许可证路径屏蔽列表"
- `LICENSE_CONFLICT_EXCLUDED_PATHS_HINT`: "该路径下的文件级许可证不参与许可证冲突检测，降低未被正式使用的许可证文本引起的误报。"

**翻译文件格式示例**（基于 cn.js 前 100 行）:

```javascript
export default {
  ABOUT: "关于",
  ACCESS_TOKEN: "访问令牌",
  // ...
};
```

### 技术栈说明

- **框架**: Vue 3 (Composition API with `<script setup>`)
- **UI 组件库**: Element Plus
- **状态管理**: Pinia (stores: compliance, project, scan, org)
- **类型系统**: TypeScript
- **表单验证**: Element Plus FormRules

### 显示/隐藏逻辑

DeepLicense 设置区域的显示由 `scanEngineSelectionStatus.deeplicense` 控制：

- 值为 `true` 时：显示"深度许可证检测设置"和"许可证冲突检测设置"两个区域
- 值为 `false` 时：隐藏两个设置区域

新增的"许可证冲突检测设置"区域与"深度许可证检测设置"区域使用相同的显示/隐藏逻辑，保持一致性。

### 表单验证规则

- **DeepLicense 表单**: `deep` 和 `white_list` 为必填字段
- **许可证冲突检测表单**: `excluded_paths` 为非必填字段
  - 参考 `codetraceForm.scanning_files` 的实现（第 692-708 行），无需验证规则
  - 但需要在 `validateAllForms` 中调用 `validate()` 以保持代码一致性

### 路径格式说明

参考 `codetraceForm.scanning_files` 的实现：

- **分隔符**: 使用分号 `;` 分隔多个路径
- **路径格式**: 支持相对路径和绝对路径
- **示例输入**:
  - 单个路径: `src/licenses`
  - 多个路径: `src/licenses;docs/legal;vendor/third-party`
  - 带换行的输入会被自动处理（换行符会被移除）
- **处理逻辑**: 通过 `renderScanningFiles` 方法处理（第 879-883 行）
  - 移除所有换行符（`\n`, `\r`, `\r\n`, `↵`）
  - 按分号分割
  - 过滤空字符串
  - 返回字符串数组

## 📌 关键代码修改点速查

### 需要修改的代码位置汇总

| 修改类型     | 行号范围        | 具体操作                                                                     |
| ------------ | --------------- | ---------------------------------------------------------------------------- |
| 新增表单数据 | 第 583-592 行后 | 添加 `licenseConflictForm`、`licenseConflictFormRef`、`licenseConflictRules` |
| 修改标题     | 第 222 行       | 修改为 `t("DEEP_LICENSE_SETTINGS")`                                          |
| 新增 UI 区域 | 第 272 行后     | 添加许可证冲突检测设置区域                                                   |
| 修改数据生成 | 第 989-1005 行  | 在 `deep_license_scan_parameter` 中添加 `excluded_paths`                     |
| 修改表单重置 | 第 826-843 行   | 在 `resetForm` 中调用 `resetLicenseConflictSettings()`                       |
| 修改表单取消 | 第 868-877 行   | 在 `handleCancel` 中重置许可证冲突表单                                       |
| 修改表单验证 | 第 1133-1177 行 | 在 `validateAllForms` 中添加许可证冲突表单验证                               |
| 新增重置函数 | 第 1216 行后    | 添加 `resetLicenseConflictSettings` 函数                                     |
| 添加样式     | 第 1339 行后    | 添加 `&__license-conflict-settings-container` 样式                           |
| 编辑国际化   | cn.js, en.js    | 手动添加 4 个翻译键（需用户手动操作）                                        |

### 核心数据流向

```
用户输入 (licenseConflictForm.excluded_paths)
    ↓
renderScanningFiles() 方法处理
    ↓
转换为数组 ["path1", "path2", ...]
    ↓
添加到 deep_license_scan_parameter.excluded_paths
    ↓
随 payloadList 发送到后端
```

## 🎯 解决方案

### 技术选型

- 方案：在现有 DeepLicense 设置基础上，拆分为两个独立的表单区域，共享同一个 `scanEngineSelectionStatus.deeplicense` 状态控制显示/隐藏

### 实施步骤

#### 1. 新增许可证冲突检测表单数据（第 583-592 行附近）

在 `deeplicenseForm` 定义后，新增：

```javascript
// 许可证冲突检测表单（参考 codetraceForm 实现）
const licenseConflictFormRef = ref<FormInstance>();
const licenseConflictForm = reactive({
  excluded_paths: "", // 文件级许可证路径屏蔽列表，参考 codetraceForm.scanning_files
});
const licenseConflictRules = reactive<FormRules>({
  // 非必填，无需验证规则（与 codetraceForm.scanning_files 一致）
});
});
```

#### 2. 修改 DeepLicense 设置区域标题（第 217-272 行）

- 将第 222 行的标题从 `{{ t("DEEPLICENSE_SETTINGS") }}` 改为 `{{ t("DEEP_LICENSE_SETTINGS") }}`
- 保持现有的 DeepLicense 表单字段不变

#### 3. 新增许可证冲突检测设置区域（第 272 行后）

在第 272 行 `</div>` 后、第 273 行注释前，新增：

```vue
      <!-- 许可证冲突检测设置 -->
      <el-divider v-if="scanEngineSelectionStatus.deeplicense" />

      <div v-if="scanEngineSelectionStatus.deeplicense" class="SCAScanModal__license-conflict-settings-container">
        <el-txt type="subtitle1" class="title-text">
          {{ t("LICENSE_CONFLICT_SETTINGS") }}
        </el-txt>
        <el-form ref="licenseConflictFormRef" :model="licenseConflictForm" label-position="top"
          @submit.native.prevent>
          <el-form-item :label="t('LICENSE_CONFLICT_EXCLUDED_PATHS')">
            <el-input v-model="licenseConflictForm.excluded_paths" type="textarea" :rows="3" maxlength="500"
              show-word-limit :placeholder="t('LIST_FILES_OR_FOLDERS_PATH')" />
          </el-form-item>

          <el-form-item>
            <div class="flex flex-row">
              <i class="fa-solid fa-circle-info" style="color: var(--el-color-info); margin-right: 8px;"></i>
              <el-txt type="subtitle2" class="title-text">
                {{ t("LICENSE_CONFLICT_EXCLUDED_PATHS_HINT") }}
              </el-txt>
            </div>
          </el-form-item>
        </el-form>
      </div>
```

#### 4. 修改数据生成逻辑（第 989-1005 行）

在 `deep_license_scan_parameter` 对象中添加 `excluded_paths` 字段（字段名暂定，后续对接后端时可能调整）：

```javascript
deep_license_scan_parameter: {
  ...createBaseScanParameter(project),
  max_deep: deeplicenseForm.deep,
  perform_deep_license: deeplicenseForm.isCheckAll,
  filename_white_list_regex: deeplicenseForm.white_list,
  project_license_name: deeplicenseForm.project_license_name,
  license_text: deeplicenseForm.license_text,
  excluded_paths: renderScanningFiles(licenseConflictForm.excluded_paths), // 新增，参考 codetraceForm.scanning_files 处理方式
},
```

#### 5. 修改表单重置和验证逻辑

- **resetForm 函数**（第 826-843 行）：在 `resetSastBinarySettings();` 后添加：

  ```javascript
  resetLicenseConflictSettings();
  ```

- **handleCancel 函数**（第 868-877 行）：在 `deeplicenseFormRef.value.resetFields();` 后添加：

  ```javascript
  if (licenseConflictFormRef.value) {
    licenseConflictFormRef.value.resetFields();
    licenseConflictFormRef.value.clearValidate();
  }
  ```

- **validateAllForms 函数**（第 1133-1177 行）：在 `Promise.all` 中添加：

  ```javascript
  licenseConflictFormRef.value?.validate(),
  ```

  并在解构数组中添加 `licenseConflictValid`

  **说明**: 虽然 `licenseConflictRules` 为空（无验证规则），但为了保持代码一致性，所有表单都需要调用 `validate()`（参考 `codetraceForm` 的处理方式）

- **新增 resetLicenseConflictSettings 函数**（第 1216 行后）：
  ```javascript
  function resetLicenseConflictSettings() {
    licenseConflictForm.excluded_paths = "";
  }
  ```

#### 6. 添加样式（第 1339 行附近）

在 `&__deeplicense-settings-container` 后添加：

```scss
&__license-conflict-settings-container {
  display: flex;
  flex-direction: column;
  gap: 12px;

  ::v-deep(.el-input__inner) {
    text-align: left;
  }
}
```

#### 7. 新增国际化翻译键（需手动编辑）

⚠️ **重要提示**：国际化文件较大（cn.js 约 169KB，en.js 约 182KB），为了避免编辑错误，此步骤需要用户手动编辑。

**需要编辑的文件**:

1. `src/language/cn.js` - 在合适位置添加以下翻译键
2. `src/language/en.js` - 在对应位置添加以下英文翻译

**需要添加的翻译键**:

```javascript
// cn.js（中文）
DEEP_LICENSE_SETTINGS: "深度许可证检测设置",
LICENSE_CONFLICT_SETTINGS: "许可证冲突检测设置",
LICENSE_CONFLICT_EXCLUDED_PATHS: "文件级许可证路径屏蔽列表",
LICENSE_CONFLICT_EXCLUDED_PATHS_HINT: "该路径下的文件级许可证不参与许可证冲突检测，降低未被正式使用的许可证文本引起的误报。",

// en.js（英文）
DEEP_LICENSE_SETTINGS: "Deep License Detection Settings",
LICENSE_CONFLICT_SETTINGS: "License Conflict Detection Settings",
LICENSE_CONFLICT_EXCLUDED_PATHS: "File-Level License Path Exclusion List",
LICENSE_CONFLICT_EXCLUDED_PATHS_HINT: "File-level licenses in this path will not participate in license conflict detection, reducing false positives caused by license texts not formally used.",
```

**建议位置**：可以按字母顺序或功能分组，建议添加在其他 LICENSE 或 DEEP 相关的翻译键附近。

**关于旧键 `DEEPLICENSE_SETTINGS` 的处理**:

- 新增 `DEEP_LICENSE_SETTINGS` 键用于新的标题
- 旧的 `DEEPLICENSE_SETTINGS` 键可能在其他地方使用，建议保留以保持向后兼容
- 如果确认旧键未在其他地方使用，可以考虑删除

### 影响评估

- **影响模块**：ScanModal.vue 组件的 DeepLicense 设置区域
- **多品牌兼容性**：需要确认是否所有品牌都启用 DeepLicense 功能，如有差异需要条件判断
- **性能影响**：无明显性能影响，仅UI调整和字段新增

## ✅ 验证计划

### 功能测试

- [ ] 验证 DeepLicense 设置区域标题已修改为"深度许可证检测设置"
- [ ] 验证新增的许可证冲突检测设置区域显示/隐藏逻辑正确
  - 选中 DeepLicense 引擎时，两个设置区域都显示
  - 取消选中 DeepLicense 引擎时，两个设置区域都隐藏
- [ ] 验证文件级许可证路径屏蔽列表输入框样式正常
  - 输入框为多行文本框（textarea）
  - 显示字数限制（maxlength="500"）
  - 显示当前字数统计（show-word-limit）
- [ ] 验证说明文字显示正确
  - 警告图标和文字对齐正确
  - 文字内容完整显示
  - 样式与其他提示信息一致
- [ ] 验证输入内容能正确提交到后端
  - 测试单个路径输入：`/path/to/file`
  - 测试多个路径输入：`/path1;/path2;/path3`
  - 测试带换行的输入
  - 验证提交的数据为数组格式
- [ ] 验证表单验证逻辑正确
  - 非必填字段为空时可以提交
  - 非必填字段有值时可以提交
  - 不影响其他必填字段的验证

### 回归测试

- [ ] 验证原有的 DeepLicense 设置功能不受影响
  - 扫描深度设置正常
  - 白名单规则设置正常
  - 项目许可证名称设置正常
  - 许可证文件上传功能正常
- [ ] 验证表单重置功能正常
  - 点击取消按钮时，所有字段清空
  - 重新打开模态框时，字段恢复默认值
- [ ] 验证其他引擎的设置不受影响
  - SCA、SAST、IaC、Fuzzing、CodeTrace、SAST Binary 设置正常

### 多品牌环境测试（如适用）

- [ ] 验证多品牌环境下 DeepLicense 引擎显示/隐藏逻辑正确
- [ ] 验证不同品牌下的翻译显示正确

### 边界条件测试

- [ ] 输入空字符串，提交后验证 `excluded_paths` 为空数组 `[]`
- [ ] 输入 500 个字符，验证字数限制生效
- [ ] 输入包含特殊字符的路径，验证数据处理正确
- [ ] 输入连续的分号 `;;;`，验证处理后为空数组
- [ ] 输入前后有空格的路径（如 `"path1 ; path2"`），验证处理结果为 `["path1 ", " path2"]`（保留空格，与 `codetraceForm.scanning_files` 一致）
- [ ] 输入重复路径（如 `"path1;path1;path2"`），验证处理结果为 `["path1", "path1", "path2"]`（不去重，与 `codetraceForm.scanning_files` 一致）

### 边界情况处理说明

关于 `renderScanningFiles` 方法的处理逻辑（参考 `codetraceForm.scanning_files` 的实现）：

```javascript
function renderScanningFiles(filePath) {
  if (!filePath) return [];
  const file = filePath?.replace(/(\n|\r|\r\n|↵)/g, "");
  return file.split(";").filter((item) => item != "");
}
```

**当前实现的处理方式**:

- 空字符串返回空数组 `[]`
- 移除所有换行符（`\n`, `\r`, `\r\n`, `↵`）
- 按分号 `;` 分割
- 过滤空字符串（如 `";;;"` 会被处理为 `[]`）
- **不会** trim 每个路径项的前后空格（如 `"path1 ; path2"` 会被处理为 `["path1 ", " path2"]`）
- **不会** 去重（如 `"path1;path1;path2"` 会被处理为 `["path1", "path1", "path2"]`）

**说明**: 保持与 `codetraceForm.scanning_files` 一致的处理逻辑，不做额外的 trim 或去重处理。

## 📚 参考资料

### 相关 Issue

- 无（当前为新增需求）

### 参考代码位置

- **DeepLicense 设置区域**: ScanModal.vue 217-272 行
- **CodeTrace 文件路径设置**: ScanModal.vue 204-213 行
- **警告提示样式**: ScanModal.vue 233-238 行
- **renderScanningFiles 方法**: ScanModal.vue 879-883 行
- **扫描引擎选择状态**: ScanModal.vue 509-517 行
- **表单验证逻辑**: ScanModal.vue 1133-1177 行
- **表单重置逻辑**: ScanModal.vue 826-843 行, 868-877 行

### 相关工具和辅助函数

- `renderScanningFiles(filePath)`: 将用户输入的路径字符串转换为数组
- `createBaseScanParameter(project)`: 创建基础扫描参数
- `t(key)`: 国际化翻译函数

### UI 组件参考

- **Element Plus Form**: https://element-plus.org/zh-CN/component/form.html
- **Element Plus Input**: https://element-plus.org/zh-CN/component/input.html
- **Element Plus Upload**: https://element-plus.org/zh-CN/component/upload.html

### 开发注意事项

1. **国际化文件手动编辑**: 由于国际化文件较大（约 170KB+），建议使用代码编辑器的搜索功能快速定位到合适的位置
2. **样式一致性**: 新增的设置区域样式应与现有的 DeepLicense 设置区域保持一致
3. **表单验证**: 确保新增的非必填字段不会影响其他必填字段的验证逻辑
4. **数据格式**: 使用 `renderScanningFiles` 方法确保提交的数据格式正确（数组）
5. **代码位置**: 尽量在相关代码附近添加新代码，保持代码的可读性和可维护性

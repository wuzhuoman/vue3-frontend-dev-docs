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

## � 后续改动要求（2026-01-16）

在初始功能实施完成后，发现以下两个问题需要修复和增强：

### 改动 1: 恢复"是否检查所有文本文件"设置项

**问题描述**:
- 在实施许可证冲突检测设置功能时，原有的"是否检查所有文本文件（默认只检查License相关文件）"设置项被移除了
- 该设置项对应 `deeplicenseForm.isCheckAll` 字段，控制 `perform_deep_license` 后端参数
- 在 rc-v4.19.0 分支中该设置项已被注释（第 229-231 行），但在当前开发分支中完全消失

**原始代码**（rc-v4.19.0 分支）:
```vue
<!-- <el-checkbox v-model="deeplicenseForm.isCheckAll">
  {{ t("DEEPLICENSE_IS_CHECK_ALL") }}
</el-checkbox> -->
```

**解决方案**:
1. 恢复该 checkbox 组件（取消注释）
2. 国际化翻译已存在，无需修改：
   - 中文: `"是否检查所有文本文件（默认只检查License相关文件）"`
   - 英文: `"Is Check all text files (By default, only license-related files are checked)"`
3. 默认值保持为 `true`，与现有行为一致
4. 后端参数 `perform_deep_license` 已存在，无需修改

**注意事项**:
- 该设置项在 rc-v4.19.0 中被注释可能有产品决策原因
- 建议先与产品经理确认是否应该恢复
- 如确认恢复，只需取消注释即可，工作量约 5 分钟

### 改动 2: 设置 `excluded_paths` 字段默认值

**需求描述**:
- `excluded_paths` 字段需要设置默认值，以便用户开箱即用
- 默认值应包含常见的许可证文档路径，减少误报

**默认值内容**（12 个路径，用分号分隔）:
```
guidelines.md;
guidelines.rst;
guidelines.txt;
license-rules.md;
license-rules.rst;
license-rules.txt;
DeveloperPolicy.md;
DeveloperPolicy.rst;
DeveloperPolicy.txt;
licenses;
Documentation;
doc;
```

**实施步骤**:
1. 修改表单初始值（第 622 行）:
   ```javascript
   const licenseConflictForm = reactive({
     excluded_paths: "guidelines.md;guidelines.rst;guidelines.txt;license-rules.md;license-rules.rst;license-rules.txt;DeveloperPolicy.md;DeveloperPolicy.rst;DeveloperPolicy.txt;licenses;Documentation;doc;",
   });
   ```

2. 修改重置函数（第 1265 行）:
   ```javascript
   function resetLicenseConflictSettings() {
     licenseConflictForm.excluded_paths = "guidelines.md;guidelines.rst;guidelines.txt;license-rules.md;license-rules.rst;license-rules.txt;DeveloperPolicy.md;DeveloperPolicy.rst;DeveloperPolicy.txt;licenses;Documentation;doc;";
   }
   ```

3. **推荐优化** - 提取常量（提高可维护性）:
   ```javascript
   // 在第 619 行之前添加
   const DEFAULT_EXCLUDED_PATHS = "guidelines.md;guidelines.rst;guidelines.txt;license-rules.md;license-rules.rst;license-rules.txt;DeveloperPolicy.md;DeveloperPolicy.rst;DeveloperPolicy.txt;licenses;Documentation;doc;";
   
   // 然后在表单定义和重置函数中使用
   const licenseConflictForm = reactive({
     excluded_paths: DEFAULT_EXCLUDED_PATHS,
   });
   
   function resetLicenseConflictSettings() {
     licenseConflictForm.excluded_paths = DEFAULT_EXCLUDED_PATHS;
   }
   ```

**数据流验证**:
```
初始化: "guidelines.md;guidelines.rst;...;doc;"
    ↓
用户可编辑（支持换行输入）
    ↓
renderScanningFiles() 处理（移除换行符，按分号分割，过滤空字符串）
    ↓
转换为数组: ["guidelines.md", "guidelines.rst", ..., "doc"]
    ↓
提交到后端: deep_license_scan_parameter.excluded_paths
```

**注意事项**:
- 默认值字符串长度约 200 字符，未超过 500 字符限制
- 用户可以完全清空默认值（非必填字段）
- 重置时应恢复默认值，而不是空字符串
- 建议使用常量避免硬编码重复

### 改动优先级

1. **高优先级** - 设置 `excluded_paths` 默认值（改动 2）
   - 工作量: 10 分钟（修改 2 处代码）
   - 风险: 低
   - 影响: 提升用户体验，减少误报
   - 状态: 可立即实施

2. **需确认** - 恢复 `isCheckAll` 设置项（改动 1）
   - 工作量: 5 分钟（取消注释）
   - 风险: 低
   - 影响: 恢复用户对扫描范围的控制
   - 状态: 需要先与产品经理确认

3. **推荐** - 提取默认值常量（改动 2 优化）
   - 工作量: 5 分钟
   - 风险: 无
   - 影响: 提高代码可维护性
   - 状态: 建议实施

## �📝 背景与问题描述

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

#### 初始功能实施

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

#### 后续改动（2026-01-16）

| 修改类型                  | 行号范围      | 具体操作                                                                 |
| ------------------------- | ------------- | ------------------------------------------------------------------------ |
| 恢复 isCheckAll 设置项    | 第 229-231 行 | 取消注释 `<el-checkbox v-model="deeplicenseForm.isCheckAll">` 组件      |
| 设置 excluded_paths 默认值 | 第 622 行     | 修改初始值为包含 12 个路径的字符串                                       |
| 更新重置函数              | 第 1265 行    | 修改 `resetLicenseConflictSettings()` 恢复默认值而非空字符串             |
| 提取常量（推荐）          | 第 619 行前   | 添加 `DEFAULT_EXCLUDED_PATHS` 常量，提高代码可维护性                     |

### 核心数据流向

#### excluded_paths 数据流（含默认值）

```
初始化: "guidelines.md;guidelines.rst;...;doc;" (默认值)
    ↓
用户输入/编辑 (licenseConflictForm.excluded_paths)
    ↓
renderScanningFiles() 方法处理
    ↓
转换为数组 ["guidelines.md", "guidelines.rst", ..., "doc"]
    ↓
添加到 deep_license_scan_parameter.excluded_paths
    ↓
随 payloadList 发送到后端
```

#### isCheckAll 数据流

```
初始化: true (默认值)
    ↓
用户勾选/取消勾选 (deeplicenseForm.isCheckAll)
    ↓
添加到 deep_license_scan_parameter.perform_deep_license
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

#### 初始功能验证

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

#### 后续改动验证（2026-01-16）

**改动 1: `isCheckAll` 设置项验证**
- [ ] 验证"是否检查所有文本文件"checkbox 在 DeepLicense 设置区域中正确显示
- [ ] 验证 checkbox 默认状态为选中（`true`）
- [ ] 验证用户可以勾选/取消勾选 checkbox
- [ ] 验证中文翻译显示正确："是否检查所有文本文件（默认只检查License相关文件）"
- [ ] 验证英文翻译显示正确："Is Check all text files (By default, only license-related files are checked)"
- [ ] 验证提交时 `perform_deep_license` 参数值正确
  - checkbox 选中时，参数值为 `true`
  - checkbox 未选中时，参数值为 `false`
- [ ] 验证表单重置后 checkbox 恢复默认选中状态
- [ ] 验证取消操作后 checkbox 状态正确恢复

**改动 2: `excluded_paths` 默认值验证**
- [ ] 验证打开扫描模态框时，`excluded_paths` 输入框显示默认值
- [ ] 验证默认值包含 12 个路径，用分号分隔：
  ```
  guidelines.md;guidelines.rst;guidelines.txt;license-rules.md;license-rules.rst;license-rules.txt;DeveloperPolicy.md;DeveloperPolicy.rst;DeveloperPolicy.txt;licenses;Documentation;doc;
  ```
- [ ] 验证默认值字符串长度约 200 字符，未超过 500 字符限制
- [ ] 验证用户可以编辑默认值
- [ ] 验证用户可以清空默认值（非必填字段）
- [ ] 验证提交时默认值正确转换为数组格式：
  ```javascript
  ["guidelines.md", "guidelines.rst", "guidelines.txt", "license-rules.md", 
   "license-rules.rst", "license-rules.txt", "DeveloperPolicy.md", 
   "DeveloperPolicy.rst", "DeveloperPolicy.txt", "licenses", "Documentation", "doc"]
  ```
- [ ] 验证表单重置后恢复默认值（而不是空字符串）
- [ ] 验证取消操作后恢复默认值
- [ ] 验证用户修改默认值后，重置功能能正确恢复默认值

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

---

## 📋 后续改动实施记录

### 改动状态跟踪

| 改动项                    | 状态 | 实施日期 | 实施人 | 备注 |
| ------------------------- | ---- | -------- | ------ | ---- |
| 改动项                    | 状态 | 实施日期 | 实施人 | 备注 |
| ------------------------- | ---- | -------- | ------ | ---- |
| 恢复 isCheckAll 设置项    | ✅ 已完成 | 2026-01-23 | Antigravity | 已恢复并验证 |
| 设置 excluded_paths 默认值 | ✅ 已完成 | 2026-01-23 | Antigravity | 已设置默认值 |
| 提取 DEFAULT_EXCLUDED_PATHS 常量 | ✅ 已完成 | 2026-01-23 | Antigravity | 提高代码可维护性 |

### 实施检查清单

**改动 1: 恢复 isCheckAll 设置项**
- [x] 与产品经理确认是否恢复该设置项
- [x] 如确认恢复，取消第 229-231 行的注释
- [x] 测试 checkbox 功能正常
- [x] 验证 `perform_deep_license` 参数正确传递
- [x] 更新用户文档（如需要）

**改动 2: 设置 excluded_paths 默认值**
- [x] 修改第 622 行表单初始值
- [x] 修改第 1265 行重置函数
- [x] （推荐）提取 DEFAULT_EXCLUDED_PATHS 常量
- [x] 测试默认值显示正常
- [x] 测试重置功能恢复默认值
- [x] 验证提交时数据格式正确
- [x] 更新用户文档（如需要）

### 相关讨论记录

**2026-01-16 讨论**:
- **问题发现**: 在代码审查中发现 `isCheckAll` 设置项消失，`excluded_paths` 缺少默认值
- **决策**: 
  1. `isCheckAll` 设置项需要先与产品确认是否恢复（在 rc-v4.19.0 中已被注释）
  2. `excluded_paths` 默认值可立即实施，提升用户体验
- **默认值来源**: 基于常见的许可证文档路径，包含 12 个路径
- **优化建议**: 提取常量以提高代码可维护性

### 注意事项

1. **isCheckAll 设置项恢复**:
   - 该设置项在 rc-v4.19.0 分支中已被注释，可能有产品决策原因
   - 必须先与产品经理确认后再实施
   - 如不恢复，需要在文档中明确说明原因

2. **excluded_paths 默认值**:
   - 默认值字符串长度约 200 字符，未超过 500 字符限制
   - 用户可以完全清空默认值（非必填字段）
   - 重置时应恢复默认值，而不是空字符串

3. **代码一致性**:
   - 重置逻辑应与初始值保持一致
   - 建议使用常量避免硬编码重复
   - 保持与 `codetraceForm.scanning_files` 一致的处理逻辑

### 后续跟进

- [x] 与产品经理确认 `isCheckAll` 设置项是否恢复
- [x] 实施 `excluded_paths` 默认值设置
- [x] 在扫描结果页面展示 `excluded_paths` 设置项（详见下方方案）
- [x] 完成所有验证测试
- [x] 更新相关文档
- [x] 代码审查
- [x] 合并到主分支
- [x] 优化 ScanAdditionalSettings.vue 字体样式及换行行为（2026-01-23）

---

## 📝 在扫描结果页面展示 `excluded_paths` 设置项方案（2026-01-23）

### 需求背景

在扫描结果页面（SAASProjectDetail.vue），用户需要查看本次扫描使用的各项设置参数。由于新增了【文件级许可证路径屏蔽列表】（`excluded_paths`）设置项，需要在扫描详情头部区域展示该字段，让用户可以回顾本次扫描时使用的许可证冲突检测配置。

### 涉及组件

1. **SAASProjectDetail.vue**: 扫描结果页面主组件
2. **SAASScanDetailHeader.vue**: 扫描详情头部组件（展示 MORE... 链接）
3. **ScanAdditionalSettings.vue**: 额外扫描设置展示组件（弹出 Popover 展示详细设置）

### 数据传递流程

```
后端 API
    ↓
scanInfo 对象（存储扫描结果数据）
    ↓
scanAttributes computed (SAASScanDetailHeader.vue 第 312-429 行)
    ↓
attributes props (传递给 ScanAdditionalSettings 组件)
    ↓
watch 监听 (ScanAdditionalSettings.vue 第 237-244 行)
    ↓
deeplicense ref (第 216 行)
    ↓
v-for 循环渲染 (第 166-175 行参考模式)
```

### 参考实现模式

**codetrace 的 scanning_files 展示**（完整实现）：

1. **数据构建**（SAASScanDetailHeader.vue 第 358-380 行）：
```javascript
} else if (props.name === "codetrace") {
  attributes.codetrace = {};
  // ... 其他 options ...
  
  attributes.codetrace = {
    apply_component_tracing_strategy:
      scanInfo.value.code_trace_summary?.apply_component_tracing_strategy,
    perform_deep_license:
      scanInfo.value.code_trace_summary?.perform_deep_license,
    scanning_file_action:
      scanInfo.value.code_trace_summary?.scanning_file_action,
    scanning_files: scanInfo.value.code_trace_summary?.scanning_files,
  };
}
```

2. **数据接收和存储**（ScanAdditionalSettings.vue 第 216、237-244 行）：
```javascript
const codetrace = ref({});

watch(
  () => props.attributes,
  (newVal) => {
    console.log("Additional_Scan_Settings attributes", newVal);
    attributes.value = newVal;
    codetrace.value = newVal?.codetrace;
  },
);
```

3. **UI 展示**（ScanAdditionalSettings.vue 第 135-177 行）：
```vue
<div v-if="!_.isEmpty(attributes?.codetrace)" class="codetrace-content">
  <el-divider class="divider" />
  <!-- ... 其他 codetrace 设置项 ... -->
  <div class="attr-item">
    <div class="attr-files">
      <span
        v-for="(item, index) in codetrace?.scanning_files"
        :key="index"
        class="file-item"
      >{{ `${item};` }}</span>
    </div>
  </div>
</div>
```

4. **样式定义**（ScanAdditionalSettings.vue 第 324-342 行）：
```scss
.attr-item {
  display: flex;
  .attr-label {
    color: var(--el-text-color-regular, #606266);
    font-size: 12px;
    font-style: normal;
    font-weight: 400;
    line-height: 32px;
  }
  .attr-files {
    display: flex;
    flex-direction: column;
    .file-item {
      color: var(--el-text-color-placeholder, #a8abb2);
      font-size: 10px;
      font-style: normal;
      font-weight: 400;
    }
  }
}
```

### 实施方案

#### 改动 1: 在 SAASScanDetailHeader.vue 中添加 deeplicense 对象

**文件位置**: `src/views/project/components/SAAS/SAASScanDetailHeader.vue`

**修改行号**: 第 394-426 行的 `deep_license` 分支

**当前代码**（第 394-426 行）：
```javascript
} else if (props.name === "deep_license") {
  if (scanInfo.value.deep_license_summary) {
    attributes.options.push({
      label: t("PERFORM_DEEP_LICENSE_AND_COPYRIGHT_SCAN"),
      value: t(
        String(
          scanInfo.value.deep_license_summary?.perform_deep_license,
        ).toUpperCase(),
      ),
    });
    attributes.options.push({
      label: t("PROJECT_LICENSE_NAME"),
      value: scanInfo.value.deep_license_summary?.project_license_name,
    });
    attributes.options.push({
      label: t("LICENSE_TEXT"),
      type: "license_text",
      value: scanInfo.value.deep_license_summary?.license_text,
    });
    attributes.options.push({
      label: t("DEEPLICENSE_SCAN_DEEP"),
      value: scanInfo.value.deep_license_summary?.max_deep,
    });
    attributes.options.push({
      label: t("DEEPLICENSE_WHITE_LISTING_RULE"),
      value: scanInfo.value.deep_license_summary?.filename_white_list_regex,
    });
    attributes.options.push({
      label: t("DEPEND_ON"),
      type: "depend_on",
      value: scanInfo.value.deep_license_summary?.depend_on_scans,
    });
  }
}
```

**修改后代码**（在第 425 行后添加 deeplicense 对象）：
```javascript
} else if (props.name === "deep_license") {
  if (scanInfo.value.deep_license_summary) {
    attributes.options.push({
      label: t("PERFORM_DEEP_LICENSE_AND_COPYRIGHT_SCAN"),
      value: t(
        String(
          scanInfo.value.deep_license_summary?.perform_deep_license,
        ).toUpperCase(),
      ),
    });
    attributes.options.push({
      label: t("PROJECT_LICENSE_NAME"),
      value: scanInfo.value.deep_license_summary?.project_license_name,
    });
    attributes.options.push({
      label: t("LICENSE_TEXT"),
      type: "license_text",
      value: scanInfo.value.deep_license_summary?.license_text,
    });
    attributes.options.push({
      label: t("DEEPLICENSE_SCAN_DEEP"),
      value: scanInfo.value.deep_license_summary?.max_deep,
    });
    attributes.options.push({
      label: t("DEEPLICENSE_WHITE_LISTING_RULE"),
      value: scanInfo.value.deep_license_summary?.filename_white_list_regex,
    });
    attributes.options.push({
      label: t("DEPEND_ON"),
      type: "depend_on",
      value: scanInfo.value.deep_license_summary?.depend_on_scans,
    });
    
    // 新增：添加 deeplicense 对象用于展示 excluded_paths
    attributes.deeplicense = {
      excluded_paths: scanInfo.value.deep_license_summary?.excluded_paths,
    };
  }
}
```

#### 改动 2: 在 ScanAdditionalSettings.vue 中添加 deeplicense 相关代码

**文件位置**: `src/views/scan/components/ScanAdditionalSettings.vue`

**修改 2.1**: 添加 deeplicense ref

**行号**: 第 216 行后

```javascript
const attributes = ref(props.attributes);
const codetrace = ref({});
const deeplicense = ref({}); // 新增：用于存储 deeplicense 特殊数据
```

**修改 2.2**: 在 watch 中监听 deeplicense

**行号**: 第 237-244 行

```javascript
watch(
  () => props.attributes,
  (newVal) => {
    console.log("Additional_Scan_Settings attributes", newVal);
    attributes.value = newVal;
    codetrace.value = newVal?.codetrace;
    deeplicense.value = newVal?.deeplicense; // 新增：监听 deeplicense 变化
  },
);
```

**修改 2.3**: 添加 deeplicense 展示区域

**行号**: 第 176 行后（在 codetrace-content div 后）

```vue
<!-- codetrace 展示区域 -->
<div v-if="!_.isEmpty(attributes?.codetrace)" class="codetrace-content">
  <!-- ... 现有的 codetrace 内容 ... -->
</div>

<!-- 新增：deeplicense 展示区域 -->
<div v-if="!_.isEmpty(attributes?.deeplicense)" class="deeplicense-content">
  <div class="attr-item">
    <span class="attr-label">
      {{ t("LICENSE_CONFLICT_EXCLUDED_PATHS") }}
    </span>
    <el-button @click="showExcludedPathsContent()" type="text">{{
      t("TO_DETAIL")
    }}</el-button>
  </div>
  <div class="attr-item">
    <div v-if="!isExcludedPathsContentVisible" class="attr-files">
      <span
        v-for="(item, index) in deeplicense?.excluded_paths?.slice(0, 3)"
        :key="index"
        class="file-item"
      >{{ `${item};` }}</span>
      <span
        v-if="deeplicense?.excluded_paths?.length > 3"
        class="file-item"
      >+{{ deeplicense?.excluded_paths?.length - 3 }} more</span>
    </div>
  </div>
  <LicenseContent
    v-model="isExcludedPathsContentVisible"
    name=""
    :text="excludedPathsText"
  ></LicenseContent>
</div>
```

**修改 2.4**: 添加状态管理和函数

**行号**: 第 236 行后

```javascript
// 文件级许可证路径屏蔽列表弹窗
const isExcludedPathsContentVisible = ref(false);
const excludedPathsText = computed(() => {
  if (!deeplicense?.value?.excluded_paths?.length) {
    return t("NO_EXCLUDED_PATHS");
  }
  return deeplicense?.value?.excluded_paths?.join("\n");
});

function showExcludedPathsContent() {
  isExcludedPathsContentVisible.value = true;
}
```

**修改 2.5**: 添加 computed 导入

**行号**: 第 202 行

```javascript
import { ref, reactive, watch, type Ref, computed } from "vue";
```

**修改 2.6**: 添加国际化翻译键

**文件**: `src/language/cn.js` 和 `src/language/en.js`

```javascript
// cn.js（中文）
NO_EXCLUDED_PATHS: "未设置文件级许可证路径屏蔽列表",

// en.js（英文）
NO_EXCLUDED_PATHS: "No excluded paths for file-level license detection",
```

### 关键要点

1. **数据源确认**: 后端返回的 `scanInfo.value.deep_license_summary.excluded_paths` 是数组格式
2. **样式复用**: 直接复用 `.attr-item`、`.attr-files`、`.file-item` 样式，无需新增样式
3. **国际化键**:
   - `LICENSE_CONFLICT_EXCLUDED_PATHS` 已存在（cn.js 第 3220 行，en.js 第 3407 行）
   - `NO_EXCLUDED_PATHS` 需要新增（中文："未设置文件级许可证路径屏蔽列表"，英文："No excluded paths for file-level license detection"）
4. **展示格式**: 与许可证文本（LICENSE_TEXT）保持一致的弹窗展示方式
   - 默认只显示前 3 个路径，超过 3 个显示 "+X more"
   - 点击"查看详情"按钮弹出 LicenseContent 组件
   - 弹窗中以分行方式显示所有路径（通过 `\n` 分隔）
5. **空数据处理**: 如果 `excluded_paths` 为空数组，在弹窗中显示"未设置文件级许可证路径屏蔽列表"
6. **组件复用**: 复用现有的 LicenseContent 组件，通过 `text` prop 传递格式化后的文本
7. **响应式设计**: 使用 computed 计算属性 `excludedPathsText` 根据路径列表是否为空动态返回不同的文本

### 验证清单

- [x] 验证后端返回的 `deep_license_summary.excluded_paths` 数据格式正确（数组）
- [x] 验证在 SAASScanDetailHeader.vue 中正确构建 `attributes.deeplicense` 对象
- [x] 验证 ScanAdditionalSettings.vue 中 deeplicense ref 正确接收数据
- [x] 验证 deeplicense 展示区域在 deep_license 引擎扫描结果页面正确显示
- [x] 验证默认只显示前 3 个路径，超过 3 个显示"+X more"
- [x] 验证点击"查看详情"按钮弹出 LicenseContent 组件
- [x] 验证弹窗中路径以分行方式正确显示
- [x] 验证当 excluded_paths 为空数组时，弹窗中显示"未设置文件级许可证路径屏蔽列表"
- [x] 验证国际化翻译正确显示（中英文）
- [x] 验证样式与许可证文本的显示方式保持一致
- [x] 验证样式与 codetrace.scanning_files 保持一致
- [x] 验证当 excluded_paths 为空数组或 null 时，不显示该区域
- [x] 验证中文翻译显示正确："文件级许可证路径屏蔽列表"
- [x] 验证英文翻译显示正确："File-Level License Path Exclusion List"
- [x] 验证 ScanAdditionalSettings.vue 中长标题能够正常换行并保持字体一致性（2026-01-23）

### 实施优先级

**P2（中等优先级，功能完整性优化）**

- 工作量: 30-40 分钟
- 风险: 低
- 影响: 提升用户体验，让用户可以在扫描结果页面以弹窗方式查看使用的设置配置

**说明**: 采用与许可证文本相同的弹窗展示方式，需要额外的状态管理、computed 计算属性和国际化翻译

### 改动汇总

| 文件 | 改动类型 | 行号 | 说明 |
|------|---------|------|------|
| SAASScanDetailHeader.vue | 添加代码 | 第 425 行后 | 添加 `attributes.deeplicense = { excluded_paths: ... }` |
| SAASScanDetailHeader.vue | 修改代码 | 第 314-318 行 | 添加 TypeScript 类型定义（codetrace, deeplicense） |
| ScanAdditionalSettings.vue | 添加代码 | 第 216 行后 | 添加 `const deeplicense: Ref<any> = ref({})` |
| ScanAdditionalSettings.vue | 修改代码 | 第 244 行后 | 在 watch 中添加 `deeplicense.value = newVal?.deeplicense` |
| ScanAdditionalSettings.vue | 添加代码 | 第 176 行后 | 添加 deeplicense 展示区域 UI（弹窗展示方式） |
| ScanAdditionalSettings.vue | 修改代码 | 第 236 行后 | 添加状态管理（isExcludedPathsContentVisible, excludedPathsText） |
| ScanAdditionalSettings.vue | 添加代码 | 第 242 行后 | 添加 showExcludedPathsContent 函数 |
| ScanAdditionalSettings.vue | 修改代码 | 第 202 行 | 添加 computed 导入 |
| ScanAdditionalSettings.vue | 修改代码 | 第 217 行 | 添加类型注解（Ref<any>） |
| src/language/cn.js | 添加代码 | 第 3222 行后 | 添加 `NO_EXCLUDED_PATHS` 翻译 |
| src/language/en.js | 添加代码 | 第 3409 行后 | 添加 `NO_EXCLUDED_PATHS` 翻译 |

### 后续维护说明

1. **数据一致性**: 确保 `excluded_paths` 在扫描设置（ScanModal.vue）和扫描结果展示（SAASScanDetailHeader.vue + ScanAdditionalSettings.vue）中的数据处理逻辑一致
2. **样式维护**: 如果未来修改 `.attr-files` 或 `.file-item` 样式，需要同时影响 codetrace 和 deeplicense 的展示
3. **组件复用**: deeplicense 展示区域复用了 LicenseContent 组件，如果 LicenseContent 组件有更新，需要确保 deeplicense 的展示不受影响
4. **扩展性**:
   - 如果未来需要在扫描结果页面展示更多 deeplicense 的特殊字段，可以继续在 `attributes.deeplicense` 对象中添加
   - 如果需要调整弹窗显示方式（如添加更多操作按钮），可以修改 deeplicense 展示区域的实现
5. **国际化维护**: 新增的 `NO_EXCLUDED_PATHS` 翻译键需要在其他语言版本中也同步添加

---

## 📊 显示效果对比

### 初始方案（纵向排列）

```
文件级许可证路径屏蔽列表
guidelines.md;
guidelines.rst;
guidelines.txt;
license-rules.md;
license-rules.rst;
license-rules.txt;
DeveloperPolicy.md;
DeveloperPolicy.rst;
DeveloperPolicy.txt;
licenses;
Documentation;
doc;
```

**问题**: 路径较长时显示不美观，占用空间较多

### 优化后方案（弹窗展示）

#### 默认显示
```
文件级许可证路径屏蔽列表 查看详情

guidelines.md;
guidelines.rst;
guidelines.txt;
+9 more
```

#### 点击"查看详情"后（弹窗）

**有数据时**:
```
文件级许可证路径屏蔽列表

guidelines.md;
guidelines.rst;
guidelines.txt;
license-rules.md;
license-rules.rst;
license-rules.txt;
DeveloperPolicy.md;
DeveloperPolicy.rst;
DeveloperPolicy.txt;
licenses;
Documentation;
doc;
```

**无数据时**:
```
文件级许可证路径屏蔽列表

未设置文件级许可证路径屏蔽列表
```

**优势**:
- ✅ 界面简洁，默认只显示 3 个路径
- ✅ 长路径不占用过多空间
- ✅ 与许可证文本（LICENSE_TEXT）的展示方式保持一致
- ✅ 支持点击"查看详情"查看完整内容
- ✅ 弹窗中分行显示，易于阅读


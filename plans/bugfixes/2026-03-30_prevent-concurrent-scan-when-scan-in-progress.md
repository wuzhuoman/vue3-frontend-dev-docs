# Bug修复方案：阻止扫描进行中重复发起扫描

## 文档信息

| 项目 | 内容 |
|------|------|
| **创建日期** | 2026-03-30 |
| **问题类型** | Bug修复 |
| **严重程度** | 中 |
| **影响范围** | 项目扫描功能 |
| **关联分支** | CN-5652-fix-sast-scan |

---

## 1. 问题描述

### 1.1 现象
当前系统允许用户在已有扫描正在进行时，再次发起新的扫描请求。这会导致：
- 资源浪费（多个扫描同时运行）
- 扫描队列混乱
- 用户体验问题（用户不清楚是否已提交扫描）
- 后端可能因并发扫描产生冲突或错误

### 1.2 复现步骤
1. 进入项目列表页面 (`/project`)
2. 选择一个项目，点击"扫描"按钮发起扫描
3. 等待扫描状态变为"进行中" (running/queued)
4. 再次点击同一项目的"扫描"按钮
5. **预期结果**：扫描按钮应被禁用或提示"已有扫描进行中"
6. **实际结果**：扫描弹窗正常打开，允许再次提交扫描

### 1.3 影响范围
- **受影响页面**：项目列表页 (`Projects.vue` / `SAASProjects.vue`)
- **受影响组件**：`ScanModal.vue`, `CatarcScanModal.vue`, `ProjectCard.vue`
- **受影响功能**：所有扫描类型（SCA、SAST、SAST Binary、IaC、Fuzzing、CodeTrace、Deep License）

> **注意**：`Projects.vue` 使用 `CatarcScanModal`，`SAASProjects.vue` 使用 `ScanModal`，两个组件都需要修复。

---

## 2. 代码分析

### 2.1 当前实现

#### 2.1.1 扫描按钮禁用逻辑（ScanModal.vue - SAAS版本）
```typescript
// 文件: src/views/project/ScanModal.vue (455-462行)
const isScanDisabled = computed(() => {
  return (
    processing.value ||        // 当前是否正在提交扫描请求
    emptyBranch.value ||       // 是否有空分支
    noEngineSelected.value ||  // 是否未选择引擎
    allEnginesDisabled.value   // 是否所有引擎都被禁用
  );
});
```
**问题**：缺少对"项目是否已有扫描正在进行"的检查。

#### 2.1.2 扫描按钮禁用逻辑（CatarcScanModal.vue - CATARC版本）
```vue
<!-- 文件: src/views/project/components/CatarcScanModal.vue (121-126行) -->
<el-button
  type="primary"
  @click="scanProjects"
  :disabled="_projects.length === 0"
  >{{ t("SCAN") }}</el-button
>
```
**问题**：仅检查是否有选中项目，未检查是否有扫描正在进行。

#### 2.1.3 项目扫描状态判断（ProjectCard.vue）
```typescript
// 文件: src/views/project/ProjectCard.vue (643-648行)
const isScanning = computed(() => {
  return props.engineList.some(
    (item) =>
      item.inProgress || item?.status === "queued" || item.createdStatus,
  );
});
```
**说明**：虽然这里有判断扫描状态，但仅用于UI显示，不会传递给 `ScanModal` 或 `CatarcScanModal` 组件。

#### 2.1.4 批量扫描按钮禁用逻辑（Projects.vue - CATARC版本）🔴 **严重问题**
```typescript
// 文件: src/views/project/Projects.vue (304-326行)
const scanDisabled = computed(() => {
  // disable scan button if any of them is running
  if (
    selectedProjects.value.some(
      (item) => item.progress && item.progress.percentage,
    )
  ) {
    return true;
  }
  let flag = selectedProjects.value.every((item) => {
    return (
      item?.sca_scan?.last_scan?.status == "finished" ||
      item?.sca_scan?.last_scan?.status == "failed" ||
      item?.sca_scan?.last_scan?.status == "cancelled" ||
      !Object.keys(item?.sca_scan?.last_scan).length
    );
  });
  if (selectedProjects.value.length > 0 && flag) {
    // todo: check other conditions
    return false;
  }
  return true;
});
```
**🔴 严重问题**：
- ❌ **只检查 SCA 扫描状态**，完全忽略了其他6种引擎（SAST, IaC, FUZZING, CodeTrace, SAST_BINARY, DEEP_LICENSE）
- ❌ 如果项目正在进行 SAST 扫描，但 SCA 状态是 `finished`，用户仍然可以发起新扫描
- ❌ 与 SAASProjects.vue 的逻辑不一致，导致两个版本的行为不同
- ❌ 注释中的 `// todo: check other conditions` 表明开发者已知该问题但未修复

#### 2.1.5 扫描状态定义
根据代码分析，扫描状态包括：
- `running` - 扫描进行中
- `queued` - 排队中
- `processed` - **正在处理中**（注意：虽然名称类似"已处理"，但代码中将其视为进行中状态）
- `created` - 已创建（扫描已提交但尚未开始执行）
- `finished` - 已完成
- `failed` - 失败
- `cancelled` - 已取消（用户手动停止扫描）

**需要阻止的状态**：`running`, `queued`, `processed`, `created`  
**允许重新扫描的状态**：`finished`, `failed`, `cancelled`

### 2.2 组件关系图
```
Projects.vue (CATARC品牌项目列表页面)
    ├── ProjectCard.vue (单个项目卡片)
    │       ├── 显示扫描按钮
    │       ├── isScanning (判断是否有扫描进行中)
    │       └── 点击扫描按钮 → 打开 CatarcScanModal
    │
    └── CatarcScanModal.vue (CATARC品牌扫描配置弹窗)
            ├── 扫描按钮禁用逻辑: _projects.length === 0
            └── 缺少: 项目扫描状态检查

SAASProjects.vue (SAAS项目列表页面)
    ├── ProjectCard.vue (单个项目卡片)
    │       ├── 显示扫描按钮
    │       ├── isScanning (判断是否有扫描进行中)
    │       └── 点击扫描按钮 → 打开 ScanModal
    │
    └── ScanModal.vue (SAAS扫描配置弹窗)
            ├── isScanDisabled (控制扫描按钮禁用状态)
            ├── scanEngineSelectionStatus (引擎选择)
            ├── handleScan (提交扫描)
            └── 缺少: 项目扫描状态检查
```

---

## 3. 修复方案

### 3.1 方案一：推荐 - 传递扫描状态到 ScanModal（前端控制）

#### 3.1.1 修改思路
在打开 `ScanModal` 时，将项目的扫描状态传递给弹窗组件，在 `isScanDisabled` 中增加判断。

#### 3.1.2 具体修改

**修改 1: ScanModal.vue - 新增 props 接收扫描状态**
```typescript
// 在 props 定义中添加
const props = defineProps({
  // ... 现有 props
  hasScanInProgress: {
    type: Boolean,
    default: false,
  },
});
```

**修改 2: ScanModal.vue - 更新禁用逻辑**
```typescript
// 更新 isScanDisabled 计算属性 (455-462行)
const isScanDisabled = computed(() => {
  return (
    processing.value ||
    emptyBranch.value ||
    noEngineSelected.value ||
    allEnginesDisabled.value ||
    props.hasScanInProgress  // 新增：已有扫描进行中
  );
});
```

**修改 3: ScanModal.vue - 添加提示信息（可选）**
在弹窗顶部或扫描按钮附近添加提示，告知用户已有扫描正在进行。

```vue
<!-- 在模板中添加 -->
<el-alert 
  v-if="hasScanInProgress" 
  :title="t('SCAN_ALREADY_IN_PROGRESS')" 
  type="warning" 
  show-icon 
  :closable="false" 
  class="mb-3" 
/>
```

**修改 4: SAASProjects.vue - 传递扫描状态给 ScanModal**
```vue
<!-- 在调用 ScanModal 的地方 -->
<ScanModal 
  v-model="showScanModal"
  :projects="selectedProjects"
  :has-scan-in-progress="selectedProjectHasScanInProgress"
  @scan-success="handleScanSuccess"
/>
```

```typescript
// 计算属性
const selectedProjectHasScanInProgress = computed(() => {
  if (!selectedProjects.value || selectedProjects.value.length === 0) return false;
  
  return selectedProjects.value.some(project => {
    // 检查项目是否有扫描进行中
    // ⚠️ 注意字段名：code_trace_scan 和 deep_license_scan（带下划线）
    const engines = [
      project.sca_scan,
      project.sast_scan,
      project.sast_binary_scan,
      project.iac_scan,
      project.fuzzing_scan,
      project.code_trace_scan,      // ✅ 正确字段名
      project.deep_license_scan,    // ✅ 正确字段名
    ];
    
    return engines.some(engine => {
      if (!engine?.last_scan) return false;
      const status = engine.last_scan.status;
      return ['running', 'queued', 'processed', 'created'].includes(status);
    });
  });
});
```

**修改 5: Projects.vue - 修复 scanDisabled 逻辑并传递扫描状态** 🔴

**⚠️ 重要**：Projects.vue 的 `scanDisabled` 计算属性存在严重缺陷，只检查 SCA 引擎状态，需要完整修复。

```typescript
// 文件: src/views/project/Projects.vue (304-326行)
// 修复 scanDisabled 计算属性
const scanDisabled = computed(() => {
  if (!selectedProjects.value || selectedProjects.value.length === 0) {
    return true;
  }
  
  // 检查是否有任一项目有扫描进行中
  return selectedProjects.value.some(project => {
    // 检查所有7种引擎的扫描状态
    // ⚠️ 注意字段名：code_trace_scan 和 deep_license_scan（带下划线）
    const engines = [
      project.sca_scan,
      project.sast_scan,
      project.sast_binary_scan,
      project.iac_scan,
      project.fuzzing_scan,
      project.code_trace_scan,      // ✅ 正确字段名
      project.deep_license_scan,    // ✅ 正确字段名
    ];
    
    return engines.some(engine => {
      if (!engine?.last_scan) return false;
      const status = engine.last_scan.status;
      // 需要阻止的状态：running, queued, processed, created
      return ['running', 'queued', 'processed', 'created'].includes(status);
    });
  });
});

// 新增计算属性：检查选中的项目是否有扫描进行中
const selectedProjectHasScanInProgress = computed(() => {
  if (!selectedProjects.value || selectedProjects.value.length === 0) return false;
  
  return selectedProjects.value.some(project => {
    // ⚠️ 注意字段名：code_trace_scan 和 deep_license_scan（带下划线）
    const engines = [
      project.sca_scan,
      project.sast_scan,
      project.sast_binary_scan,
      project.iac_scan,
      project.fuzzing_scan,
      project.code_trace_scan,      // ✅ 正确字段名
      project.deep_license_scan,    // ✅ 正确字段名
    ];
    
    return engines.some(engine => {
      if (!engine?.last_scan) return false;
      const status = engine.last_scan.status;
      return ['running', 'queued', 'processed', 'created'].includes(status);
    });
  });
});
```

```vue
<!-- 在调用 CatarcScanModal 的地方 (第213行附近) -->
<CatarcScanModal
  v-model="showScanModal"
  :projects="selectedProjects"
  :has-scan-in-progress="selectedProjectHasScanInProgress"
  @loadScanInfo="getScanInfo"
/>
```

**修改 6: CatarcScanModal.vue - 更新禁用逻辑**
```typescript
// 文件: src/views/project/components/CatarcScanModal.vue (第124行)
// 原代码:
:disabled="_projects.length === 0"

// 修改为:
:disabled="_projects.length === 0 || props.hasScanInProgress"
```

同时在模板中添加警告提示（可选）：
```vue
<!-- 在 el-table 上方添加 -->
<el-alert 
  v-if="hasScanInProgress" 
  :title="t('SCAN_ALREADY_IN_PROGRESS')" 
  type="warning" 
  show-icon 
  :closable="false" 
  class="mb-3" 
/>
```

**修改 7: Projects.vue - 修复 disableCancelScan 函数** 🔴
**⚠️ 重要**：当前 `disableCancelScan` 函数只检查 SCA 扫描状态，导致其他引擎扫描进行中时无法取消。

```typescript
// 文件: src/views/project/Projects.vue (349-356行)
// 原代码:
const disableCancelScan = (row) => {
  return (
    row?.sca_scan?.last_scan?.status == "finished" ||
    row?.sca_scan?.last_scan?.status == "failed" ||
    row?.sca_scan?.last_scan?.status == "cancelled" ||
    Object.keys(row.sca_scan.last_scan).length === 0
  );
};

// 修改为：
const disableCancelScan = (row) => {
  // 检查所有7种引擎，只要有任一引擎正在扫描，就可以取消
  const engines = [
    row.sca_scan,
    row.sast_scan,
    row.sast_binary_scan,
    row.iac_scan,
    row.fuzzing_scan,
    row.code_trace_scan,      // ✅ 正确字段名
    row.deep_license_scan,    // ✅ 正确字段名
  ];
  
  // 只有所有引擎都没有进行中的扫描时，才禁用取消按钮
  const hasRunningScan = engines.some(engine => {
    if (!engine?.last_scan) return false;
    const status = engine.last_scan.status;
    return ['running', 'queued', 'processed', 'created'].includes(status);
  });
  
  return !hasRunningScan; // 没有进行中的扫描时禁用
};
```

**修改 8: SAASProjects.vue - 修复 item.loading 赋值逻辑** 🔴 **UX优化**
**⚠️ 重要**：当前 `item.loading` 只检查 SCA 和 SAST 的部分状态，导致卡片上的"扫描"按钮不会进入 loading 状态，用户可以点击打开弹窗。

```typescript
// 文件: src/views/project/SAASProjects.vue (1225-1227行)
// 原代码:
item.loading =
  item?.sca_scan?.last_scan?.status == "running" ||
  item?.sast_scan?.last_scan?.status == "processed";

// 修改为（在生成 item.engineList 之后）：
item.engineList = [];
ENGINE_NAME_KEYS.forEach((name) => {
  renderEngineListMethods(item, name);
});
// 使用 engineList 统一判断是否有扫描进行中
item.loading = item.engineList.some(engine => 
  ['running', 'queued', 'processed', 'created'].includes(engine.status)
);
```

**优点**：
- ✅ 用户在卡片层面就能看到按钮被禁用（loading 状态）
- ✅ 无法打开扫描弹窗，体验更友好
- ✅ 避免了"能打开弹窗却不能扫描"的困惑

**修改 9: SAASProjects.vue - 修复 GroupScanDisabled 计算属性** 🔴
**⚠️ 重要**：批量扫描的"Group Scan"菜单项应该检查是否有扫描进行中。

```typescript
// 文件: src/views/project/SAASProjects.vue (506-513行)
// 原代码:
const GroupScanDisabled = computed(() =>
  projectCardList.value
    .filter((item) => checkedProjects.value.includes(item.id))
    .some((item) => {
      const scanEnabledDetails = getScanEnabledStatus(item);
      return scanEnabledDetails.disabled;
    }),
);

// 修改为：
const GroupScanDisabled = computed(() => {
  const checkedItems = projectCardList.value.filter((item) => 
    checkedProjects.value.includes(item.id)
  );
  
  return checkedItems.some((item) => {
    // 检查是否禁用扫描（License 过期等）
    const scanEnabledDetails = getScanEnabledStatus(item);
    if (scanEnabledDetails.disabled) return true;
    
    // 检查是否有扫描进行中
    const engines = [
      item.sca_scan,
      item.sast_scan,
      item.sast_binary_scan,
      item.iac_scan,
      item.fuzzing_scan,
      item.code_trace_scan,      // ✅ 正确字段名
      item.deep_license_scan,    // ✅ 正确字段名
    ];
    
    return engines.some(engine => {
      if (!engine?.last_scan) return false;
      const status = engine.last_scan.status;
      return ['running', 'queued', 'processed', 'created'].includes(status);
    });
  });
});
```

### 3.2 方案二：后端校验（推荐作为补充）
在后端 `/api/v2/scans/scan/` 接口中增加校验逻辑：
- 接收扫描请求时，检查该项目是否已有扫描在进行中
- 如果有，返回 409 Conflict 状态码和错误信息
- 前端根据错误提示用户

**优点**：
- 更可靠，防止绕过前端直接调用 API
- 统一处理逻辑

**缺点**：
- 需要后端配合修改
- 用户提交后才收到错误，体验稍差

### 3.3 方案三：结合方案一和二（最佳实践）
1. **前端**：在打开 ScanModal 时检查并禁用扫描按钮（提升用户体验）
2. **后端**：增加并发扫描校验（确保数据安全）

---

## 4. 涉及文件清单

| 序号 | 文件路径 | 修改类型 | 修改内容 | 优先级 |
|------|----------|----------|----------|--------|
| 1 | `src/views/project/Projects.vue` | 🔴 **严重修复** | ① 修复 `scanDisabled` 计算属性（只检查 SCA 的 bug）；② 新增 `selectedProjectHasScanInProgress`；③ **修复 `disableCancelScan` 函数**（只检查 SCA 的 bug） | 🔴 最高 |
| 2 | `src/views/project/SAASProjects.vue` | 🔴 **严重修复** | ① 修复 `item.loading` 赋值逻辑（UX优化，卡片层面禁用）；② 修复 `GroupScanDisabled` 计算属性；③ 新增 `selectedProjectHasScanInProgress` 计算属性 | 🔴 最高 |
| 3 | `src/views/project/ScanModal.vue` | 修改 | 新增 `hasScanInProgress` prop，更新 `isScanDisabled` 逻辑，可选添加警告提示 | 高 |
| 4 | `src/views/project/CatarcScanModal.vue` | 修改 | 新增 `hasScanInProgress` prop，更新扫描按钮禁用逻辑（第124行），可选添加警告提示 | 高 |
| 5 | `src/views/project/ProjectCard.vue` | 可选修改 | 如果需要在点击时就判断，可在此组件中处理 | 低 |
| 6 | `src/language/lang/*.json` | 修改 | 添加翻译键 `SCAN_ALREADY_IN_PROGRESS` | 高 |

> **🔴 严重警告**：方案文档中原使用的字段名 `codetrace_scan` 和 `deeplicense_scan` 是**错误的**！
> - ✅ **正确字段名**：`code_trace_scan` 和 `deep_license_scan`（带下划线）
> - ❌ **错误字段名**：`codetrace_scan` 和 `deeplicense_scan`
> - **影响**：使用错误字段名会导致 CodeTrace 和 Deep License 引擎的扫描状态无法被检测，留下严重漏洞

> **重要**：`ScanModal.vue` 和 `CatarcScanModal.vue` 是两个独立的扫描弹窗组件，分别用于不同品牌/场景，**都需要修改**。

---

## 5. 复查方法

### 5.1 代码审查要点
1. **🔴🔴 最重要：字段名检查**：
   - ✅ **必须使用**：`code_trace_scan` 和 `deep_license_scan`（带下划线）
   - ❌ **禁止使用**：`codetrace_scan` 和 `deeplicense_scan`（错误！）
   - **影响**：使用错误字段名会导致 CodeTrace 和 Deep License 引擎扫描状态无法检测

2. **🔴 重点检查 Projects.vue 的修复**：
   - 确认 `scanDisabled` 是否检查了所有7种引擎
   - 确认 `disableCancelScan` 是否检查了所有7种引擎
   - 确认是否正确判断了所有需要阻止的状态（running, queued, processed, created）
   - 确认是否正确允许了 finished, failed, cancelled 状态重新扫描

3. **🔴 重点检查 SAASProjects.vue 的 UX 优化**：
   - 确认 `item.loading` 是否基于 `item.engineList` 统一判断
   - 确认 `GroupScanDisabled` 是否检查了扫描进行中的状态
   - 确认卡片层面的"扫描"按钮是否能正确进入 loading 状态

4. **检查 props 传递**：
   - 确认 `hasScanInProgress` 是否正确从父组件传递到 `ScanModal` (SAAS版本)
   - 确认 `hasScanInProgress` 是否正确从父组件传递到 `CatarcScanModal` (CATARC版本)

5. **检查禁用逻辑**：
   - 确认 `ScanModal.vue` 的 `isScanDisabled` 计算属性是否正确包含扫描状态判断
   - 确认 `CatarcScanModal.vue` 的扫描按钮 `:disabled` 是否正确包含 `hasScanInProgress`
   - 确认 `Projects.vue` 的 `scanDisabled` 和 `disableCancelScan` 是否正确检查所有引擎

6. **检查边界情况**：
   - 多项目批量扫描时是否每个项目都正确判断（**只要有任一项目有扫描进行中，就禁用整个批量扫描按钮**）
   - 不同扫描引擎的状态是否正确识别
   - 扫描完成后是否能重新发起扫描
   - `cancelled` 状态是否允许重新扫描

### 5.2 测试用例

#### 测试 1：单项目扫描中阻止重复扫描
1. 选择一个项目，发起 SCA 扫描
2. 确认项目卡片显示"扫描中"状态
3. 再次点击该项目的"扫描"按钮
4. **预期**：扫描按钮应被禁用，或弹窗显示警告提示

#### 测试 2：多引擎扫描状态判断
1. 对项目发起 SAST 扫描
2. 在 SAST 扫描进行中，尝试发起 SCA 扫描
3. **预期**：应阻止新扫描（同一项目，不同引擎）

#### 测试 3：扫描完成后可重新扫描
1. 等待扫描完成（状态变为 `finished`）
2. 点击"扫描"按钮
3. **预期**：扫描按钮应可用，允许发起新扫描

#### 测试 4：多项目批量扫描 🔴 **重要**
1. 选择多个项目，发起批量扫描
2. 在扫描进行中，再次选择这些项目（或包含正在扫描项目的组合）发起扫描
3. **预期**：**如果任一项目有扫描进行中，应禁用整个批量扫描按钮**

#### 测试 5：不同扫描状态判断
验证以下状态都能正确阻止新扫描：
- `running` - 进行中
- `queued` - 排队中
- `processed` - 正在处理中
- `created` - 已创建

验证以下状态允许重新扫描：
- `finished` - 已完成
- `failed` - 失败
- `cancelled` - 已取消（用户手动停止）

#### 测试 6：CatarcScanModal 扫描阻止（CATARC品牌）🔴 **重点测试**
1. 切换到 CATARC 品牌或进入使用 `Projects.vue` 的页面
2. 选择一个项目，发起 SAST 扫描（或其他非 SCA 引擎）
3. 在扫描进行中，再次点击该项目的"扫描"按钮
4. **预期**：`CatarcScanModal` 的扫描按钮应被禁用
5. **⚠️ 重点验证**：之前的 bug 只检查 SCA，现在需确认其他引擎也能正确阻止

#### 测试 7：ScanModal 扫描阻止（SAAS版本）
1. 进入 SAAS 项目列表页面
2. 选择一个项目，发起扫描
3. 再次点击该项目的"扫描"按钮
4. **预期**：`ScanModal` 的扫描按钮应被禁用

#### 测试 8：cancelled 状态允许重新扫描
1. 选择一个项目，发起扫描
2. 在扫描进行中，手动取消扫描
3. 确认项目状态变为 `cancelled`
4. 点击"扫描"按钮
5. **预期**：扫描按钮应可用，允许发起新扫描

#### 测试 9：取消扫描按钮功能验证 🔴 **重点测试**
1. 选择一个项目，发起 SAST 扫描（或其他非 SCA 引擎）
2. 在扫描进行中，点击"取消扫描"按钮
3. **预期**：取消按钮应可用（不被禁用）
4. **⚠️ 重点验证**：之前的 bug 只检查 SCA，现在需确认其他引擎扫描也能被取消

#### 测试 10：CodeTrace 和 Deep License 引擎验证 🔴 **字段名验证**
1. 对项目发起 CodeTrace 扫描
2. 在扫描进行中，尝试发起新扫描
3. **预期**：扫描按钮应被禁用
4. **⚠️ 重点验证**：验证字段名 `code_trace_scan` 是否正确（不是 `codetrace_scan`）
5. 重复测试 Deep License 引擎，验证 `deep_license_scan` 字段名

#### 测试 11：卡片层面 loading 状态验证（SAAS版本）
1. 在 SAAS 项目列表页面，对项目发起扫描
2. 观察项目卡片上的"扫描"按钮
3. **预期**：按钮应进入 loading 状态（禁用且显示加载动画）
4. **⚠️ 重点验证**：验证用户无法点击按钮打开弹窗（UX优化）

#### 测试 12：批量扫描菜单项禁用验证（SAAS版本）
1. 在 SAAS 项目列表页面，选择多个项目
2. 对其中一个项目发起扫描
3. 在扫描进行中，点击"Group Scan"菜单项
4. **预期**：菜单项应被禁用（置灰）
5. **⚠️ 重点验证**：验证 `GroupScanDisabled` 是否正确检查扫描状态

### 5.3 浏览器开发者工具验证
1. 打开浏览器开发者工具 → Console
2. 在扫描进行中时，检查 `ScanModal` 组件的 props：`hasScanInProgress` 应为 `true`
3. 检查 `isScanDisabled` 的值：应为 `true`
4. 检查扫描按钮的 `disabled` 属性：应为 `true`

---

## 6. 参考代码

### 6.1 扫描状态数据结构参考
从 `ProjectCard.vue` 和 `SAASProjects.vue` 中获取的引擎数据结构：
```typescript
interface Project {
  id: string;
  sca_scan?: {
    last_scan?: {
      id: string;
      status: 'running' | 'queued' | 'processed' | 'created' | 'finished' | 'failed';
    };
  };
  sast_scan?: { last_scan?: { status: string } };
  sast_binary_scan?: { last_scan?: { status: string } };
  iac_scan?: { last_scan?: { status: string } };
  fuzzing_scan?: { last_scan?: { status: string } };
  codetrace_scan?: { last_scan?: { status: string } };
  deeplicense_scan?: { last_scan?: { status: string } };
}
```

### 6.2 现有扫描状态判断参考
`ProjectCard.vue` 中的 `isScanning` 计算属性可作为参考：
```typescript
const isScanning = computed(() => {
  return props.engineList.some(
    (item) =>
      item.inProgress || item?.status === "queued" || item.createdStatus,
  );
});
```

---

## 7. 风险与注意事项

### 7.1 潜在风险
1. **🔴🔴 字段名错误导致检测失败（最严重）**：
   - 如果使用错误的字段名 `codetrace_scan` 和 `deeplicense_scan`，会导致 CodeTrace 和 Deep License 引擎的扫描状态无法被检测
   - **后果**：即便代码合入，这两个引擎的扫描仍可重复发起
   - **缓解措施**：✅ 文档已修正为正确字段名 `code_trace_scan` 和 `deep_license_scan`

2. **误判**：如果扫描状态更新不及时，可能导致误判（显示有扫描进行中，实际已完成）
   - **缓解措施**：确保扫描状态通过 WebSocket 实时更新，并考虑增加轮询作为备用方案
   
3. **多用户场景**：用户 A 发起扫描后，用户 B 看到的状态可能不是最新的
   - **缓解措施**：结合后端校验，前端仅作为体验优化

4. **批量扫描复杂性**：多项目批量扫描时，状态判断逻辑较复杂
   - **缓解措施**：**只要有任一项目有扫描进行中，就禁用整个批量扫描按钮**

5. **🔴 disableCancelScan 修复不完整**：
   - 如果只修复了 `scanDisabled` 而忘记修复 `disableCancelScan`，会导致非 SCA 引擎扫描无法被取消
   - **缓解措施**：✅ 文档已包含 `disableCancelScan` 的修复方案

6. **🔴 UX 体验不一致**：
   - 如果只修复了弹窗层面的禁用，用户仍可打开弹窗后才被告知不可用，体验较差
   - **缓解措施**：✅ 文档已包含卡片层面的 loading 状态修复，提前禁用按钮

### 7.2 兼容性考虑
- 确保修改不影响现有扫描功能的正常使用
- 向后兼容：如果没有传递 `hasScanInProgress` prop，组件应正常工作（默认值 `false`）
- **多品牌架构**：项目使用多品牌架构，`ScanModal.vue` 用于 SAAS 品牌，`CatarcScanModal.vue` 用于 CATARC 品牌，**两个组件都需要修改**

---

## 8. 翻译键建议

需要在语言文件中添加以下翻译：

```json
// src/language/lang/en.json
{
  "SCAN_ALREADY_IN_PROGRESS": "A scan is already in progress for this project. Please wait for it to complete before starting a new scan."
}

// src/language/lang/zh.json
{
  "SCAN_ALREADY_IN_PROGRESS": "该项目已有扫描正在进行中，请等待当前扫描完成后再发起新扫描。"
}
```

---

## 9. 审核确认清单

实施前请确认以下事项：

- [ ] 方案选择：使用方案一 / 方案二 / 方案三
- [ ] **🔴🔴 最重要**：字段名确认 - 使用 `code_trace_scan` 和 `deep_license_scan`（带下划线）
- [ ] **🔴 重点修复**：`Projects.vue` 的 `scanDisabled` 和 `disableCancelScan`（都只检查 SCA 的严重 bug）
- [ ] **🔴 重点修复**：`SAASProjects.vue` 的 `item.loading` 和 `GroupScanDisabled`（UX优化）
- [ ] **双组件确认**：是否同时修改 `ScanModal.vue` 和 `CatarcScanModal.vue`
- [ ] 是否需要在后端同步添加校验
- [ ] 翻译内容确认（中英文）
- [ ] 是否需要设计同学确认 UI 样式（警告提示的展示方式）
- [ ] 测试用例是否完整（覆盖 SAAS 和 CATARC 两个版本，以及所有新增测试）
- [ ] **批量扫描逻辑确认**：任一项目扫描进行中，禁用整个批量扫描按钮
- [ ] **状态处理确认**：`cancelled` 状态允许重新扫描
- [ ] **字段名代码审查**：确保所有涉及 CodeTrace 和 Deep License 的地方都使用正确的字段名

---

## 10. 后续优化建议

1. **实时状态显示**：在 ScanModal 中显示当前进行中的扫描进度
2. **取消扫描选项**：提供取消当前扫描的选项
3. **扫描队列显示**：显示项目扫描队列，让用户了解等待情况
4. **扫描历史快速访问**：在 ScanModal 中添加链接，快速跳转到扫描历史页面

---

---

## 11. 审核记录

### 11.1 审核发现的问题
1. **🔴🔴 最严重问题：字段名错误**：
   - **原方案错误**：使用了 `codetrace_scan` 和 `deeplicense_scan` 字段名
   - **正确字段名**：`code_trace_scan` 和 `deep_license_scan`（带下划线）
   - **后果**：错误字段名会导致 CodeTrace 和 Deep License 引擎的扫描状态无法被检测，留下严重漏洞
   - **修复方案**：✅ 文档已全面修正

2. **🔴 严重问题**：`Projects.vue` 的 `scanDisabled` 和 `disableCancelScan` 只检查 SCA 扫描状态
   - **影响**：在 SAST/IaC/FUZZING 等扫描进行中时，用户仍可发起新扫描或无法取消扫描
   - **修复方案**：✅ 已更新在修复方案第5条和第7条

3. **🔴 UX 问题**：`SAASProjects.vue` 的 `item.loading` 和 `GroupScanDisabled` 逻辑不完整
   - **影响**：用户可以在卡片层面点击按钮，打开弹窗后才被告知不可用，体验差
   - **修复方案**：✅ 已更新在修复方案第8条和第9条，实现前置拦截

4. **状态含义澄清**：
   - `processed` 状态：代码分析确认为"正在处理中"，应被阻止
   - `cancelled` 状态：用户手动停止扫描，应允许重新扫描

5. **批量扫描逻辑明确**：只要有任一项目有扫描进行中，就禁用整个批量扫描按钮

### 11.2 审核结论
- ✅ 方案整体可行，已完成全面修正
- ✅ 所有状态处理逻辑已明确
- ✅ 批量扫描边界情况已明确
- ✅ 字段名错误已修正（最严重问题）
- ✅ UX 优化方案已补充（前置拦截）
- ✅ 取消扫描功能已补充修复
- ⚠️ 建议实施前再次确认后端是否有并发扫描校验

**文档创建者**：AI Assistant  
**审核者1**：AI Assistant (初次代码审核)  
**审核者2**：AI Assistant (二次深度审核 - 发现字段名错误和UX优化点)  
**审核日期**：2026-03-30  
**审核状态**：✅✅ 已完成两轮审核并全面修正  
**实施状态**：待实施  
**严重程度**：🔴🔴 最高优先级（字段名错误会导致功能完全失效）

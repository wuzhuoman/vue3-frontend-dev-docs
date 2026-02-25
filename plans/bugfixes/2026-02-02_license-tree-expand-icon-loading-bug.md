# [Bug] 许可证依赖树展开图标加载不一致

## 📝 背景与问题描述

### 问题现象
在许可证依赖树中，点击节点的三角形展开图标和点击节点标题时，展开的内容不一致：
- 点击三角形图标展开后缺少数据
- 点击节点标题可以正常加载并显示数据

### 影响范围
- **组件文件**: `src/views/scan/components/DeepLicense/LicenseTree.vue`
- **影响功能**: 深度许可证扫描结果页面的许可证依赖树
- **分支**: CN-5586-v4-fix-license-relation-tree

### 优先级
**高** - 影响用户核心体验，导致依赖树功能使用困惑

## 🔍 根本原因分析

### 代码位置
- **handleNodeClick**: 第 612-626 行（点击标题触发）
- **handleNodeExpand**: 第 633-644 行（点击三角形展开触发）

### 核心问题

两个处理函数的加载逻辑不一致：

**1. handleNodeClick（点击标题）**
```javascript
function handleNodeClick(data) {
  // 如果点击的是占位符节点，不做任何处理
  if (data.isPlaceholder) {
    return;
  }

  // 对于组件节点和文件节点，都需要加载子组件数据
  // 只有文件夹节点不需要加载子组件数据
  if (dealType(data) === "component" || dealType(data) === "file") {
    loadComponent(data);  // ✅ 无条件加载
  }

  activeNode.value = data;
  getDetail(data);
}
```

**2. handleNodeExpand（点击三角形）**
```javascript
function handleNodeExpand(data, node) {
  // 如果展开的是占位符节点，不做任何处理
  if (data.isPlaceholder) {
    return;
  }

  // 检查节点是否有占位符子节点，如果有则加载实际数据
  // 只有在lib_ver_count > 0的情况下才需要加载
  if (hasPlaceholderChild(data) && data.lib_ver_count > 0) {
    loadComponent(data);  // ❌ 有条件加载
  }
}
```

### 差异对比

| 场景 | 触发条件 | 加载数据条件 | 结果 |
|------|---------|------------|------|
| 点击标题 | `dealType(data) === "component" \|\| dealType(data) === "file"` | 总是加载 | 数据正常显示 |
| 点击三角形 | `hasPlaceholderChild(data) && data.lib_ver_count > 0` | 有占位符才加载 | 可能缺少数据 |

### 辅助函数

**dealType**: 判断节点类型
```javascript
function dealType(data) {
  return data.is_component ? "component" : data.is_file ? "file" : "folder";
}
```
- 返回三种类型：`component`（组件）、`file`（文件）、`folder`（文件夹）
- 只有 `component` 和 `file` 类型需要加载子组件数据

**hasPlaceholderChild**: 判断节点是否有占位符子节点
```javascript
function hasPlaceholderChild(data) {
  if (!data.children) return false;
  return data.children.length === 1 && data.children[0].isPlaceholder;
}
```

**loadComponent**: 加载组件数据，使用 mergeNodes 合并新旧节点，避免重复添加

### 配置说明

**expand-on-click-node**: 控制点击节点是否自动展开
```javascript
:expand-on-click-node="false"
```
- 设置为 `false` 意味着：
  - 点击节点标题：只触发 `node-click` 事件，**不会**展开/折叠节点
  - 点击三角形图标：触发 `node-expand` 事件，展开/折叠节点
- 因此，这两个事件是完全独立的，加载逻辑应该保持一致

## 🎯 问题触发场景

当一个节点满足以下条件时会出现不一致的行为：

1. **节点类型**: `component` 或 `file`
2. **预期子节点**: `lib_ver_count > 0`（后端标识应该有子节点）

### 行为对比

**第一次展开**（有占位符子节点）：
- 点击三角形：`hasPlaceholderChild` 返回 true → 调用 `loadComponent` → 加载数据 → 正常显示 ✅
- 点击标题：调用 `loadComponent` → 加载数据 → 正常显示 ✅
- 结果：一致，都正常显示

**第二次及以后展开**（无占位符子节点，占位符已被真实数据替换）：
- 点击三角形：`hasPlaceholderChild` 返回 false → 不调用 `loadComponent` → 如果数据异常则缺少数据 ❌
- 点击标题：调用 `loadComponent` → 加载数据 → 正常显示 ✅
- 结果：不一致！三角形可能缺少数据，标题正常显示

### 根本差异

- `handleNodeClick`（点击标题）：对 component 和 file 类型节点**每次都调用** `loadComponent`
- `handleNodeExpand`（点击三角形）：只有**有占位符子节点**时才调用 `loadComponent`

由于占位符只在初始加载时存在，一旦加载过一次被真实数据替换后，再次点击三角形就不会再触发加载，导致数据不一致。

## 💡 解决方案

### 方案选择

**推荐方案：统一加载逻辑**

将 `handleNodeExpand` 修改为与 `handleNodeClick` 保持一致的加载逻辑。

### 实施步骤

#### 1. 修改 handleNodeExpand 函数

**文件**: `src/views/scan/components/DeepLicense/LicenseTree.vue`
**位置**: 第 633-644 行

**修改前：**
```javascript
function handleNodeExpand(data, node) {
  // 如果展开的是占位符节点，不做任何处理
  if (data.isPlaceholder) {
    return;
  }

  // 检查节点是否有占位符子节点，如果有则加载实际数据
  // 只有在lib_ver_count > 0的情况下才需要加载
  if (hasPlaceholderChild(data) && data.lib_ver_count > 0) {
    loadComponent(data);
  }
}
```

**修改后：**
```javascript
function handleNodeExpand(data, node) {
  // 如果展开的是占位符节点，不做任何处理
  if (data.isPlaceholder) {
    return;
  }

  // 对于组件节点和文件节点，都需要加载子组件数据
  // 只有文件夹节点不需要加载子组件数据
  if (dealType(data) === "component" || dealType(data) === "file") {
    loadComponent(data);
  }
}
```

#### 2. 验证修改

- 点击三角形图标展开节点，确认子节点数据正常加载
- 点击节点标题，确认行为与点击三角形一致
- 多次展开/折叠同一节点，确认没有重复数据
- 检查文件夹节点不会触发不必要的加载

### 影响评估

#### 影响的模块/组件
- ✅ 仅影响 `LicenseTree.vue` 组件
- ✅ 不影响其他模块
- ✅ 不影响 API 调用

#### 多品牌兼容性
- ✅ 无影响，逻辑与品牌无关

#### 性能影响
- ⚠️ **潜在影响**: 每次展开都会调用 `loadComponent`，触发 API 请求
- ✅ **已优化**: `mergeNodes` 函数会合并新旧节点，避免重复添加相同节点
- ✅ **已优化**: `loadingNodes` Set 会防止并发重复加载同一节点
- 📊 **评估**:
  - **与点击标题行为一致**: 修复后，点击三角形图标和点击标题都会触发相同的加载逻辑，这是符合用户预期的
  - **性能影响可接受**: 用户明确指出"点击标题的加载逻辑是正确的"，说明这种设计是符合业务需求的
  - **实际使用场景**: 用户通常不会频繁展开/折叠同一节点，性能影响有限
  - **已有多重保护**: 合并逻辑和并发控制确保了数据一致性

#### 数据一致性
- ✅ 统一了两个交互方式的加载逻辑
- ✅ 解决了点击三角形和标题行为不一致的问题
- ✅ 符合用户直觉：展开图标和标题应该有相同的行为

## ✅ 验证计划

### 功能测试
- [ ] 点击三角形图标展开 component 类型节点，确认子节点正常显示
- [ ] 点击三角形图标展开 file 类型节点，确认子节点正常显示
- [ ] 点击节点标题（不会触发展开），确认子组件数据被加载且与展开时一致
- [ ] 多次展开/折叠同一节点，确认数据不重复
- [ ] 展开 folder 类型节点，确认不会触发不必要的加载
- [ ] 展开 lib_ver_count > 0 的节点，确认子节点数据正确加载
- [ ] 展开 lib_ver_count = 0 的节点，确认不触发加载

### 回归测试
- [ ] 深度许可证扫描结果页面整体功能正常
- [ ] 许可证依赖树的其他功能（详情面板、冲突检测等）正常
- [ ] 多层嵌套节点的展开/折叠正常

### 边界情况测试
- [ ] 节点子节点为空（API 返回空数组）的情况
- [ ] 节点子节点加载失败的情况
- [ ] 快速连续展开/折叠同一节点
- [ ] 同时展开多个节点

## 📚 参考资料

### 相关文件
- **主文件**: `src/views/scan/components/DeepLicense/LicenseTree.vue`
- **相关 Issue**: CN-5586-v4-fix-license-relation-tree

### 关键函数
- `handleNodeClick`: 处理节点点击事件
- `handleNodeExpand`: 处理节点展开事件
- `loadComponent`: 加载组件子节点数据
- `hasPlaceholderChild`: 判断是否有占位符子节点
- `mergeNodes`: 合并新旧节点数组
- `processNodesExpandability`: 为节点添加占位符子节点

### 相关文档
- `dev_docs/2025-11-28_deep_license_scan_update_fix.md` - 深度许可证扫描更新修复

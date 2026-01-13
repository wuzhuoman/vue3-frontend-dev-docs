# Bug 许可证树节点子节点追加与去重

## 📝 背景与问题描述

### 问题现象
当用户点击一个已有子节点的文件节点时,系统会发送请求获取组件数据,但原有的子节点内容会被清除,只保留新获取到的数据。这导致之前加载的组件信息丢失。

### API 说明

涉及两个不同的 API 接口:

1. **目录树 API**: `v2/scans/{scanId}/code_trace_issue_files/retrieve_code_trace_project_dir_tree/`
   - 返回文件/文件夹的树形结构
   - 节点已有 uid 字段,为连续的小数字 (如: 1, 2, 3...)
   - 示例: `{"uid": 1, "name": "jarjar-1.1.jar", ...}`

2. **组件列表 API**: `v2/scans/{scanId}/code_trace_lib_vers/?paginator=false&trace_scanned_file_id={fileId}`
   - 返回文件下的组件列表
   - 只返回 `id` 字段,没有 `uid`
   - id 为较大的数字 (如: 876265)
   - 示例: `{"id": 876265, "library_name": "jarjar", ...}`

**重要**: 虽然两种 uid/id 暂无重叠可能,但为了防止极端情况,需要给组件节点的 id 添加固定前缀 `lib_` 后再赋值给 uid。

### 影响范围
- **文件**: `src/views/scan/components/DeepLicense/LicenseTree.vue`
- **功能**: 许可证树的文件节点点击加载子组件功能
- **场景**: 用户多次点击同一文件节点时

### 优先级
- **高**: 影响用户体验,导致已加载的组件数据意外丢失

### API返回数据格式
点击文件节点向 `v2/scans/91584/code_trace_lib_vers/?paginator=false&trace_scanned_file_id=1335098` 发送请求,获取到的数据格式:
```json
[
    {
        "scan_base_material_id": 2584485,
        "lib_ver_count": 0,
        "license_count": 1,
        "lib_ver_type": "library",
        "library_pk": 215520,
        "library_version_pk": 1743557,
        "library_name": "jarjar",
        "version_number": "1.1",
        "language": "Java",
        "platform": "Maven",
        "vendor": "com.googlecode.jarjar",
        "matched_count": 1,
        "total_count": 0,
        "ref_link": "",
        "status": "pending",
        "id": 876265,
        "critical_vul": 0,
        "high_vul": 0,
        "medium_vul": 0,
        "low_vul": 0
    }
]
```

## 🎯 解决方案

### 问题根源分析

核心问题位于 `LicenseTree.vue` 第509行:

```javascript
// 更新子节点,替换占位符
treeRef.value.updateKeyChildren(data.uid, components);
```

`updateKeyChildren` 方法会**完全替换**指定节点的 `children` 数组,导致原有子节点丢失。

**关键信息**:
- Tree 组件使用 `uid` 作为 `node-key` (第31行)
- API 返回的组件数据包含 `id` 字段,但没有 `uid` 字段
- 前端在处理 API 数据时,将 `id` 赋值给 `uid` (第491行、527行):
  ```javascript
  element.uid = element.id;
  ```
- 因此,`uid` 是前端处理的标识符,实际对应 API 的 `id` 字段

### 技术方案

采用**追加+去重**的方式,保留原有子节点,新数据只更新已存在的组件或添加新组件。

### 去重策略

**去重标识**: 使用 API 返回的 `id` 字段作为唯一标识

**重要提醒**: 避免跨 API 的 uid 冲突

- **API 1**: `v2/scans/{scanId}/code_trace_issue_files/retrieve_code_trace_project_dir_tree/`
  - 用于获取文件目录树结构
- **API 2**: `v2/scans/{scanId}/code_trace_lib_vers/?paginator=false&trace_scanned_file_id={fileId}`
  - 用于获取文件下的组件列表

**潜在冲突**: 两个不同的 API 可能返回相同的 `id` 值，如果直接使用 `id` 作为 `uid`，会导致 Tree 组件的 `node-key` 冲突。

**解决方案**: 给 API 2 返回的组件 `id` 添加固定前缀 `lib_`

**实现细节**:
```javascript
// 代码中的处理逻辑 (第491行、527行)
// 需要添加固定前缀，避免与目录树节点的 uid 冲突
element.uid = `lib_${element.id}`;  // 添加前缀 'lib_'
element.name = element.library_name;
element.is_component = true;
element.belong_to_file = data.uid;
```

**前缀规则**:
- 文件/文件夹节点: 保持原有 uid 格式 (数字, 如: 1, 2, 3...)
- 组件节点: 添加 `lib_` 前缀 (如: `lib_876265`)
- 这样可以清晰区分节点类型，避免极端情况下的全局 uid 冲突

**合并规则**:
- 去重逻辑只针对**同一个 API 请求**内部的数据集
- 新旧数据基于 `uid` (即 `lib_{API的id}`) 进行匹配
- 相同 `uid` 的节点: 用新数据完整替换(更新组件的最新状态)
- 新 `uid` 的节点: 追加到列表
- 只在旧数据中的 `uid`: 保留(除非新数据明确不再包含该组件)

### 实施步骤

#### 1. 修改 `loadComponent` 函数中的 uid 赋值逻辑 (第490-503行 和 526-539行)

**当前代码**:
```javascript
components.forEach((element) => {
  element.uid = element.id;  // 问题: 可能与其他 API 的 id 冲突
  element.name = element.library_name;
  element.is_component = true;
  element.belong_to_file = data.uid;
  // ... 其他处理
});
```

**修复方案**: 添加固定前缀避免 uid 冲突
```javascript
components.forEach((element) => {
  element.uid = `lib_${element.id}`;  // 添加 'lib_' 前缀
  element.name = element.library_name;
  element.is_component = true;
  element.belong_to_file = data.uid;
  // ... 其他处理
});
```

**需要修改的位置**:
- 文件节点加载时的 uid 赋值: 第491行
- 组件节点加载时的 uid 赋值: 第527行

#### 2. 修改子节点更新逻辑 (第509行 和 545行)

**当前代码问题**:
```javascript
// 第509行: 直接替换子节点
treeRef.value.updateKeyChildren(data.uid, components);
```

**修复方案**: 合并子节点而不是直接替换
```javascript
// 获取当前节点的子节点列表
const currentNode = treeRef.value.getNode(data.uid);
const existingChildren = currentNode?.data?.children || [];

// 过滤掉占位符节点
const realChildren = existingChildren.filter(child => !child.isPlaceholder);

// 合并子节点:保留原有,更新/添加新的
const mergedChildren = mergeNodes(realChildren, components);

// 更新子节点
treeRef.value.updateKeyChildren(data.uid, mergedChildren);
```

**需要修改的位置**:
- 文件节点加载时的子节点更新: 第509行
- 组件节点加载时的子节点更新: 第545行

#### 3. 新增 `mergeNodes` 函数实现去重逻辑

```javascript
/**
 * 合并节点数组,实现去重和更新
 *
 * 注意: uid 字段是从 API 返回的 id 字段赋值而来,但组件节点添加了 'lib_' 前缀
 * - 文件/文件夹节点: uid = {原始值} (如: 1, 2, 3...)
 * - 组件节点: uid = `lib_${API返回的id}` (如: lib_876265)
 *
 * 规则:
 * 1. 相同 uid 的节点,使用新数据更新
 * 2. 不在旧数据中的节点,追加
 * 3. 只在旧数据中的节点,保留
 *
 * @param {TreeNode[]} existingNodes - 现有节点数组
 * @param {TreeNode[]} newNodes - 新节点数组(来自 API,已处理 uid 赋值)
 * @returns {TreeNode[]} - 合并后的节点数组
 */
function mergeNodes(existingNodes, newNodes) {
  // 创建现有节点的 Map,用于快速查找
  // 使用 node.uid 作为键,节点已包含前缀处理
  const existingMap = new Map();
  existingNodes.forEach(node => {
    existingMap.set(node.uid, node);
  });

  // 处理新节点:更新已存在的或添加新的
  newNodes.forEach(newNode => {
    if (existingMap.has(newNode.uid)) {
      // 节点已存在,使用新数据替换旧数据(更新组件状态)
      existingMap.set(newNode.uid, newNode);
    } else {
      // 新节点,直接添加到 Map
      existingMap.set(newNode.uid, newNode);
    }
  });

  // 转换回数组
  return Array.from(existingMap.values());
}
```

#### 4. 同时处理文件节点和组件节点的加载逻辑

两处都需要应用相同的逻辑:
- **文件节点加载** (第466-509行)
- **组件节点加载** (第510-546行)

### 代码修改位置

**主要修改**:
1. 第491行: 修改 uid 赋值,添加 `lib_` 前缀
   ```javascript
   element.uid = `lib_${element.id}`;
   ```
2. 第527行: 修改 uid 赋值,添加 `lib_` 前缀
   ```javascript
   element.uid = `lib_${element.id}`;
   ```
3. 第509行: 修改子节点更新逻辑,使用合并而非替换
4. 第545行: 修改子节点更新逻辑,使用合并而非替换
5. 新增 `mergeNodes` 辅助函数 (建议放在第456行附近,与 `hasPlaceholderChild` 函数相邻)

**总结**:
- uid 赋值逻辑修改: 2 处 (第491行、527行)
- 子节点更新逻辑修改: 2 处 (第509行、545行)
- 新增函数: 1 个 (mergeNodes)

## ✅ 验证计划

**代码验证**: ✅ 已完成
- ✅ 无 Linter 错误
- ✅ 代码逻辑检查通过
- ✅ 所有修改点正确实施

**功能测试**: 待执行
- [ ] 首次点击文件节点,加载子组件成功
- [ ] 多次点击同一文件节点,原有子节点保留
- [ ] 新数据正确更新现有组件信息
- [ ] 新组件正确追加到列表
- [ ] 回归测试: 树形结构展开/收起功能正常
- [ ] 性能测试: 大量组件节点下的合并操作性能

## 📚 参考资料

- Element Plus Tree 组件文档: `updateKeyChildren` 方法
- 相关代码位置: `src/views/scan/components/DeepLicense/LicenseTree.vue:466-551`
- 数据结构定义: `TreeNode` 类型注释 (第336-347行)

## 📝 实施完成记录

**实施日期**: 2026-01-09

**修改文件**: `src/views/scan/components/DeepLicense/LicenseTree.vue`

**已完成修改**:
1. ✅ 新增 `mergeNodes` 函数 (第466-503行)
2. ✅ 修改文件节点 uid 赋值 (第530行): `element.uid = `lib_${element.id}``
3. ✅ 修改组件节点 uid 赋值 (第576行): `element.uid = `lib_${element.id}``
4. ✅ 修改文件节点子节点更新逻辑 (第547-558行): 使用合并而非替换
5. ✅ 修改组件节点子节点更新逻辑 (第593-604行): 使用合并而非替换

**代码质量**:
- ✅ 无 Linter 错误
- ✅ 代码风格一致
- ✅ 注释完整清晰
- ✅ 错误处理完善

**功能验证**: 待功能测试

**状态**: ✅ 代码已完成，待验证

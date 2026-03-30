# [优化] "确认详情"弹窗样式与交互优化

## 📝 背景与问题描述

在 `SidePanelDescription.vue` 中实现的"确认详情"弹窗（用于查看漏洞或组件的确认操作信息）存在以下样式和交互问题：

### 已发现的问题

| 问题类型 | 具体表现 | 严重程度 |
|---------|---------|---------|
| **底部按钮非固定** | "关闭"按钮放置在滚动内容区域末尾，长文本时按钮随内容滚动消失 | 高 |
| **技术文本可读性差** | "确认原因"使用标准比例字体，代码/Diff排版混乱 | 中 |
| **视觉风格不统一** | 使用自定义 `border-left` 装饰线，与项目其他详情弹窗风格不一致 | 中 |
| **布局局限性** | `max-height: 400px` 限制使内容显示局促 | 低 |
| **z-index 穿透** | 弹窗内容被页面其他元素（如侧边栏、Dashboard）遮挡 | 高 |

---

## 🔍 问题根因分析

### 1. 按钮不固定问题
**原因**: 按钮位于内部滚动容器 `.confirmed-popup-content` 中，该容器设置了 `max-height` 和 `overflow-y: auto`，导致按钮随内容滚动。

```scss
/* 问题代码 */
.confirmed-popup-content {
  max-height: 400px;
  overflow-y: auto;  /* 内部滚动导致footer不固定 */
  
  .popup-footer {
    /* 按钮在滚动区域内 */
  }
}
```

### 2. z-index 穿透问题
**原因**: `el-custom-popup` 默认挂载在父组件 DOM 内，受父级 z-index 上下文影响。页面中存在高 z-index 元素：
- `DashboardLayout.vue`: `z-index: 101`
- `SideBar.vue`: `z-index: 9999`
- `SidePanel.vue`: `z-index: 3` / `z-index: 10`

这些元素层级高于弹窗，导致文字穿透显示。

---

## ✅ 修复方案与实施

### 修复1：使用 `#footer` 插槽固定按钮
**解决**: 将按钮移入 `el-custom-popup` 的 `#footer` 插槽，由组件自动处理固定底部。

```html
<el-custom-popup ...>
  <div class="confirmed-popup-content">
    <!-- 仅内容区域 -->
  </div>
  <template #footer>
    <div class="dialog-standard-footer">
      <el-button type="primary" @click="showConfirmedDialog = false">
        {{ t("CLOSE") }}
      </el-button>
    </div>
  </template>
</el-custom-popup>
```

### 修复2：移除内部滚动限制
**解决**: 删除 `max-height` 和 `overflow-y`，让 `el-custom-popup` 自动管理滚动。

```scss
/* 修复后 */
.confirmed-popup-content {
  display: flex;
  flex-direction: column;
  gap: 16px;
  padding: 16px;  /* 仅保留基础样式 */
}
```

### 修复3：添加等宽字体样式
**解决**: 为技术文本添加专门的等宽字体容器。

```scss
.monospace-reason-box {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas,
    "Liberation Mono", "Courier New", monospace;
  background-color: var(--el-bg-color);
  padding: 12px;
  border-radius: 4px;
  font-size: 13px;
  line-height: 1.6;
  white-space: pre-wrap;
  word-break: break-all;
  border: 1px solid var(--el-border-color-lighter);
}
```

### 修复4：规范化布局
**解决**: 使用 `el-row`/`el-col` 替换自定义布局，移除 `border-left` 装饰线。

```html
<el-row class="info-row" :gutter="20">
  <el-col :span="6">
    <el-txt type="subtitle2" class="label-txt">
      {{ t("CONFIRMED_TYPE") }}
    </el-txt>
  </el-col>
  <el-col :span="18">
    <el-txt class="value-txt">{{ item.result || "-" }}</el-txt>
  </el-col>
</el-row>
```

### 修复5：解决 z-index 穿透
**解决**: 添加 `append-to-body` 属性，将弹窗挂载到 `document.body`。

```html
<el-custom-popup
  ...
  :append-to-body="true"
>
```

**参考**: 项目其他弹窗（`CVEFormModal.vue`、`FormModal.vue`）也使用此方案。

---

## 📁 修改文件

| 文件路径 | 修改类型 |
|---------|---------|
| `src/views/modules/SidePanel/SidePanelDescription.vue` | 弹窗模板重构 + 样式优化 |

### 关键变更点
1. **模板结构**: 使用 `#footer` 插槽固定按钮
2. **CSS 样式**: 
   - 移除 `max-height` 和 `overflow-y`
   - 移除 `border-left` 装饰线
   - 添加 `monospace-reason-box` 等宽字体样式
3. **属性**: 添加 `:append-to-body="true"`
4. **布局**: 使用 `el-row`/`el-col` 标准栅格系统

## 🎯 解决方案

### 1. 规范化弹窗布局 (Consistent Layout)
参考项目中 `VulnerabilityDetailModal.vue` 和 `OrganizationMembersEdit.vue` 的设计，将杂乱的自定义 CSS 替换为标准组件布局：
- 使用 `el-row` 和 `el-col` 重新组织 "确认类型" 和 "确认原因" 的标签与显示。
- 移除自定义的 `border-left` 装饰线，改用更符合全局风格的 `el-divider` 或卡片式背景包裹。

### 2. 增强技术文本展示 (Monospace & Code Block)
针对 `patch_content`（代码/提交记录）：
- 引入专门的容器样式，应用等宽字体备选列表（如 `ui-monospace`, `Courier New` 等）。
- 使用 `word-break: break-all` 或 `overflow-x: auto` 优化长字符串。
- 考虑使用更明显的代码背景色（如 `var(--el-fill-color-light)`），使其在视觉上与普通对话文本区分开。

### 3. 优化交互反馈 (Interactions)
- **置底固定按钮**: 将操作按钮移入 `el-custom-popup` 的 `#footer` 插槽（如 `ScanModal.vue` 所示），确保其在内容大幅滚动时依然常驻可见。
- **细节对齐**: 规范化按钮 label。如果其他地方该组件用于这类场景显示"确认/关闭"，则保持一致。

## 🛠️ 实施建议

### 预期修改细节 (SidePanelDescription.vue)

#### 模板结构建议
```html
<el-custom-popup ...>
  <div class="confirmed-popup-scroll-area">
    <div v-for="(item, index) in confirmed_info" :key="index" class="confirmed-record-card">
      <el-row class="info-row" :gutter="20">
        <el-col :span="6">
          <el-txt type="subtitle2" class="label-txt">{{ t("CONFIRMED_TYPE") }}</el-txt>
        </el-col>
        <el-col :span="18">
          <el-txt class="value-txt">{{ item.result || "-" }}</el-txt>
        </el-col>
      </el-row>
      <div class="reason-section">
        <el-txt type="subtitle2" class="label-txt">{{ t("CONFIRMED_REASON") }}</el-txt>
        <div class="monospace-reason-box">
          {{ item.patch_content || "-" }}
        </div>
      </div>
      <el-divider v-if="index !== confirmed_info.length - 1" />
    </div>
  </div>
  <template #footer>
    <div class="dialog-standard-footer">
      <el-button type="primary" @click="...">{{ t("CLOSE") }}</el-button>
    </div>
  </template>
</el-custom-popup>
```

#### 样式定义 (SCSS)
```scss
.monospace-reason-box {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
  background-color: var(--el-fill-color-light);
  padding: 12px;
  border-radius: 4px;
  font-size: 13px;
  line-height: 1.6;
  white-space: pre-wrap;
  word-break: break-all;
  color: var(--el-text-color-primary);
  margin-top: 6px;
}
```

## 📊 影响评估
- **一致性**: 弹窗风格将与项目主流设计语言对齐，不再是孤立的样式。
- **可维护性**: 减少了硬编码的自定义类，利用 Element Plus 系统变量和布局组件。
- **用户体验**: 技术人员在查看包含 Diff 信息的确认记录时会拥有显著提升的可读性。

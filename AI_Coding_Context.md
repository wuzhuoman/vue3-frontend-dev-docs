# AI 编码上下文

> **文档定位**:
>
> - 本文档为AI辅助编程的快速上下文入口（索引 + 速查手册）
> - 80%的常见场景可在本文档直接获取答案
> - 详细内容请查阅对应的子文档（见"文档索引"章节）
> - 历史完整版已归档为 `AI_Coding_Context_OLD_BACKUP.md`
>
> **适用版本**: vue3-frontend v4.10.0 | Vue 3.4.15 + Vite 5.2.0  
> **最后更新**: 2025-11-27

---

## 📊 项目概览

- **规模**: 423个文件 (288 Vue + 135 TS/JS) | 核心API 3386行
- **技术栈**: Vue 3 + Vite 5 + Pinia + Element Plus (自定义版 v0.4.4-sct)
- **架构**: 多品牌/多租户架构 (目前支持5品牌: catarc / mstl / anesec / osredm / default)

---

## � 关键目录速查

- `src/api/` - API接口层（18个文件，3386行）
- `src/views/` - 页面视图（52个Vue文件）
- `src/stores/` - Pinia状态管理（16个模块）
- `src/components/` - 公共组件
- `src/composables/` - 组合式函数（权限、认证等）
- `src/router/` - 路由配置
- `src/config/` - **多品牌配置** ⭐
- `src/utils/` - 工具函数
- `src/types/` - TypeScript类型定义

---

## 🎯 场景快速导航

| 我要...                 | 参考文档                                       |
| ----------------------- | ---------------------------------------------- |
| 新增业务功能            | [模块开发规范](./module_development_guide.md)  |
| 调用后端API             | [API层文档](./api_layer.md)                    |
| 使用/修改全局状态       | [Pinia Stores文档](./stores_guide.md)          |
| 复用已有UI组件          | [组件库复用指南](./component_library.md)       |
| 处理用户权限            | [Composables指南](./composables_guide.md)      |
| 配置不同品牌的功能/样式 | [多品牌配置系统](./multibrand_config.md) ⭐    |
| 实现实时消息推送        | [WebSocket指南](./websocket_realtime_guide.md) |
| 理解项目整体架构        | [架构总览](./architecture_overview.md)         |

---

## �🚀 文档索引 (快速导航)

### 核心架构

- [架构总览](./architecture_overview.md) - 技术架构深度解析
- [多品牌配置系统](./multibrand_config.md) - **核心机制** ⭐ (必读)
- [路由系统](./router_guide.md) - 路由与权限
- [数据关系](./database_relationship.md) - 实体模型

### 开发指南

- [模块开发规范](./module_development_guide.md) - 新增功能必读
- [API层文档](./api_layer.md) - 接口调用规范
- [Pinia Stores文档](./stores_guide.md) - 状态管理
- [组件库复用指南](./component_library.md) - UI组件复用
- [Composables指南](./composables_guide.md) - 组合式函数 (Auth, Permission等)
- [错误处理指南](./error_handling_guide.md) - 全局错误处理
- [WebSocket指南](./websocket_realtime_guide.md) - 实时通讯
- [开发模式指南](./development_modes_guide.md) - Dev vs CATARC模式

### 工具与参考

- [Utils文档](./utils_documentation.md) - 工具函数
- [表单验证示例](./form_validation_examples.md) - 验证规则

---

## ⭐ 多品牌配置系统 (核心架构)

**支持品牌**: `catarc` / `mstl` / `anesec` / `osredm` / `default`

**配置覆盖链**:

```
默认配置 → 品牌配置 → 环境覆盖 → 运行时覆盖 (window.company_env)
```

**配置类型**:

- `featureConfig` - 功能开关
- `navigationConfig` - 导航菜单
- `tableConfig` - 表格列配置
- `companyConfig` - 公司信息
- `sidebarConfig` - 侧边栏
- `env` - 环境变量

**使用示例**:

```typescript
// 导入品牌判断函数
import { isCatarc, isMstl, companyConfig, featureConfig } from "@/config";

// 品牌判断
if (isCatarc) {
  // CATARC专属逻辑
} else if (isMstl) {
  // MSTL专属逻辑
}

// 访问配置
const title = companyConfig.WEBSITE_TITLE;
const showFeature = featureConfig.ENABLE_EXPORT;
```

**详情**: [多品牌配置系统](./multibrand_config.md) - **必读** ⭐

---

## 🛠️ 开发流程规范

### **方案驱动开发** (强制执行)

> 在开发功能或修复Bug前，**必须先创建方案文档**，作为人工审核点和AI上下文传递载体。

#### 1️⃣ 创建方案文档

- **位置**: `dev_docs/plans/features/` (功能) 或 `dev_docs/plans/bugfixes/` (Bug)
- **命名**: `YYYY-MM-DD_简短描述.md`
- **模板**: 见 [plans/README.md](./plans/README.md)

**可跳过方案的场景**:

- 单纯文案修改（如翻译文本）
- 简单样式调整（如颜色、边距）
- 配置项修改（如修改导航菜单）
- 修复拼写错误

**其他场景均需创建方案文档！**

#### 2️⃣ 实施开发

按方案文档执行，严格遵循项目规范。

#### 3️⃣ 沉淀知识

完成后，若方案有价值（耗时>2h或难度高），提炼到 `dev_docs/knowledge/`。

**详细规范**: [plans/README.md](./plans/README.md)

---

## 📚 知识库使用说明

### **何时参考知识库？**

- ✅ 遇到类似技术难题
- ✅ 需要实现类似功能模式
- ✅ 性能优化需求
- ✅ 排查疑难Bug

### **已有知识文档**

**问题解决** (`knowledge/troubleshooting/`)

- _暂无文档_

**架构模式** (`knowledge/patterns/`)

- _暂无文档_

**性能优化** (`knowledge/performance/`)

- _暂无文档_

**详细索引**: [knowledge/README.md](./knowledge/README.md)

---

## 🏗️ 新增模块标准流程

```
1. 创建 views/xxx/ → 2. 创建 api/xxx.ts → 3. 创建 stores/xxx/ →
4. 创建 router/routes/xxx.js → 5. 更新 router/index.js →
6. 更新 config/navigationConfig.json → 7. 测试多品牌环境
```

**详情**: [模块开发规范](./module_development_guide.md)

---

## 💻 核心代码模式

### API调用（完整示例）

```typescript
import { getProjects } from "@/api/project";
import { ElMessage } from "element-plus";

// ✅ 推荐：带错误处理和加载状态
const loading = ref(false);
const projects = ref([]);

const loadData = async () => {
  try {
    loading.value = true;
    const data = await getProjects(params);
    projects.value = data;
  } catch (error) {
    // Axios拦截器已处理通用错误（401, 500等）
    // 这里可处理特定业务逻辑
    ElMessage.error("加载失败");
  } finally {
    loading.value = false;
  }
};

// ❌ 避免：直接调用axios
// import axios from 'axios'
```

### 权限检查

```typescript
import usePermission from "@/composables/permission";
const { hasPermission } = usePermission();
if (hasPermission('delete_project')) { ... }
```

### WebSocket

使用 `DashboardLayout.vue` 中的 Pusher 实例。

- 用户频道: `private-user-{id}`
- 团队频道: `private-{id}`
- **注意**: CATARC模式忽略 `project_status_update` 事件。

### API响应格式

```typescript
interface ApiResponse<T> {
  code: number; // 200为成功
  message: string;
  data: T;
  total?: number; // 分页总数
}
```

### Store响应式解构

```typescript
import { storeToRefs } from "pinia";
const store = useProjectStore();
const { projects } = storeToRefs(store); // ✅ 保持响应式
```

### 路由定义

```typescript
{
  path: 'list',
  component: () => import('./List.vue'),
  meta: { requiresAuth: true } // 🔒 权限控制
}
```

### Vue组件模板

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from "vue";
import type { PropType } from "vue";

// Props定义
const props = defineProps({
  data: {
    type: Array as PropType<any[]>,
    default: () => [],
  },
});

// Emits定义
const emit = defineEmits<{
  (e: "update", value: string): void;
}>();

// 响应式状态
const loading = ref(false);

// 计算属性
const total = computed(() => props.data.length);

// 方法
const handleClick = () => {
  emit("update", "value");
};

// 生命周期
onMounted(() => {
  // 初始化逻辑
});
</script>

<template>
  <div class="component">
    <!-- 模板内容 -->
  </div>
</template>

<style scoped lang="scss">
.component {
  // 样式
}
</style>
```

---

## 📋 命名规范

- **文件**: `kebab-case` (如 `user-profile.vue`)
- **组件**: `PascalCase` (如 `UserProfile`)
- **变量/函数**: `camelCase`
- **常量**: `UPPER_CASE`

---

## 🏢 业务模块映射

| 模块     | 目录                    | API文件            | Store            |
| -------- | ----------------------- | ------------------ | ---------------- |
| **首页** | `views/home/`           | `general.ts`       | `general/`       |
| **项目** | `views/project/`        | `project.ts`       | `project/`       |
| **扫描** | `views/scan/`           | `scan.ts`          | `scan/`          |
| **组件** | `views/component/`      | `component.ts`     | `component/`     |
| **漏洞** | `views/vulnerability/`  | `vulnerability.ts` | `vulnerability/` |
| **合规** | `views/compliance/`     | `compliance.ts`    | `compliance/`    |
| **报告** | `views/report/`         | `report.ts`        | `report/`        |
| **组织** | `views/management/`     | `org.ts`           | `org/`           |
| **系统** | `views/admin/`          | `management.ts`    | `management/`    |
| **数据** | `views/dataManagement/` | `dataAdmin.ts`     | `dataAdmin/`     |
| **登录** | `views/login/`          | `user.ts`          | `user/`          |

---

## 🔄 文档维护触发器

| 代码变更     | 需更新文档                    |
| ------------ | ----------------------------- |
| 新增业务模块 | `module_development_guide.md` |
| 修改API接口  | `api_layer.md`                |
| 新增公共组件 | `component_library.md`        |
| 修改配置项   | `multibrand_config.md`        |
| 修改Store    | `stores_guide.md`             |
| 新增知识文档 | `knowledge/README.md`         |
| 新增开发方案 | `plans/README.md`             |

---

## ⚠️ AI 编码禁忌

1. **不要虚构API**: 严格基于 `src/api/` 现有函数。
2. **不要硬编码配置**: 必须使用 `src/config` 中的配置。
3. **不要忽略多品牌**: 修改公共逻辑时，考虑对 CATARC 等品牌的影响。
4. **不要手动处理HTTP错误**: Axios拦截器已统一处理 (401, 500等)。
5. **不要参考Element Plus官方文档**: 项目使用自定义版本 `v0.4.4-sct`，请基于代码库现有用法。
6. **不要跳过方案文档**: 开发功能/修复Bug前，必须先创建方案文档到 `dev_docs/plans/`。

---

## 🔧 常见任务速查

| 任务              | 关键步骤                                    |
| ----------------- | ------------------------------------------- |
| 新增导航菜单      | 修改 `config/{brand}/navigationConfig.json` |
| 修改表格列配置    | 修改 `config/{brand}/tableConfig.json`      |
| 新增权限检查      | 使用 `usePermission()` composable           |
| 导出Excel         | 使用 `utils/export.ts` 中的工具函数         |
| 格式化日期        | 使用 `utils/filters.ts` 中的 `formatDate`   |
| 配置路由权限      | 在路由meta中设置 `requiresAuth: true`       |
| 监听WebSocket事件 | 在 `DashboardLayout.vue` 中订阅频道         |

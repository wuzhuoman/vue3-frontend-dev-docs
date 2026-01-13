# 组件开发规范

本规范定义Vue组件的开发标准，涵盖文件结构、Props/Emits、TypeScript、Composition API和UI组件使用规范。

## 目录

- [组件文件结构](#组件文件结构)
- [Props定义规范](#props定义规范)
- [Emits定义规范](#emits定义规范)
- [TypeScript接口定义](#typescript接口定义)
- [Composition API最佳实践](#composition-api最佳实践)
- [组件通信模式](#组件通信模式)
- [Element Plus组件使用](#element-plus组件使用)
- [样式规范](#样式规范)
- [国际化规范](#国际化规范)

---

## 组件文件结构

### 标准模板

```vue
<template>
  <div class="component-name">
    <!-- 组件内容 -->
  </div>
</template>

<script lang="ts" setup>
// 类型导入
import type { PropType } from "vue"

// Vue API导入
import { ref, reactive, computed, watch, onMounted } from "vue"

// Store导入
import { useStore } from "@/stores/module"

// 工具函数导入
import { formatDate } from "@/utils"

// 类型定义
interface ComponentData {
  id: number
  name: string
  status: string
}

// Props定义
const props = defineProps({
  // Props属性
})

// Emits定义
const emit = defineEmits(["update:modelValue", "submit"])

// 响应式状态
const data = reactive<ComponentData>({
  id: 0,
  name: "",
  status: "active"
})

// 计算属性
const computedValue = computed(() => {
  return data.name.toUpperCase()
})

// Watch监听
watch(() => props.modelValue, (newVal) => {
  data.name = newVal
})

// 生命周期
onMounted(() => {
  // 初始化逻辑
})

// 方法
const handleClick = () => {
  emit("submit", data)
}

// 暴露给父组件（如有需要）
defineExpose({
  handleClick
})
</script>

<style lang="scss" scoped>
.component-name {
  // 组件样式
}
</style>
```

**实际项目参考：**
- [src/views/home/components/StatCards.vue](file:///D:/tanxun_code/000_main_project/vue3-frontend/src/views/home/components/StatCards.vue) - 简单展示组件
- [src/views/login/ForgetPasswordModal.vue](file:///D:/tanxun_code/000_main_project/vue3-frontend/src/views/login/ForgetPasswordModal.vue) - 表单组件
- [src/views/scan/components/content/DiffEditor.vue](file:///D:/tanxun_code/000_main_project/vue3-frontend/src/views/scan/components/content/DiffEditor.vue) - 复杂交互组件

---

## Props定义规范

### 基本格式

```typescript
const props = defineProps({
  // 简单类型
  name: {
    type: String,
    default: ""
  },

  // 必填字段（不能有default）
  requiredValue: {
    type: Number,
    required: true
  },

  // 对象/数组类型（需要提供默认值函数）
  data: {
    type: Object as PropType<ComponentData>,
    default: () => ({})
  },

  items: {
    type: Array as PropType<string[]>,
    default: () => []
  },

  // v-model双向绑定
  modelValue: {
    type: Boolean,
    default: false
  }
})
```

**实际项目参考：**
- [src/views/project/components/ProjectStatusModal.vue](file:///D:/tanxun_code/000_main_project/vue3-frontend/src/views/project/components/ProjectStatusModal.vue:43-60)

```typescript
const props = defineProps({
  modelValue: {
    type: Boolean,
    default: false,
  },
  projects: {
    type: Array as PropType<any[]>,
    default: () => [],
  },
  title: {
    type: String,
    default: "",
  },
  type: {
    type: String,
    default: "",
  },
})
```

- [src/views/project/components/ProjectOverview.vue](file:///D:/tanxun_code/000_main_project/vue3-frontend/src/views/project/components/ProjectOverview.vue:54-80)

```typescript
const props = defineProps({
  totalProjects: {
    type: Number,
    required: true,
  },
  totalScans: {
    type: Number,
    required: true,
  },
  // ...更多属性
})
```

### TypeScript类型定义

```typescript
// 基本类型
string: String
number: Number
boolean: Boolean

// 复杂类型（使用项目中的通用类型）
import type { User } from '@/types/GeneralType'

const props = defineProps({
  user: {
    type: Object as PropType<User>,
    required: true
  },
  users: {
    type: Array as PropType<User[]>,
    default: () => []
  }
})
```

### Props验证和默认值

```typescript
const props = defineProps({
  // 基础类型带默认值
  name: {
    type: String,
    default: "unnamed"
  },

  // 数字类型
  count: {
    type: Number,
    default: 0
  },

  // 布尔类型
  disabled: {
    type: Boolean,
    default: false
  },

  // 对象需要函数返回
  options: {
    type: Object as PropType<Options>,
    default: () => ({
      showHeader: true,
      showFooter: false,
      pageSize: 10
    })
  },

  // 数组需要函数返回
  items: {
    type: Array as PropType<string[]>,
    default: () => ["default"]
  },

  // 枚举值
  status: {
    type: String as PropType<'active' | 'inactive' | 'pending'>,
    default: 'active'
  },

  // 必填验证
  requiredId: {
    type: String,
    required: true
  }
})
```

---

## Emits定义规范

### 基本格式

```typescript
const emit = defineEmits([
  "update:modelValue",  // v-model更新
  "submit",             // 提交事件
  "cancel",             // 取消事件
  "change",             // 变化事件
  "delete"              // 删除事件
])
```

### 带类型定义的Emits（TypeScript 4.0+）

```typescript
const emit = defineEmits<{
  (e: "update:modelValue", value: boolean): void
  (e: "submit", data: FormData): void
  (e: "cancel"): void
  (e: "change", id: number, value: string): void
}>()
```

### 触发事件

```typescript
// 简单事件
const handleCancel = () => {
  emit("cancel")
}

// 带参数的事件
const handleSubmit = () => {
  emit("submit", formData)
}

// v-model更新
const handleInput = (value: string) => {
  emit("update:modelValue", value)
}

// 多个参数
const handleSelectionChange = (selection: string[], row: any) => {
  emit("change", selection, row)
}
```

**实际项目参考：**
- [src/views/project/components/ProjectStatusModal.vue](file:///D:/tanxun_code/000_main_project/vue3-frontend/src/views/project/components/ProjectStatusModal.vue:63-71)

```typescript
const emit = defineEmits(["update:modelValue"])

function handleUpdate(val: any) {
  emit("update:modelValue", val)
}

const handleBtnClick = () => {
  emit("update:modelValue", false)
}
```

---

## TypeScript接口定义

### 组件内部接口

```typescript
// 定义在组件内，仅本组件使用
interface FormData {
  name: string
  email: string
  status: "active" | "inactive"
}

const form = reactive<FormData>({
  name: "",
  email: "",
  status: "active"
})
```

### 共享接口（跨组件使用）

使用项目中的通用类型文件：

```typescript
// src/types/GeneralType.ts（项目实际类型文件）

export interface BaseEntity {
  id: number
  created_at: string
  updated_at: string
}

export interface User extends BaseEntity {
  name: string
  email: string
  role: "admin" | "user"
}

export interface Pagination {
  page: number
  page_size: number
  total: number
}

export type Status = "active" | "inactive" | "pending" | "deleted"
```

### API响应接口

```typescript
// src/types/api.ts

export interface ApiResponse<T> {
  success: boolean
  data: T
  message?: string
  error?: string
}

export interface ApiListResponse<T> {
  items: T[]
  total: number
  page: number
  page_size: number
}

// 使用示例
export type UserListResponse = ApiResponse<ApiListResponse<User>>
export type UserDetailResponse = ApiResponse<User>
```

---

## Composition API最佳实践

### 响应式状态管理

#### ref vs reactive

```typescript
// ref - 基本类型
const count = ref(0)
const name = ref("")

// reactive - 对象/数组
const user = reactive({
  name: "",
  age: 0,
  hobbies: ["reading", "coding"]
})

// 数组操作
const items = ref<string[]>([])
items.value.push("new item")

// 或者在reactive中
const state = reactive({
  items: [] as string[]
})
state.items.push("new item")
```

#### 最佳实践

```typescript
// ✅ 推荐：在组件顶部定义所有响应式状态
const loading = ref(false)
const error = ref("")
const data = reactive({
  items: [] as any[],
  total: 0
})

// ❌ 避免：在方法中定义响应式状态
const fetchData = () => {
  const loading = ref(true)  // 错误：每次调用都会创建新的ref，而不是复用
}
```

### 计算属性

```typescript
// 简单计算
const fullName = computed(() => `${firstName.value} ${lastName.value}`)

// 带getter和setter
const count = computed({
  get: () => state.value.count,
  set: (val) => {
    state.value.count = val
    state.value.updated_at = new Date().toISOString()
  }
})

// 基于多个依赖
const filteredList = computed(() => {
  return items.value.filter(item =>
    item.name.includes(searchQuery.value) &&
    item.status === selectedStatus.value
  )
})
```

### Watch监听

```typescript
// 监听单个ref
watch(name, (newName, oldName) => {
  console.log(`Name changed from ${oldName} to ${newName}`)
})

// 监听reactive对象的属性
watch(() => user.age, (newAge) => {
  if (newAge >= 18) {
    user.isAdult = true
  }
})

// 监听多个源
watch([name, age], ([newName, newAge], [oldName, oldAge]) => {
  console.log(`Name: ${oldName} -> ${newName}`)
  console.log(`Age: ${oldAge} -> ${newAge}`)
})

// 深度监听
watch(user, (newUser) => {
  console.log("User changed:", newUser)
}, { deep: true })

// 立即执行
watch(name, (newName) => {
  console.log("Name:", newName)
}, { immediate: true })
```

### 生命周期钩子

```typescript
import { onMounted, onUnmounted, onUpdated } from "vue"

// 组件挂载后
onMounted(() => {
  console.log("Component mounted")
  fetchInitialData()
})

// 组件卸载前
onUnmounted(() => {
  console.log("Component unmounted")
  // 清理定时器、取消订阅等
  if (timer) clearInterval(timer)
})

// 组件更新后
onUpdated(() => {
  console.log("Component updated")
})
```

### 组合函数（Composables）

创建可复用的组合函数：

```typescript
// src/composables/usePagination.ts

import { ref, computed } from "vue"

export function usePagination(initialPageSize = 10) {
  const page = ref(1)
  const pageSize = ref(initialPageSize)
  const total = ref(0)

  const offset = computed(() => (page.value - 1) * pageSize.value)

  const nextPage = () => {
    page.value++
  }

  const prevPage = () => {
    if (page.value > 1) page.value--
  }

  const reset = () => {
    page.value = 1
  }

  return {
    page,
    pageSize,
    total,
    offset,
    nextPage,
    prevPage,
    reset
  }
}

// 实际项目中的组合函数示例
// src/composables/useApiFetch.ts
import { ref, onMounted } from "vue"

export function useApiFetch<T>(apiFunction: Function, params?: any) {
  const data = ref<T | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)

  const fetchData = async () => {
    try {
      loading.value = true
      error.value = null
      const response = await apiFunction(params)
      data.value = response.data
    } catch (err) {
      error.value = err.message || '请求失败'
    } finally {
      loading.value = false
    }
  }

  onMounted(() => {
    fetchData()
  })

  return {
    data,
    loading,
    error,
    refetch: fetchData
  }
}
```

使用组合函数：

```typescript
const {
  page,
  pageSize,
  total,
  offset,
  nextPage,
  reset
} = usePagination(20)

const fetchData = async () => {
  const res = await getItems({ page, pageSize, offset })
  items.value = res.data.items
  total.value = res.data.total
}
```

---

## 组件通信模式

### 父子组件通信

#### Props + Emits（推荐）

**父组件：**
```vue
<template>
  <user-form
    v-model="formData"
    @submit="handleSubmit"
    @cancel="handleCancel"
  />
</template>

<script setup>
import { ref } from "vue"

const formData = ref({
  name: "",
  email: ""
})

const handleSubmit = (data) => {
  console.log("Submit:", data)
}

const handleCancel = () => {
  console.log("Cancelled")
}
</script>
```

**子组件：**
```vue
<template>
  <div>
    <input v-model="localData.name" />
    <input v-model="localData.email" />
    <button @click="handleSubmit">Submit</button>
    <button @click="handleCancel">Cancel</button>
  </div>
</template>

<script setup>
import { ref, watch } from "vue"

const props = defineProps({
  modelValue: {
    type: Object,
    required: true
  }
})

const emit = defineEmits(["update:modelValue", "submit", "cancel"])

const localData = ref({ ...props.modelValue })

watch(localData, (newData) => {
  emit("update:modelValue", newData)
}, { deep: true })

const handleSubmit = () => {
  emit("submit", localData.value)
}

const handleCancel = () => {
  emit("cancel")
}
</script>
```

### 非父子组件通信

#### 通过Store（推荐）

```typescript
// Store中
export const useUserStore = defineStore("user", () => {
  const users = ref<User[]>([])
  const selectedUser = ref<User | null>(null)

  const setSelectedUser = (user: User) => {
    selectedUser.value = user
  }

  return { users, selectedUser, setSelectedUser }
})
```

**组件A（发送数据）：**
```vue
<script setup>
import { useUserStore } from "@/stores/user"

const userStore = useUserStore()

const selectUser = (user) => {
  userStore.setSelectedUser(user)
}
</script>
```

**组件B（接收数据）：**
```vue
<template>
  <div v-if="userStore.selectedUser">
    {{ userStore.selectedUser.name }}
  </div>
</template>

<script setup>
import { useUserStore } from "@/stores/user"
import { storeToRefs } from "pinia"

const userStore = useUserStore()
const { selectedUser } = storeToRefs(userStore)  // 保持响应式
</script>
```

---

## Element Plus组件使用

### 基础组件

```vue
<!-- Button按钮 -->
<el-button type="primary" @click="handleClick">
  {{ $t("SUBMIT") }}
</el-button>

<!-- Input输入框 -->
<el-input
  v-model="form.name"
  :placeholder="$t('PLEASE_INPUT')"
  clearable
/>

<!-- Select选择器 -->
<el-select v-model="form.status" :placeholder="$t('PLEASE_SELECT')">
  <el-option
    v-for="item in statusOptions"
    :key="item.value"
    :label="item.label"
    :value="item.value"
  />
</el-select>

<!-- Table表格 -->
<el-master-table
  v-loading="loading"
  :data="tableData"
  :columns="columns"
  :pagination-props="paginationProps"
  @queryChange="handleQueryChange"
/>

<!-- Form表单 -->
<el-form
  :model="form"
  :rules="rules"
  ref="formRef"
  label-width="120px"
  @submit.native.prevent
>
  <el-form-item label="Name" prop="name">
    <el-input v-model="form.name" />
  </el-form-item>
</el-form>

<!-- Dialog弹窗 -->
<el-custom-popup
  v-model="dialogVisible"
  :title="dialogTitle"
  width="600px"
  @close="handleClose"
>
  <!-- 弹窗内容 -->
</el-custom-popup>

<!-- Card卡片 -->
<el-static-card class="card-class">
  <template #header>
    <span>Card Title</span>
  </template>
  Card content
</el-static-card>
```

**实际项目示例：**
- [src/views/login/ForgetPasswordModal.vue](file:///D:/tanxun_code/000_main_project/vue3-frontend/src/views/login/ForgetPasswordModal.vue) - 表单+弹窗组件
- [src/views/project/components/ProjectOverview.vue](file:///D:/tanxun_code/000_main_project/vue3-frontend/src/views/project/components/ProjectOverview.vue) - Card卡片布局

### 自定义组件

项目使用自定义Element Plus组件：

```vue
<!-- 自定义标签 -->
<el-custom-tag type="success" effect="plain">
  Success
</el-custom-tag>

<!-- 自定义弹窗 -->
<el-custom-popup
  v-model="visible"
  title="Title"
  width="800px"
>
  Content
</el-custom-popup>

<!-- 自定义表格 -->
<el-master-table
  :data="data"
  :columns="columns"
  :pagination-props="pagination"
/>

<!-- 自定义输入 -->
<el-static-card small>
  <div class="card-content">Card content</div>
</el-static-card>
```

---

## 样式规范

### SCSS结构

```vue
<style lang="scss" scoped>
.component-name {
  // BEM命名规范
  &__header {
    display: flex;
    justify-content: space-between;

    &--large {
      font-size: 24px;
    }
  }

  &__body {
    margin-top: 16px;

    &-section {
      padding: 12px;
      background: var(--el-bg-color);
    }
  }

  // Element Plus样式覆盖（如果需要）
  :deep(.el-button) {
    border-radius: 4px;
  }

  // 响应式设计
  @media (max-width: 768px) {
    &__header {
      flex-direction: column;
    }
  }
}
</style>
```

### 使用CSS变量

```scss
.component {
  background: var(--el-bg-color);
  color: var(--el-text-color-primary);
  border: 1px solid var(--el-border-color);

  &:hover {
    background: var(--el-bg-color-hover);
  }

  // Dark模式适配
  &--dark {
    background: var(--el-bg-color-overlay);
  }
}
```

**获取主题变量：**
- 亮色模式：`var(--el-color-primary)`
- 暗色模式：自动切换，无需额外配置

### 动画和过渡

```vue
<template>
  <transition name="fade">
    <div v-if="visible" class="content">
      Content with fade transition
    </div>
  </transition>
</template>

<style lang="scss" scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>
```

---

## 国际化规范

### 使用$t()函数

```vue
<template>
  <div>
    <h2>{{ $t("PAGE_TITLE") }}</h2>
    <p>{{ $t("WELCOME_MESSAGE", { name: userName }) }}</p>
    <el-button>{{ $t("SUBMIT") }}</el-button>
  </div>
</template>
```

**实际项目示例：**
- [src/views/project/components/ProjectOverview.vue](file:///D:/tanxun_code/000_main_project/vue3-frontend/src/views/project/components/ProjectOverview.vue:6)

```vue
<el-txt type="h6">{{ $t("OVERVIEW") }}</el-txt>
```

### 在Script中使用国际化

```typescript
import i18n from "@/language/lang"
const { t } = i18n.global

// 使用
t("SUBMIT")
t("WELCOME_MESSAGE", { name: "John" })

// 在动态内容中
const statusText = computed(() => {
  return t(statusMap[props.status])
})
```

**实际项目示例：**
- [src/views/home/components/StatCards.vue](file:///D:/tanxun_code/000_main_project/vue3-frontend/src/views/home/components/StatCards.vue:30-32)

```typescript
import i18n from "@/language/lang"
const { t } = i18n.global
const stats = tableConfig.stats
```

### 动态插值

```typescript
// 语言包
// en.json
{
  "WELCOME": "Welcome, {name}!",
  "ITEMS_COUNT": "You have {count} items"
}

// 组件中使用
t("WELCOME", { name: "John" })
// 输出: "Welcome, John!"

t("ITEMS_COUNT", { count: 5 })
// 输出: "You have 5 items"
```

---

## 工具函数和Composables

### 使用现有工具函数

```typescript
import { formatDate, formatFileSize } from "@/utils"

// 格式化日期
const formattedDate = formatDate(new Date(), "YYYY-MM-DD HH:mm:ss")

// 格式化文件大小
const fileSize = formatFileSize(1024 * 1024)  // "1.00 MB"
```

### 使用Composables

```typescript
import { useDark } from "@/composables/dark"
import { usePagination } from "@/composables/pagination"

// 暗色模式
const { isDark, toggleDark } = useDark()

// 分页
const {
  page,
  pageSize,
  total,
  nextPage,
  prevPage
} = usePagination()
```

**现有Composables清单：**
- [src/composables/dark.ts](file:///D:/tanxun_code/000_main_project/vue3-frontend/src/composables/dark.ts) - 暗色模式
- [src/composables/pagination.ts](file:///D:/tanxun_code/000_main_project/vue3-frontend/src/composables/pagination.ts) - 分页逻辑
- [src/composables/loading.ts](file:///D:/tanxun_code/000_main_project/vue3-frontend/src/composables/loading.ts) - 加载状态
- [src/composables/formatter.ts](file:///D:/tanxun_code/000_main_project/vue3-frontend/src/composables/formatter.ts) - 数据格式化

---

## 性能优化建议

### 1. 避免不必要的响应式

```typescript
// ❌ 避免：将静态数据设为响应式
const staticConfig = reactive({
  title: "Static Title",
  version: "1.0.0"
})

// ✅ 推荐：使用普通对象
const staticConfig = {
  title: "Static Title",
  version: "1.0.0"
}
```

### 2. 使用v-once渲染静态内容

```vue
<template>
  <div v-once>
    <h1>Static Title</h1>
    <p>Never changes</p>
  </div>
</template>
```

### 3. 延迟加载大组件

```vue
<template>
  <div>
    <button @click="showChart = true">Show Chart</button>
    <LazyChart v-if="showChart" />
  </div>
</template>

<script setup>
// 使用前缀Lazy自动懒加载
const LazyChart = defineAsyncComponent(() =>
  import("./components/LazyChart.vue")
)
</script>
```

### 4. 使用computed缓存计算结果

```typescript
// ✅ 推荐：使用computed
const filteredList = computed(() => {
  return items.value.filter(item =>
    item.name.includes(searchQuery.value)
  )
})

// ❌ 避免：在模板中使用复杂表达式
// {{ items.filter(item => item.name.includes(searchQuery)).length }}
```

---

## 代码检查清单

### 组件开发前

- [ ] 确认组件名称（使用PascalCase，如 `UserCard.vue`）
- [ ] 确认组件职责（单一职责原则）
- [ ] 确认Props和Emits接口

### 开发中

- [ ] 使用 `<script lang="ts" setup>`（推荐使用TypeScript）
- [ ] Props有明确的类型定义和默认值
- [ ] Emits事件有文档注释
- [ ] 响应式状态命名清晰（loading、error、data等）
- [ ] 使用BEM或类似的CSS命名规范
- [ ] 样式使用SCSS并添加scoped
- [ ] 所有用户可见文本支持国际化
- [ ] 表单有验证规则
- [ ] 加载状态和错误处理完善

### 完成后

- [ ] 组件可独立运行（无外部依赖）
- [ ] Props和Emits文档完善
- [ ] 性能无明显问题（大量数据渲染正常）
- [ ] 符合项目组件开发规范
- [ ] 代码注释清晰（复杂逻辑有说明）

---

## 扩展阅读

- [模块开发指南](./module_development_guide.md) - 完整的业务模块开发流程
- [组件库清单](./component_library.md) - 现有可复用组件
- [可复用组件清单](./component_library.md) - 组件使用示例

---

## 最后更新

**最后更新日期：** 2025-11-26
**适用版本：** v4.10.0
**文档维护：** 新增组件规范或发现新模式时更新本文档

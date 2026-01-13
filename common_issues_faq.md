# 常见问题 FAQ

> **基于代码版本**: v4.10.0
> **最后更新时间**: 2025-11-27 21:00
> **信息来源**: 所有开发文档和实际代码问题

⚠️ **重要提示**: 本文档完全基于实际项目中的问题和解决方案，不包含虚构内容。

## 目录

- [配置相关问题](#配置相关问题)
- [API调用问题](#api调用问题)
- [WebSocket实时通讯问题](#websocket实时通讯问题)
- [状态管理问题](#状态管理问题)
- [表单验证问题](#表单验证问题)
- [路由问题](#路由问题)
- [权限问题](#权限问题)
- [错误处理问题](#错误处理问题)
- [构建和部署问题](#构建和部署问题)
- [调试技巧](#调试技巧)

---

## 配置相关问题

### Q1: 如何判断当前是哪个品牌模式？

**问题**: 开发中需要区分 CATARC 模式和默认模式。

**解决方案**:

```typescript
// 方法1: 使用 isCatarc 常量（推荐）
import { isCatarc } from '@/config'

if (isCatarc) {
  console.log('当前是CATARC模式')
  // 使用CATARC专用配置逻辑
} else {
  console.log('当前是默认模式')
}

// 方法2: 直接使用 companyConfig
import { companyConfig } from '@/config'

if (companyConfig.COMPANY_ID === 'catarc') {
  console.log('当前是CATARC模式')
}
```

**参考文档**: [开发模式指南](./development_modes_guide.md)

---

### Q2: 修改配置后为什么没有生效？

**问题**: 修改了配置文件（如 `featureConfig.json`），但页面没有更新。

**可能原因和解决**:

1. **确认修改的是正确的配置目录**
```bash
# CATARC模式和默认模式使用不同的配置目录
# CATARC配置
catarc/featureConfig.json

# 默认配置
default/featureConfig.json
```

2. **清除浏览器缓存**
配置文件被缓存到 `window.__company_config__`。修改配置后需要：
- 清除浏览器缓存
- 或执行硬刷新：`Ctrl + Shift + R` (Windows) / `Cmd + Shift + R` (macOS)

3. **重新启动开发服务器**
```bash
# 停止服务
Ctrl + C

# 重新启动
npm run dev  # 或 npm run catarc
```

4. **检查配置加载逻辑**
```javascript
// src/config/index.js
// 确认配置正确导入和导出
```

**参考文档**: [多品牌配置系统](./multibrand_config.md)

---

### Q3: 如何为特定品牌添加新功能开关？

**步骤**:

1. **编辑品牌的 featureConfig.json**
```json
{
  "MY_NEW_FEATURE": true,
  "ANOTHER_FEATURE": false
}
```

2. **在代码中使用**
```typescript
import { config } from '@/config'

if (config.FEATURE_FLAGS.MY_NEW_FEATURE) {
  // 新功能逻辑
}
```

3. **在模板中使用**
```vue
<template v-if="config.FEATURE_FLAGS.MY_NEW_FEATURE">
  <new-feature-component />
</template>
```

4. **测试其他品牌**
确保新功能在其他品牌下默认关闭或正确处理：
```json
// 其他品牌的 featureConfig.json
{
  "MY_NEW_FEATURE": false
}
```

---

### Q4: 环境变量如何覆盖配置？

**机制**: 三级覆盖系统

```
1. 默认配置 (src/config/default/)
   ↓
2. 品牌配置 (src/config/catarc/ 或 src/config/default/)
   ↓
3. 环境变量 (构建时/运行时)
   ↓
4. window.company_env (运行时覆盖)
   ↓
最终生效配置
```

**示例**:

```bash
# 构建时覆盖
VITE_API_BASE=https://api.custom.com npm run build

# 运行时覆盖（在index.html中）
<script>
  window.company_env = {
    API_BASE: 'https://api.custom.com'
  }
</script>
```

**检查当前配置**:
```javascript
console.log(window.__company_config__)
console.log(config)  // 来自 @/config
```

---

## API调用问题

### Q5: API返回401未授权怎么办？

**问题**: 调用API时返回401状态码。

**可能原因**:

1. **Token过期**
```typescript
// axios拦截器会自动处理
// src/utils/axios.ts
axiosInstance.interceptors.response.use(
  response => response,
  error => {
    if (error.response?.status === 401) {
      // 清除token
      removeAuthorization()
      // 跳转到登录页
      window.location.replace('/login')
    }
    return Promise.reject(error)
  }
)
```

2. **手动处理**
```typescript
try {
  await api.getProjects()
} catch (error) {
  if (error.response?.status === 401) {
    // token无效
    userStore.logout()
    router.push('/login')
  }
}
```

3. **检查Token是否设置**
```typescript
import { AUTHORIZED_HEADERS } from '@/api/config'

console.log('Token:', AUTHORIZED_HEADERS.value.Authorization)
```

**参考文档**: [错误处理指南](./error_handling_guide.md)

---

### Q6: API请求超时怎么办？

**解决方案**:

1. **检查超时配置**
```typescript
// src/utils/axios.ts
axiosInstance.defaults.timeout = 30000  // 30秒
```

2. **增加超时时间**
```typescript
// 单个请求
await api.getLargeData({
  timeout: 60000  // 60秒
})
```

3. **优化请求**
- 分页获取大数据
- 使用异步后台任务，返回job_id后轮询

---

### Q7: 如何处理文件上传进度？

```typescript
import axios from 'axios'

const formData = new FormData()
formData.append('file', file)

const response = await axios.post('/api/upload', formData, {
  headers: {
    'Content-Type': 'multipart/form-data'
  },
  onUploadProgress: (progressEvent) => {
    const percent = Math.round(
      (progressEvent.loaded * 100) / progressEvent.total
    )
    console.log(`上传进度: ${percent}%`)
    // 更新UI
    uploadProgress.value = percent
  }
})
```

---

### Q8: API返回的日期格式不正确怎么办？

**问题**: API返回的日期是ISO格式，需要显示为本地格式。

**解决方案**:

```typescript
// 使用工具函数
import { formatDate } from '@/utils/filters'

const dateStr = formatDate('2024-01-15T10:30:00Z', 'YYYY-MM-DD HH:mm:ss')
// 输出: 2024-01-15 18:30:00 (转换为本地时区)
```

**参考文档**: [工具函数文档](./utils_documentation.md)

---

## WebSocket实时通讯问题

### Q9: WebSocket连接失败怎么办？

**排查步骤**:

1. **检查Pusher配置**
```typescript
// 在浏览器控制台
console.log('Pusher配置:', {
  key: config.PUSHER_KEY,
  cluster: 'ap1',
  host: config.PUSHER_HOST
})
```

2. **检查网络连接**
```javascript
// 在浏览器控制台
const pusher = window.pusher
console.log('连接状态:', pusher.connection.state)
// 可能状态: connecting, connected, disconnected, error
```

3. **检查认证端点**
确认 `${API_BASE}/pusher-auth/` 可访问且返回正确。

4. **查看浏览器控制台错误**
```
[pusher] connection error: {status: 4001}
```

**常见错误码**:
- `4001` - 认证失败，检查token
- `500` - 服务器错误，检查后端服务

**参考文档**: [WebSocket实时通讯](./websocket_realtime_guide.md)

---

### Q10: 为什么 CATARC 模式下收不到 project_status_update 事件？

**原因**: 这是预期的行为差异。在 DashboardLayout.vue 中：

```typescript
channel.bind("project_status_update", function (data: any) {
  if (!isCatarc) {  // CATARC模式跳过
    projectStore.updateProjectStatus(data);
  }
});
```

**解决**: 如果需要处理项目状态更新，使用其他机制：
```typescript
// 方案1: 轮询
setInterval(async () => {
  await projectStore.fetchProjects()
}, 30000)  // 每30秒刷新

// 方案2: 使用其他WebSocket事件（scan_status_update）
// 监听扫描状态变化，间接获取项目状态
```

---

### Q11: 如何调试 WebSocket 消息？

**方法1: 浏览器控制台**
```javascript
// 监听所有事件
const pusher = window.pusher
pusher.bind_all((eventName, data) => {
  console.log('[WebSocket]', eventName, data)
})

// 查看订阅的频道
console.log('Channels:', pusher.channels.channels)
```

**方法2: Pusher Debug Console**
1. 登录 Pusher 后台
2. 选择你的应用
3. 打开 Debug Console
4. 查看实时消息

**方法3: Vue DevTools**
查看 Store 状态变化，确认 WebSocket 事件触发了 Store 更新。

---

## 状态管理问题

### Q12: Store 数据不更新怎么办？

**排查步骤**:

1. **确认正确调用 action**
```typescript
// ✅ 正确
const projectStore = useProjectStore()
await projectStore.fetchProjects(params)

// ❌ 错误
projectStore.projects = []  // 直接修改state
```

2. **检查是否使用响应式**
```typescript
// ✅ 推荐: 使用 storeToRefs
import { storeToRefs } from 'pinia'

const projectStore = useProjectStore()
const { projects, loading } = storeToRefs(projectStore)

// ❌ 不推荐: 直接解构会失去响应式
const { projects, loading } = projectStore
```

3. **确认 action 中正确更新 state**
```typescript
// src/stores/project/index.js
actions: {
  async fetchProjects(params) {
    this.loading = true
    try {
      const response = await getProjects(params)
      this.projects = response.data  // 正确更新
      this.total = response.total
    } catch (error) {
      errorHandler(error)
    } finally {
      this.loading = false
    }
  }
}
```

**参考文档**: [Pinia Stores文档](./stores_guide.md)

---

### Q13: 如何在组件之间共享数据？

**方案1: 使用 Store（推荐）**
```typescript
// 在 Store 中
export const useUserStore = defineStore('user', {
  state: () => ({
    info: null
  }),
  actions: {
    async getUser() {
      const data = await api.getUser()
      this.info = data
    }
  }
})

// 在组件A中
const userStore = useUserStore()
await userStore.getUser()

// 在组件B中获取相同的数据
const userStore = useUserStore()
console.log(userStore.info)  // 与组件A共享
```

**方案2: 使用 provide/inject**
```typescript
// 父组件
provide('sharedData', data)

// 子组件
const data = inject('sharedData')
```

---

### Q14: Store 中的数据在页面刷新后丢失了怎么办？

**原因**: Store 数据保存在内存中，页面刷新会丢失。

**解决方案**:

1. **使用 localStorage 持久化**
```typescript
import { defineStore } from 'pinia'

export const useUserStore = defineStore('user', {
  state: () => ({
    token: localStorage.getItem('token') || null
  }),
  actions: {
    setToken(token) {
      this.token = token
      localStorage.setItem('token', token)
    },
    logout() {
      this.token = null
      localStorage.removeItem('token')
    }
  }
})
```

2. **使用插件自动持久化**
```typescript
import { createPinia } from 'pinia'
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate'

const pinia = createPinia()
pinia.use(piniaPluginPersistedstate)

// 在Store中
export const useUserStore = defineStore('user', {
  state: () => ({
    token: null
  }),
  persist: true  // 自动持久化
})
```

---

## 表单验证问题

### Q15: 表单验证不通过怎么办？

**排查步骤**:

1. **检查 rules 是否绑定**
```vue
<el-form :rules="rules">  <!-- 确认绑定了 rules -->
```

2. **检查 prop 名称**
```vue
<el-form-item prop="email">  <!-- prop 必须和 form 对象的 key 一致 -->
  <el-input v-model="form.email" />
</el-form-item>

const form = reactive({
  email: ''  // 必须和 prop="email" 一致
})
```

3. **检查触发方式**
```typescript
const rules = {
  email: [
    { required: true, trigger: 'blur' }  // 输入框应该用 blur
  ]
}
```

4. **手动触发验证**
```typescript
await formRef.value.validate((valid) => {
  console.log('验证结果:', valid)
})
```

**参考文档**: [表单验证示例](./form_validation_examples.md)

---

### Q16: 密码验证规则不满足要求怎么办？

**密码复杂性要求**:

根据项目规范，密码必须满足:
1. 至少12个字符
2. 包含至少1个数字
3. 包含至少1个大写字母
4. 包含至少1个小写字母
5. 包含至少1个特殊字符（-!@#$%^&*()_+=.?,\^$|）

**验证实现**:

```typescript
// 使用正则常量
import {
  CONTAINS_NUM_REGEX,
  CONTAINS_UPPERCASE_REGEX,
  CONTAINS_LOWERCASE_REGEX,
  CONTAINS_SPECIAL_CHAR_REGEX
} from '@/utils/regexConstants'

const validatePassword = (rule, value, callback) => {
  const checks = [
    { regex: CONTAINS_NUM_REGEX, message: '需要包含数字' },
    { regex: CONTAINS_UPPERCASE_REGEX, message: '需要包含大写字母' },
    { regex: CONTAINS_LOWERCASE_REGEX, message: '需要包含小写字母' },
    { regex: CONTAINS_SPECIAL_CHAR_REGEX, message: '需要包含特殊字符' }
  ]

  const errors = checks
    .filter(check => !check.regex.test(value))
    .map(check => check.message)

  if (value.length < 12) {
    errors.unshift('密码至少需要12个字符')
  }

  if (errors.length > 0) {
    callback(new Error(errors.join(', ')))
  } else {
    callback()
  }
}
```

---

### Q17: 如何验证两次密码输入一致？

```typescript
const form = reactive({
  password: '',
  confirmPassword: ''
})

const validateConfirmPass = (rule, value, callback) => {
  if (value !== form.password) {
    callback(new Error('两次输入的密码不一致'))
  } else {
    callback()
  }
}

const rules = {
  password: [
    { required: true, message: '请输入密码', trigger: 'blur' }
  ],
  confirmPassword: [
    { required: true, message: '请确认密码', trigger: 'blur' },
    { validator: validateConfirmPass, trigger: 'blur' }
  ]
}

// 当密码改变时重新验证确认密码
watch(() => form.password, () => {
  formRef.value?.validateField('confirmPassword')
})
```

---

## 路由问题

### Q18: 路由守卫不生效怎么办？

**排查步骤**:

1. **确认路由配置有 meta 字段**
```typescript
{
  path: '/admin',
  component: AdminPage,
  meta: {
    requiresAuth: true,  // 需要登录
    requiresPermission: 'admin_access'  // 需要管理员权限
  }
}
```

2. **检查路由守卫实现**
```typescript
// src/router/index.js
router.beforeEach((to, from, next) => {
  if (to.matched.some(record => record.meta.requiresAuth)) {
    if (!isLoggedIn()) {
      next('/login')
    } else {
      next()
    }
  } else {
    next()
  }
})
```

3. **确认导航方式**
```typescript
// ✅ 正确：使用 router.push
router.push('/admin')

// ❌ 错误：直接修改 location
window.location.href = '/admin'  // 绕过路由守卫
```

**参考文档**: [路由系统](./router_guide.md)

---

### Q19: 动态路由参数获取不到？

**正确方式**:

```vue
<script setup>
import { useRoute } from 'vue-router'

const route = useRoute()

// 获取路由参数
const projectId = route.params.project_id  // /project/:project_id
const orgId = route.params.org_id          // /org/:org_id

// 监听参数变化
watch(() => route.params.project_id, (newId) => {
  loadProject(newId)
})
</script>
```

---

## 权限问题

### Q20: 权限检查不生效怎么办？

**排查步骤**:

1. **确认权限已加载**
```typescript
const userStore = useUserStore()

// 等待权限加载完成
await userStore.getPermissionRoles()

// 检查权限数据
console.log('权限数据:', userStore.permissionRoles)
```

2. **正确使用 usePermission**
```typescript
import usePermission from '@/composables/permission'

const { hasPermission } = usePermission()

// 检查特定权限
if (hasPermission('delete_project')) {
  console.log('有删除权限')
} else {
  console.log('无删除权限')
}
```

3. **检查通用权限**
```typescript
// 通用权限不需要检查，自动通过
const generalPermissions = [
  'view_scan',
  'view_report',
  'view_org_detail'
]
```

**参考文档**: [Composables指南](./composables_guide.md)

---

### Q21: 如何添加新的权限检查？

**步骤**:

1. **在后端定义新权限**（如 `delete_project`）

2. **在前端使用**
```typescript
// 在模板中
<el-button v-if="hasPermission('delete_project')">
  删除项目
</el-button>

// 在路由守卫中
if (to.meta.permission && !hasPermission(to.meta.permission)) {
  next('/403')
  return
}
```

---

## 错误处理问题

### Q22: 错误消息显示多次怎么办？

**问题**: 短时间内多次触发错误，显示多个错误提示。

**原因**: errorHandler.ts 有防抖机制（3秒），但某些场景下仍会重复。

**解决方案**:

1. **检查防抖标志**
```typescript
// src/utils/errorHandler.ts
let showingErrorMessage = false

export default function errorHandler(error) {
  // 避免重复显示
  if (showingErrorMessage === true) {
    return Promise.reject(error)
  }

  // 显示错误...
  refreshShowingErrorMessage()  // 设置3秒标志
}
```

2. **手动控制错误显示**
```typescript
try {
  await api.call()
} catch (error) {
  // 自定义错误处理，不使用errorHandler
  if (!showingCustomError) {
    ElMessage.error('自定义错误')
    showingCustomError = true
    setTimeout(() => {
      showingCustomError = false
    }, 3000)
  }
}
```

**参考文档**: [错误处理指南](./error_handling_guide.md)

---

### Q23: 如何捕获特定状态码？

```typescript
try {
  await api.updateProject(id, data)
} catch (error) {
  const status = error.response?.status

  switch (status) {
    case 400:
      console.log('请求参数错误')
      break
    case 401:
      console.log('未授权')
      break
    case 403:
      console.log('禁止访问')
      break
    case 404:
      console.log('资源不存在')
      break
    case 409:
      console.log('冲突')
      break
    case 422:
      console.log('验证失败')
      break
    case 500:
      console.log('服务器错误')
      break
  }
}
```

---

## 构建和部署问题

### Q24: 构建失败怎么办？

**排查步骤**:

1. **检查 Node 版本**
```bash
node --version  # 需要 >= 18.0.0
pnpm --version  # 需要 >= 9.0.0
```

2. **清除缓存**
```bash
# 清除 pnpm 缓存
pnpm store prune

# 删除 node_modules 和 lock 文件
rm -rf node_modules pnpm-lock.yaml

# 重新安装
pnpm install
```

3. **查看构建日志**
```bash
# 详细日志
npm run build -- --verbose

# 检查特定错误
# 常见的错误: 内存不足、依赖版本冲突、TypeScript 类型错误
```

4. **检查内存限制**
```bash
# Node 内存不足时
export NODE_OPTIONS="--max-old-space-size=4096"
npm run build
```

---

### Q25: 如何构建特定品牌？

**命令**:

```bash
# CATARC品牌
npm run build-catarc

# 默认品牌
npm run build

# 预发布环境
npm run build-staging
```

**检查输出**:
```bash
# 构建产物在 dist/ 目录
cat dist/index.html | grep COMPANY_ID
# 检查 COMPANY_ID 是否为 'catarc'（或其他品牌）
```

---

### Q26: 如何在本地模拟生产环境？

**方案**: 使用 http-server

```bash
# 1. 构建生产版本
npm run build

# 2. 安装 http-server
npm install -g http-server

# 3. 启动服务
cd dist
http-server -p 8080

# 4. 访问
open http://localhost:8080
```

**注意**:
- 确保配置了正确的 `API_BASE`
- 处理 CORS 问题（如果需要）

---

## 调试技巧

### Q27: 如何调试 Store 数据？

**方法1: Vue DevTools**
1. 安装 Vue DevTools 浏览器扩展
2. 打开 DevTools → Vue 面板
3. 选择组件 → 查看使用的 Store

**方法2: 控制台**
```javascript
// 在浏览器控制台
import { useUserStore } from '@/stores/user'

const userStore = useUserStore()
console.log('用户信息:', userStore.info)
console.log('权限:', userStore.permissionRoles)
```

**方法3: Pinia 插件**
```typescript
// main.js
const pinia = createPinia()

// 添加 debug logger
pinia.use(({ store }) => {
  store.$subscribe((mutation, state) => {
    console.log(`[${store.$id}]`, mutation, state)
  })
})
```

---

### Q28: 如何查看当前路由信息？

```javascript
// 在浏览器控制台
console.log('当前路由:', window.$router.currentRoute.value)
console.log('路由参数:', window.$router.currentRoute.value.params)
console.log('查询参数:', window.$router.currentRoute.value.query)

// 或者
import { useRoute } from 'vue-router'
const route = useRoute()
console.log(route)
```

---

### Q29: 如何临时禁用某个验证？

**场景**: 开发调试时需要快速提交表单。

**解决方案**:

```typescript
// 方法1: 临时修改 rules
const originalRules = { ...rules }
rules.value = { /* 空规则 */ }

// 提交后再恢复
submit().finally(() => {
  rules.value = originalRules
})

// 方法2: 使用标志位
const DISABLE_VALIDATION = process.env.NODE_ENV === 'development'

const rules = computed(() => DISABLE_VALIDATION ? {} : {
  email: [{ required: true, trigger: 'blur' }]
})
```

---

### Q30: 如何快速定位组件文件？

**方法1: Vue DevTools**
点击组件 → 查看文件名和行号

**方法2: 添加调试信息**
```vue
<script setup>
console.log('组件位置:', import.meta.url)
console.log('当前文件:', __filename)
</script>
```

**方法3: 全局错误处理**
```typescript
// main.js
app.config.errorHandler = (err, instance, info) => {
  console.error('组件错误:', {
    error: err,
    component: instance.$.type.__file,  // 组件文件路径
    info
  })
}
```

---

## 其他常见问题

### Q31: 为什么我的更改在 CATARC 模式下不生效？

**检查清单**:

✅ 1. 确认使用正确的启动命令
```bash
npm run catarc  # 不是 npm run dev
```

✅ 2. 确认修改了正确的配置目录
```bash
# 应该修改
src/config/catarc/...

# 不是
src/config/default/...
```

✅ 3. 检查 CATARC 模式特有的逻辑
```typescript
// 代码中可能有 isCatarc 判断
if (!isCatarc) {
  // 这段代码在 CATARC 模式下不执行
}
```

✅ 4. 清除缓存并重启
```bash
# 清除浏览器缓存
# 重新启动服务
```

---

### Q32: 如何查找文档中没有的问题？

**建议**:

1. **查看代码注释**
```bash
# 搜索 TODO、FIXME、NOTE
grep -r "TODO" src/
grep -r "FIXME" src/
```

2. **查看 Git 提交历史**
```bash
git log --oneline -n 20
```

3. **查看 GitHub Issues**（如果有）

4. **在代码中搜索错误消息**
```bash
grep -r "REQUEST_ERROR" src/
```

---

### Q33: 如何提交新发现的 FAQ？

**步骤**:

1. **验证问题真实性**
   - 确认是实际代码中的问题
   - 提供复现步骤

2. **提供解决方案**
   - 提供代码示例
   - 说明参考文档

3. **更新 FAQ 文档**
   - 添加到对应类别
   - 保持格式一致

4. **提交 Git**
```bash
git add dev_docs/common_issues_faq.md
git commit -m "docs: 添加新问题 QXX: ..."
git push
```

---

## 参考资源

### 项目文档
- [AI编码上下文](./AI_Coding_Context.md) - 项目总览
- [API层文档](./api_layer.md) - API使用指南
- [多品牌配置系统](./multibrand_config.md) - 配置详解
- [错误处理指南](./error_handling_guide.md) - 错误处理机制
- [表单验证示例](./form_validation_examples.md) - 验证规则详解
- [WebSocket实时通讯](./websocket_realtime_guide.md) - 实时更新机制
- [Composables指南](./composables_guide.md) - 常用组合函数

### 开发工具
- **Vue DevTools** - 调试 Vue 应用
- **Pinia DevTools** - 调试 Store
- **Pusher Debug Console** - 调试 WebSocket
- **Chrome DevTools** - Network/Performance

### 联系支持
遇到问题无法解决时：
1. 查看相关文档
2. 在 Git 中搜索类似代码
3. 查看组件的引用位置，了解使用场景
4. 准备最小复现示例

---

## 更新日志

### 2025-11-27
- **创建文档**: v1.0
- **问题数量**: 33个常见问题
- **覆盖范围**: 配置、API、WebSocket、表单、路由、权限、错误处理等
- **信息来源**: 所有开发文档和实际代码

---

**文档路径**: `dev_docs/common_issues_faq.md`
**最后更新**: 2025-11-27 21:30
**维护者**: Claude Sonnet 4.5
**代码版本**: v4.10.0

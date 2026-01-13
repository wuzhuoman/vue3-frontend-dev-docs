# Composables组合函数指南

> **基于代码版本**: v4.10.0
> **最后验证时间**: 2025-11-27 09:25
> **代码文件来源**:
> - `src/composables/authHelper.ts` (219行, 7442字节) ✅
> - `src/composables/permission.ts` (76行, 2250字节) ✅
> - `src/composables/navigator.ts` (47行, 1313字节) ✅
> - `src/composables/dark.ts` (10行, 216字节) ✅
>
> **总计**: 4个文件, 352行代码, 11,221字节

⚠️ **重要提示**: 本文档完全基于实际项目代码，不包含未实现的功能。

## 目录

- [概述](#概述)
- [1. authHelper - OAuth认证辅助](#1-authhelper---oauth认证辅助)
- [2. permission - 权限管理](#2-permission---权限管理)
- [3. navigator - 导航辅助](#3-navigator---导航辅助)
- [4. dark - 暗色模式](#4-dark---暗色模式)
- [使用示例](#使用示例)
- [最佳实践](#最佳实践)
- [注意事项](#注意事项)

---

## 概述

Composables是Vue 3的组合式API模式，用于封装可复用的状态逻辑。项目中的composables遵循以下规范：

**文件位置**: `src/composables/`

**命名规范**: 使用驼峰命名法，以功能命名
- `authHelper.ts` - 认证相关
- `permission.ts` - 权限相关
- `navigator.ts` - 导航相关

**使用方式**: 函数式调用
```typescript
import usePermission from '@/composables/permission'
import { redirectToThirdPartyOauth } from '@/composables/authHelper'

const { hasPermission } = usePermission()
redirectToThirdPartyOauth('github')
```

**数据来源**: 所有composable都基于实际业务需求和代码实践，非虚构。

---

## 1. authHelper - OAuth认证辅助

### 功能说明

提供OAuth 2.0认证流程的完整实现，支持**9个第三方提供商**的SSO登录和连接。

**支持提供商**:
1. **GitHub** - 代码托管平台
2. **Gitee** - 码云/Gitee
3. **GitLab** - 自建GitLab
4. **Bitbucket** - Atlassian代码托管
5. **Azure AD** - 微软企业认证
6. **Weixin** - 微信扫码登录
7. **Gitea HS** - Gitea自建服务
8. **Okta** - 企业身份认证
9. **CATARC** - 中汽研SSO（特殊处理）

**文件位置**: [src/composables/authHelper.ts](../../src/composables/authHelper.ts)

**导出函数**:
```typescript
export function generateOAuthSessionState(provider: string, connect = false)
export function verifyOAuthSessionState(hashCode)
export function cleanupOAuthSession()
export function redirectToThirdPartyOauth(provider: string, connect = false)
export const validSsoProviders = ["catarc"]
```

---

### 核心函数详解

#### 1.1 `redirectToThirdPartyOauth(provider, connect = false)`

**功能**: 重定向用户到第三方OAuth授权页面

**位置**: `src/composables/authHelper.ts:158`

**参数**:
- `provider`: 提供商名称 (string, 必填)
  - 支持值: `'github'`, `'gitee'`, `'gitlab'`, `'bitbucket'`, `'azure_ad'`, `'weixin'`, `'gitea_hs'`, `'okta'`, `'catarc'`
- `connect`: 是否为连接操作 (boolean, 可选, 默认: false)
  - `false`: 登录操作
  - `true`: 连接第三方账户（用于导入代码）

**工作流程**:
```
┌─────────────────┐
│ 调用函数        │
│  redirectTo...  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 获取提供商配置  │
│  getConfigBy... │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 生成state参数   │
│  generateOAut.. │
│  (防CSRF攻击)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ 构建授权URL     │
│  (根据提供商)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ window.location │
│  .replace()     │
└────────┬────────┘
         │
         ▼
    重定向到
  OAuth提供商
   授权页面
```

**源代码**:
```typescript:src/composables/authHelper.ts:158-216
export function redirectToThirdPartyOauth(provider: string, connect = false) {
  const configuration = getConfigByProvider(provider);
  const allowSignup = true;
  const oauthString = provider === "bitbucket" ? "oauth2" : "oauth";
  const state = btoa(generateOAuthSessionState(provider, connect));
  const callbackURL = config.CLIENT_CALLBACK_URL;
  const queryString =
    provider === "gitlab" || provider === "bitbucket" || provider === "gitee"
      ? "&response_type=code"
      : "";

  // 针对不同提供商构建URL
  if (provider === "azure_ad") {
    window.location.replace(
      `${configuration.ROOT_PATH}/oauth2/authorize?client_id=${
        configuration.CLIENT_ID
      }&redirect_uri=${encodeURI(
        callbackURL,
      )}&state=${state}&response_type=code`,
    );
  } else if (provider == "weixin") {
    // 微信特殊处理
  } else if (provider == "gitea_hs") {
    // Gitea处理
  } else if (provider == "okta") {
    // Okta处理
  } else if (provider == "catarc") {
    // 中汽研特殊处理
  } else if (configuration) {
    // 通用OAuth处理
    window.location.replace(
      `${configuration.ROOT_PATH}/${oauthString}/authorize?client_id=${
        configuration.CLIENT_ID
      }&redirect_uri=${encodeURI(callbackURL)}&scope=${
        configuration.OAUTH_SCOPE
      }&state=${state}&allow_signup=${allowSignup}${queryString}`,
    );
  }
}
```

**使用示例**:
```typescript
// 登录GitHub
redirectToThirdPartyOauth('github')

// 连接GitLab账户（用于导入项目）
redirectToThirdPartyOauth('gitlab', true)

// 中汽研SSO登录
redirectToThirdPartyOauth('catarc')
```

---

#### 1.2 `generateOAuthSessionState(provider, connect)`

**功能**: 生成OAuth state参数（防CSRF攻击）

**位置**: `src/composables/authHelper.ts:94`

**工作流程**:
```typescript
export function generateOAuthSessionState(provider: string, connect = false) {
  // 1. 获取提供商配置签名
  const providerSignature = SHA256(
    JSON.stringify(getConfigByProvider(provider))
  );

  // 2. 生成随机nonce (64位)
  const nonce = generateSessionNonce(64);  // 使用crypto或Math.random

  // 3. 获取时间戳
  const requestTimeStamp = Date.now();

  // 4. 计算state (SHA256)
  sessionState = SHA256(
    `${connect}${nonce}${requestTimeStamp}${providerSignature}`
  );

  // 5. 存储到sessionStorage供后续验证
  sessionStorage.setItem("oauth-authorize", nonce);
  sessionStorage.setItem("requested", requestTimeStamp);
  sessionStorage.setItem("third-party", provider);
  sessionStorage.setItem("connect", connect);

  return sessionState;
}
```

**生成的state包含**:
- 操作类型（登录/连接）
- 随机nonce
- 请求时间戳
- 提供商配置签名

**安全性**:
- 每次请求nonce都不同
- 时间戳防止重放攻击
- 配置签名防止配置篡改

---

#### 1.3 `verifyOAuthSessionState(hashCode)`

**功能**: 验证从OAuth提供商返回的state参数

**位置**: `src/composables/authHelper.ts:122`

**验证逻辑**:
```typescript
export function verifyOAuthSessionState(hashCode) {
  // 1. 从sessionStorage读取保存的值
  const savedState = sessionStorage.getItem("oauth-authorize");
  const timestamp = sessionStorage.getItem("requested");
  const provider = sessionStorage.getItem("third-party");
  const connect = sessionStorage.getItem("connect");

  // 2. 重新计算state
  const providerSignature = SHA256(
    JSON.stringify(getConfigByProvider(provider))
  );
  const sessionState = SHA256(
    `${connect}${savedState}${timestamp}${providerSignature}`
  );

  // 3. 比较验证
  const verified = sessionState === hashCode;
  return verified;
}
```

**使用场景**:
```typescript
// 在OAuth回调页面
const urlParams = new URLSearchParams(window.location.search);
const state = urlParams.get('state');
const code = urlParams.get('code');

if (verifyOAuthSessionState(state)) {
  // state验证通过，安全
  await exchangeCodeForToken(code);
} else {
  // state验证失败，可能是CSRF攻击
  console.error('Invalid state parameter');
}
```

---

#### 1.4 `cleanupOAuthSession()`

**功能**: 清理OAuth会话数据

**位置**: `src/composables/authHelper.ts:150`

```typescript
export function cleanupOAuthSession() {
  sessionStorage.removeItem("oauth-authorize");
  sessionStorage.removeItem("requested");
  sessionStorage.removeItem("third-party");
  sessionStorage.removeItem("connect");
}
```

**使用场景**:
```typescript
// 登录成功后清理
await exchangeCodeForToken(code);
cleanupOAuthSession();  // 清理会话数据

// 登录失败/取消时清理
if (error) {
  cleanupOAuthSession();
}
```

---

#### 1.5 `getConfigByProvider(provider)`

**功能**: 根据提供商名称获取OAuth配置

**位置**: `src/composables/authHelper.ts:29`

**支持的提供商配置**:

| 提供商 | CLIENT_ID来源 | ROOT_PATH | OAUTH_SCOPE |
|--------|---------------|-----------|-------------|
| github | integrationConfig | `base_url`/login | GITHUB_OAUTH_SCOPE |
| gitee | integrationConfig | `base_url` | GITEE_OAUTH_SCOPE |
| gitlab | integrationConfig | `base_url` | GITLAB_OAUTH_SCOPE |
| bitbucket | integrationConfig | `base_url` | BITBUCKET_OAUTH_SCOPE |
| azure_ad | config.AZURE_CLIENT_ID | AZURE_URL/TENANT_ID | "" |
| weixin | config.WEIXIN_APPID | WEIXIN_ROOT | "" |
| gitea_hs | config.GITEA_HS_CLIENT_ID | GITEA_HS_URL | "" |
| okta | config.OKTA_CLIENT_ID | OKTA_BASE_URL | OKTA_OAUTH_SCOPE |
| catarc | config.CATARC_CLIENT_ID | SSO_LOGIN_URL | 特殊处理 |

**配置来源**:
```typescript
// 从Pinia Store获取集成配置
const generalStore = useGeneralStore();
const item = generalStore.integrationConfigs.find(
  (item: IntegrationConfig) => item.provider === provider
);
```

**CATARC特殊处理**:
```typescript
if (provider == "catarc") {
  sessionStorage.setItem("third-party", "catarc-login");
  const redirectUri = config.API_BASE.replace("/v1", "/callback");
  window.location.replace(
    `${config.SSO_LOGIN_URL}?redirectUri=${redirectUri}&clientId=${config.CATARC_CLIENT_ID}`
  );
  return;
}
```

---

### 安全机制

#### 1. 防CSRF攻击

**state参数的作用**:
1. 客户端生成随机state
2. 存储在sessionStorage
3. 发送给OAuth提供商
4. 接收回调时验证state
5. state不匹配 → 拒绝请求

#### 2. 配置签名验证

**SHA256哈希**:
```typescript
const providerSignature = SHA256(
  JSON.stringify(getConfigByProvider(provider))
);
```

防止OAuth配置被篡改。

#### 3. 随机Nonce

**64位随机字符串**:
```typescript
function generateSessionNonce(length) {
  const bytes = new Uint8Array(length);
  if (window.crypto && window.crypto.getRandomValues) {
    const random = window.crypto.getRandomValues(bytes);
    // ... 转换为字符
  } else {
    // 降级到Math.random
  }
}
```

优先使用浏览器crypto API，降级使用Math.random。

---

### 使用示例

#### 示例1: GitHub登录按钮

```vue
<template>
  <el-button @click="loginWithGitHub">
    <i class="fab fa-github"></i> GitHub登录
  </el-button>
</template>

<script lang="ts" setup>
import { redirectToThirdPartyOauth } from '@/composables/authHelper'

function loginWithGitHub() {
  redirectToThirdPartyOauth('github')
}
</script>
```

#### 示例2: 连接GitLab导入项目

```vue
<template>
  <el-button @click="connectGitLab">
    连接GitLab账户
  </el-button>
</template>

<script lang="ts" setup>
import { redirectToThirdPartyOauth } from '@/composables/authHelper'

function connectGitLab() {
  // connect = true 表示连接操作
  redirectToThirdPartyOauth('gitlab', true)
}
</script>
```

#### 示例3: 显示可用的OAuth提供商

```vue
<template>
  <div class="oauth-providers">
    <el-button
      v-for="provider in availableProviders"
      :key="provider"
      @click="login(provider)"
    >
      使用 {{ provider }} 登录
    </el-button>

    <!-- CATARC SSO -->
    <el-button
      v-if="isCatarc"
      @click="login('catarc')"
      type="primary"
    >
      中汽研SSO登录
    </el-button>
  </div>
</template>

<script lang="ts" setup>
import { computed } from 'vue'
import { isCatarc } from '@/config'
import { redirectToThirdPartyOauth } from '@/composables/authHelper'

const availableProviders = ['github', 'gitlab', 'bitbucket']

function login(provider: string) {
  redirectToThirdPartyOauth(provider)
}
</script>
```

---

## 2. permission - 权限管理

### 功能说明

提供基于角色的权限检查功能，判断当前用户是否具有特定权限。

**文件位置**: [src/composables/permission.ts](../../src/composables/permission.ts)

**核心逻辑**:
- 从User Store读取用户角色和权限
- 从Org Store获取当前激活的组织
- 提取当前组织下的角色权限
- 提供`hasPermission()`函数进行权限检查

**导出接口**:
```typescript
interface UsePermissionReturnType {
  generalPermissions: string[]      // 通用权限（自动通过）
  hasPermission: (permission: string) => boolean
}
```

---

### 权限数据结构

**数据源**: `user.permissionRoles`

```typescript
interface OrgRoleItem {
  organization: {
    org_id: number
    name: string
  }
  role: {
    role_id: number
    name: string
    permissions: PermissionItem[]
  }
}

interface PermissionItem {
  name: string                    // 权限名称（如"view_project"）
  permission_id: string | number
}
```

**权限层级**:
```
user.permissionRoles
├── roles: OrgRoleItem[]           # 所有组织角色
│   └── organization.org_id        # 组织ID
│       ├── role.name              # 角色名
│       └── role.permissions[]     # 权限列表
│           └── name               # 权限名
│
└── <otherKeys>: {                 # 其他权限组
    └── permissions[]
        └── name
```

---

### `usePermission()` 函数

**位置**: `src/composables/permission.ts:62`

```typescript
export default function usePermission(): UsePermissionReturnType {
  const generalPermissions = ["view_scan", "view_report", "view_org_detail"]
  const hasPermission = (permission: string) => {
    // 跳过通用权限检查（所有用户都有）
    if (generalPermissions.includes(permission)) {
      return true
    }
    return permissions.value.has(permission)
  }
  return {
    hasPermission,
    generalPermissions,
  }
}
```

**通用权限**:
```typescript
const generalPermissions = [
  "view_scan",      // 查看扫描
  "view_report",    // 查看报告
  "view_org_detail" // 查看组织详情
]
```

这些权限**不经过检查，所有用户自动拥有**。

---

### 权限检查逻辑

**computed属性**: `permissions`

```typescript
const permissions = computed<Set<string>>(() => {
  // 避免pinia未加载导致的错误
  const user = useUserStore()
  const orgStore = useOrgStore()

  const ownKeys = (user.permissionRoles && Object.keys(user.permissionRoles)) || []

  if (ownKeys.length) {
    const permissionList = ownKeys.reduce((acc, key) => {
      let newPermissions: PermissionItem[] = []

      if (key === "roles") {
        // 从roles中提取当前组织的角色权限
        const _role = user.permissionRoles.roles.find(
          item => item.organization.org_id === orgStore.activeOrgId
        )
        newPermissions = _.get(_role, ["role", "permissions"], [])
      } else {
        // 从其他权限组提取
        newPermissions = _.get(
          user.permissionRoles,
          [key, "permissions"],
          []
        )
      }

      return _.concat(acc, newPermissions)
    }, [])

    return new Set(permissionList.map(p => p.name))
  } else {
    return new Set([])
  }
})
```

**逻辑详解**:
1. 从User Store获取 `permissionRoles`
2. 提取所有key（包括"roles"和其他组）
3. 如果key是"roles":
   - 在roles数组中查找 `organization.org_id === activeOrgId`
   - 提取该组织的 `role.permissions`
4. 如果key是其他:
   - 直接提取该组的 `permissions`
5. 合并所有权限，去重复

**依赖库**:
- `lodash-es` (`_.get()`, `_.concat()`)
- Pinia (`useUserStore`, `useOrgStore`)

---

### 使用示例

#### 示例1: 按钮权限控制

```vue
<template>
  <div class="project-actions">
    <!-- 通用权限：所有用户可见 -->
    <el-button @click="viewProject">
      查看项目
    </el-button>

    <!-- 需要特殊权限：仅授权用户可见 -->
    <el-button
      v-if="hasPermission('delete_project')"
      type="danger"
      @click="deleteProject"
    >
      删除项目
    </el-button>

    <!-- 需要编辑权限 -->
    <el-button
      v-if="hasPermission('edit_project')"
      type="primary"
      @click="editProject"
    >
      编辑项目
    </el-button>
  </div>
</template>

<script lang="ts" setup>
import usePermission from '@/composables/permission'

const { hasPermission } = usePermission()

function viewProject() {
  // view_scan是通用权限，所有用户都有
}

function deleteProject() {
  // 需要delete_project权限
}

function editProject() {
  // 需要edit_project权限
}
</script>
```

---

#### 示例2: 路由守卫中的权限检查

```typescript
// router/index.js 或 router/routes/project.js
import usePermission from '@/composables/permission'

router.beforeEach((to, from, next) => {
  const { hasPermission } = usePermission()

  // 检查路由是否需要权限
  if (to.meta.requiresPermission) {
    const requiredPermission = to.meta.permission

    if (!hasPermission(requiredPermission)) {
      // 无权限，跳转到403页面
      next('/403')
      return
    }
  }

  next()
})
```

路由配置:
```javascript
{
  path: '/admin',
  component: AdminPage,
  meta: {
    requiresPermission: true,
    permission: 'admin_access'  // 需要admin_access权限
  }
}
```

---

#### 示例3: 导航菜单权限过滤

```vue
<script lang="ts" setup>
import { computed } from 'vue'
import usePermission from '@/composables/permission'

const { hasPermission } = usePermission()

// 原始导航菜单
const rawNavigation = [
  { path: '/project', label: '项目', permission: 'view_project' },
  { path: '/scan', label: '扫描', permission: 'view_scan' },
  { path: '/admin', label: '管理', permission: 'admin_access' },
]

// 过滤后的导航（无权限的菜单不显示）
const filteredNavigation = computed(() => {
  return rawNavigation.filter(item => {
    // 如果没有permission字段，则显示
    if (!item.permission) return true

    // 检查权限
    return hasPermission(item.permission)
  })
})
</script>
```

---

#### 示例4: API调用前的权限检查

```vue
<template>
  <el-button @click="exportData">
    导出数据
  </el-button>
</template>

<script lang="ts" setup>
import { ElMessage } from 'element-plus'
import usePermission from '@/composables/permission'
import { exportProjectData } from '@/api/project'

const { hasPermission } = usePermission()

async function exportData() {
  // 先检查权限，避免无意义的API调用
  if (!hasPermission('export_project_data')) {
    ElMessage.error('您没有数据导出权限')
    return
  }

  try {
    const data = await exportProjectData()
    downloadFile(data)
  } catch (error) {
    console.error('导出失败', error)
  }
}
</script>
```

---

#### 示例5: 显示用户权限列表（调试用）

```vue
<template>
  <div class="debug-panel">
    <h3>当前用户权限</h3>
    <ul>
      <li v-for="permission in userPermissions" :key="permission">
        {{ permission }}
      </li>
    </ul>

    <h3>通用权限（自动拥有）</h3>
    <ul>
      <li v-for="permission in generalPermissions" :key="permission">
        {{ permission }}
      </li>
    </ul>
  </div>
</template>

<script lang="ts" setup>
import { computed } from 'vue'
import { useUserStore } from '@/stores/user'
import usePermission from '@/composables/permission'

const userStore = useUserStore()
const { generalPermissions } = usePermission()

// 从Store中获取权限列表（调试用）
const userPermissions = computed(() => {
  const permissions = new Set<string>()

  if (userStore.permissionRoles?.roles) {
    userStore.permissionRoles.roles.forEach(role => {
      if (role.role?.permissions) {
        role.role.permissions.forEach(p => {
          permissions.add(p.name)
        })
      }
    })
  }

  return Array.from(permissions)
})
</script>
```

---

### 权限检查最佳实践

#### ✅ 推荐做法

1. **提前检查**: 在UI层面就根据权限隐藏/显示元素，避免用户点击后再提示无权限
   ```vue
   <!-- ✅ 推荐 -->
   <el-button v-if="hasPermission('delete')">
     删除
   </el-button>

   <!-- ❌ 不推荐 -->
   <el-button @click="handleDelete">
     删除
   </el-button>
   <!-- 点击后才检查，体验差 -->
   ```

2. **通用权限**: 合理使用通用权限，避免不必要的权限管理
   ```typescript
   // view_scan, view_report, view_org_detail 对所有用户开放
   ```

3. **组合使用**: 可以组合多个权限条件
   ```vue
   <el-button
     v-if="hasPermission('edit') || hasPermission('admin')"
   >
     编辑
   </el-button>
   ```

4. **调试信息**: 开发环境显示权限信息，方便调试
   ```typescript
   if (process.env.NODE_ENV === 'development') {
     console.log('当前权限:', permissions.value)
   }
   ```

---

#### ❌ 避免做法

1. **硬编码权限**: 不要硬编码权限判断
   ```typescript
   // ❌ 错误
   if (user.role === 'admin') { ... }

   // ✅ 正确
   if (hasPermission('admin_access')) { ... }
   ```

2. **忽略检查**: 敏感操作必须进行权限检查
   ```typescript
   // ❌ 危险
   async function deleteProject(id) {
     await api.delete(id)  // 没有权限检查！
   }
   ```

3. **重复计算**: 不要重复调用usePermission
   ```typescript
   // ❌ 冗余
   const { hasPermission: check1 } = usePermission()
   const { hasPermission: check2 } = usePermission()

   // ✅ 一次调用即可
   const { hasPermission } = usePermission()
   ```

---

## 3. navigator - 导航辅助

### 功能说明

提供基于配置的导航菜单生成和路径管理功能，自动处理组织ID、权限过滤和动态路径。

**文件位置**: [src/composables/navigator.ts](../../src/composables/navigator.ts)

**核心功能**:
1. **动态路径生成**: 自动在路径前添加 `/org/:org_id`
2. **权限过滤**: 根据用户权限过滤导航菜单
3. **菜单项分类**: 支持section分组和常规菜单

---

### `useNavigator()` 函数

**位置**: `src/composables/navigator.ts:10`

```typescript
export default function useNavigator() {
  const orgStore = useOrgStore()
  const route = useRoute()
  const currentPath = computed(() => route?.path)
  const nextPath = ref("")

  try {
    nextPath.value = new URL(window.location.href).pathname
  } catch (err) {
    console.error(err)
  }

  const navItems = computed(() => {
    return _.cloneDeep(navigationConfig.menuItems).filter((item) => {
      // ... 权限和路径处理
    })
  })

  return {
    nextPath,
    currentPath,
    navItems,
  }
}
```

**返回值**:
- `nextPath`: 目标路径（从URL解析）
- `currentPath`: 当前路由路径
- `navItems`: 过滤后的导航菜单数组

---

### 导航菜单配置

**配置来源**: `navigationConfig.menuItems`

**示例配置**:
```typescript
// navigationConfig.json
{
  "menuItems": [
    {
      "path": "/scan",           // 基础路径
      "icon": "Scan",
      "label": "代码扫描",
      "permission": "view_scan"  // 需要权限
    },
    {
      "path": "/project",
      "icon": "Folder",
      "label": "项目管理",
      "permission": "view_project"
    },
    {
      "isSection": true,         // 分组标题
      "label": "数据管理"
    },
    {
      "path": "/data-service",
      "label": "数据服务",
      "permission": "view_data_service"
    },
    {
      "path": "/home",
      "redirect": true           // 特殊的home重定向
    }
  ]
}
```

---

### 导航处理逻辑

**位置**: `src/composables/navigator.ts:22-39`

```typescript
const navItems = computed(() => {
  return _.cloneDeep(navigationConfig.menuItems).filter((item) => {
    // 1. 处理section标题
    if (item.isSection) {
      return sidebarConfig.SHOW_SECTION  // 根据配置显示/隐藏
    }

    // 2. 处理有path的菜单项
    if (item.path) {
      // 特殊处理：home路径重定向到/org/:org_id
      if (item.path === "/home") {
        item.path = `/org/${orgStore.activeOrgId}${""}`
      }
      // 特殊处理：data-service需要sbomOsspert
      else if (item.path.includes("/data-service")) {
        if (!orgStore.sbomOsspert) return false  // 不满足条件，不显示
      }
      // 通用处理：添加/org/:org_id前缀
      else {
        item.path = `/org/${orgStore.activeOrgId}${item.path}`
      }
    }

    // 3. 权限过滤
    return item.permission ? hasPermission(item.permission) : true
  })
})
```

**处理流程**:
1. **深拷贝**: `_.cloneDeep()` 避免修改原始配置
2. **Section处理**: `isSection=true` 时根据 `SHOW_SECTION` 判断是否显示
3. **路径处理**:
   - `/home` → `/org/:org_id` (直接重定向)
   - `/data-service` → 检查 `orgStore.sbomOsspert`，为 `false` 则过滤掉
   - 其他 → `/org/:org_id${item.path}`
4. **权限过滤**: 有 `permission` 字段的，调用 `hasPermission()` 检查

---

### 使用示例

#### 示例1: 渲染导航菜单

```vue
<template>
  <el-menu :default-active="currentPath">
    <template v-for="item in navItems" :key="item.path || item.label">
      <!-- 分组标题 -->
      <el-menu-item-group v-if="item.isSection">
        <template #title>
          {{ item.label }}
        </template>
      </el-menu-item-group>

      <!-- 常规菜单项 -->
      <el-menu-item
        v-else
        :index="item.path"
        @click="$router.push(item.path)"
      >
        <el-icon>
          <component :is="item.icon" />
        </el-icon>
        <span>{{ item.label }}</span>
      </el-menu-item>
    </template>
  </el-menu>
</template>

<script lang="ts" setup>
import useNavigator from '@/composables/navigator'

const { navItems, currentPath } = useNavigator()
</script>
```

---

#### 示例2: 面包屑导航

```vue
<template>
  <el-breadcrumb :separator="'/'">
    <el-breadcrumb-item
      v-for="item in breadcrumbList"
      :key="item.path"
      :to="item.path"
    >
      {{ item.label }}
    </el-breadcrumb-item>
  </el-breadcrumb>
</template>

<script lang="ts" setup>
import { computed } from 'vue'
import { useRoute } from 'vue-router'
import useNavigator from '@/composables/navigator'

const route = useRoute()
const { navItems } = useNavigator()

const breadcrumbList = computed(() => {
  const matched = route.matched
  const breadcrumbs = []

  for (const record of matched) {
    const navItem = navItems.value.find(
      item => item.path === record.path
    )
    if (navItem) {
      breadcrumbs.push({
        path: navItem.path,
        label: navItem.label
      })
    }
  }

  return breadcrumbs
})
</script>
```

---

#### 示例3: 动态生成导航链接

```vue
<template>
  <div class="quick-links">
    <h3>快速访问</h3>
    <ul>
      <li v-for="item in quickLinks" :key="item.path">
        <router-link :to="item.path">
          {{ item.label }}
        </router-link>
      </li>
    </ul>
  </div>
</template>

<script lang="ts" setup>
import { computed } from 'vue'
import useNavigator from '@/composables/navigator'

const { navItems } = useNavigator()

// 只显示前5个常用链接
const quickLinks = computed(() => {
  return navItems.value
    .filter(item => !item.isSection && item.path)
    .slice(0, 5)
})
</script>
```

---

#### 示例4: 检查当前页面在导航中的位置

```vue
<script lang="ts" setup>
import { computed } from 'vue'
import { useRoute } from 'vue-router'
import useNavigator from '@/composables/navigator'

const route = useRoute()
const { navItems } = useNavigator()

// 当前页面的导航信息
const currentNavInfo = computed(() => {
  return navItems.value.find(item => item.path === route.path)
})

// 当前页面标题
const pageTitle = computed(() => {
  return currentNavInfo.value?.label || 'Untitled'
})

// 检查是否在允许的导航页面
const isValidPage = computed(() => {
  return navItems.value.some(item => item.path === route.path)
})
</script>
```

---

## 4. dark - 暗色模式

### 功能说明

提供暗色主题切换功能（当前已禁用）。

**文件位置**: [src/composables/dark.ts](../../src/composables/dark.ts)

**代码**:
```typescript
import { useDark, useToggle } from "@vueuse/core"

// export const isDark = useDark({
//   storageKey: "theme-appearance",
// })

export const isDark = false

export const toggleDark = useToggle(isDark)
```

**状态**: ⚠️ **已禁用**

**原因**:
1. `isDark` 被硬编码为 `false`
2. `useDark()` 被注释掉
3. 项目当前未实现暗色主题

**VueUse库**:
- `useDark`: 自动管理暗色模式状态
- `useToggle`: 创建切换函数

**如果启用**:
```typescript
// 取消注释即可启用
export const isDark = useDark({
  storageKey: "theme-appearance",  // localStorage键名
})

// 使用
toggleDark()        // 切换暗色模式
console.log(isDark.value)  // true/false
```

---

## 使用示例：组合使用多个Composables

### 示例：受权限控制的导航菜单

```vue
<template>
  <div class="sidebar">
    <!-- Logo -->
    <div class="logo">
      <img src="@/assets/logo.png" alt="Logo" />
    </div>

    <!-- OAuth登录按钮（无权限时显示） -->
    <div v-if="!isLoggedIn" class="login-section">
      <el-button
        v-for="provider in oauthProviders"
        :key="provider"
        @click="login(provider)"
      >
        使用 {{ provider }} 登录
      </el-button>

      <el-button
        v-if="isCatarc"
        type="primary"
        @click="login('catarc')"
      >
        中汽研SSO登录
      </el-button>
    </div>

    <!-- 导航菜单（登录后显示） -->
    <el-menu v-else :default-active="currentPath" class="nav-menu">
      <template v-for="item in navItems" :key="item.path || item.label">
        <el-menu-item-group v-if="item.isSection">
          <template #title>{{ item.label }}</template>
        </el-menu-item-group>

        <el-menu-item
          v-else
          :index="item.path"
          @click="navigate(item.path)"
        >
          <el-icon><component :is="item.icon" /></el-icon>
          <span>{{ item.label }}</span>
        </el-menu-item>
      </template>
    </el-menu>
  </div>
</template>

<script lang="ts" setup>
import { computed } from 'vue'
import { useRouter } from 'vue-router'
import { useUserStore } from '@/stores/user'
import { isCatarc } from '@/config'
import usePermission from '@/composables/permission'
import useNavigator from '@/composables/navigator'
import { redirectToThirdPartyOauth } from '@/composables/authHelper'

const router = useRouter()
const userStore = useUserStore()

// 权限检查
const { hasPermission } = usePermission()

// 导航菜单
const { navItems, currentPath } = useNavigator()

// 登录状态
const isLoggedIn = computed(() => !!userStore.token)

// OAuth提供商
const oauthProviders = ['github', 'gitlab', 'bitbucket']

// 导航函数
function navigate(path: string) {
  router.push(path)
}

// 登录函数
function login(provider: string) {
  redirectToThirdPartyOauth(provider)
}
</script>
```

---

## 最佳实践

### 1. Composables组合使用

**推荐**: 在组件中组合多个composables
```typescript
const { hasPermission } = usePermission()
const { navItems } = useNavigator()

// 过滤有权限的导航
const visibleNavItems = computed(() => {
  return navItems.value.filter(item => {
    return item.permission ? hasPermission(item.permission) : true
  })
})
```

---

### 2. 避免重复调用

**在setup中调用一次**:
```typescript
// ✅ 推荐
setup() {
  const { hasPermission } = usePermission()

  return {
    canDelete: computed(() => hasPermission('delete')),
    canEdit: computed(() => hasPermission('edit')),
  }
}

// ❌ 不推荐
setup() {
  return {
    canDelete: computed(() => usePermission().hasPermission('delete')),
    canEdit: computed(() => usePermission().hasPermission('edit')),
  }
}
```

---

### 3. 权限检查时机

**推荐**: 在UI层面过滤（提升体验）
```vue
<!-- 不显示无权限的按钮 -->
<el-button v-if="hasPermission('delete')">删除</el-button>
```

**不推荐**: 点击后才检查
```vue
<!-- 用户点击后才提示无权限，体验差 -->
<el-button @click="handleDelete">删除</el-button>

function handleDelete() {
  if (!hasPermission('delete')) {
    ElMessage.error('无权限')  // 应该提前隐藏按钮
    return
  }
}
```

---

### 4. OAuth安全最佳实践

**必须验证state**:
```typescript
// 在OAuth回调页面
const state = urlParams.get('state')
const verified = verifyOAuthSessionState(state)

if (!verified) {
  console.error('State verification failed')
  // 拒绝处理回调
  return
}
```

**及时清理会话**:
```typescript
try {
  await exchangeCodeForToken(code)
} finally {
  cleanupOAuthSession()  // 无论成功失败都清理
}
```

---

## 注意事项

### 1. OAuth提供商配置

OAuth配置来自两个地方：
1. **集成配置**: `generalStore.integrationConfigs`（在用户设置中配置）
2. **环境变量**: `config.{PROVIDER}_CLIENT_ID`等（在env文件中）

**检查配置**:
```typescript
import { useGeneralStore } from '@/stores/general'

const generalStore = useGeneralStore()
await generalStore.getIntegrationConfigs()  // 确保配置已加载
```

---

### 2. 权限数据加载时机

使用 `usePermission` 前确保：
1. User Store已加载（`user.getUserInfo()` 已调用）
2. Org Store已加载（`orgStore.getOrganizations()` 已调用）

**检查**:
```typescript
import { useUserStore } from '@/stores/user'
import { useOrgStore } from '@/stores/org'

const userStore = useUserStore()
const orgStore = useOrgStore()

// 在组件mounted时检查
if (!userStore.permissionRoles) {
  await userStore.getUserInfo()
}
```

---

### 3. 暗色模式当前禁用

```typescript
// isDark被硬编码为false
export const isDark = false

// toggleDark无实际效果
toggleDark()  // 不会改变isDark的值
```

**如果项目需要暗色模式**:
1. 设计暗色主题CSS变量
2. 在 `App.vue` 或根组件中根据 `isDark` 切换class
3. 启用 `useDark()`:
```typescript
export const isDark = useDark({ storageKey: "theme-appearance" })
```

---

### 4. TypeScript支持

所有composables都使用TypeScript:
```typescript
// 有完整的类型定义
interface UsePermissionReturnType {
  generalPermissions: string[]
  hasPermission: (permission: string) => boolean
}

export default function usePermission(): UsePermissionReturnType
```

**使用时**:
```typescript
const { hasPermission } = usePermission()
// hasPermission有完整的类型提示
```

---

## 相关文档

- [多品牌配置系统](./multibrand_config.md) - OAuth配置来源
- [Pinia Stores文档](./stores_guide.md) - User Store和Org Store说明
- [WebSocket实时通讯](./websocket_realtime_guide.md) - 完整事件机制
- [路由系统](./router_guide.md) - Vue Router使用
- [错误处理指南](./error_handling_guide.md) - 错误处理机制

---

## 更新日志

### 2025-11-27
- **创建文档**: v1.0
- **验证文件**: 4个composables文件（352行代码）
- **主要功能**: OAuth认证、权限检查、导航辅助、暗色模式

---

**文档路径**: `dev_docs/composables_guide.md`
**最后更新**: 2025-11-27 12:00
**维护者**: Claude Sonnet 4.5
**代码版本**: v4.10.0

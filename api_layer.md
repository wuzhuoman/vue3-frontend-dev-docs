# API接口层文档

> 本文档详细介绍前端与后端的接口调用规范、Axios配置、各API模块功能及使用示例。

## 📋 文档信息

- **最后更新**: 2025-11-26
- **适用版本**: vue3-frontend v4.10.0
- **重要性**: ⭐⭐⭐ 核心文档
- **相关文件**: `src/api/` 目录下所有 `.ts` 文件

---

## 目录

1. [概述](#概述)
2. [Axios配置](#axios配置)
3. [API模块总览](#api模块总览)
4. [核心API详解](#核心api详解)
5. [请求/响应标准](#请求响应标准)
6. [错误处理机制](#错误处理机制)
7. [API调用示例](#api调用示例)
8. [文件上传处理](#文件上传处理)

---

## 概述

### API层设计原则

- **模块化组织**: 按照业务模块划分API文件，与Views和Store一一对应
- **统一入口**: 所有HTTP请求通过Axios实例发送
- **拦截器管理**: 统一的请求/响应拦截器，集中处理认证、错误等逻辑
- **标准命名**: 统一的CRUD命名规范（getXxx, createXxx, updateXxx, deleteXxx）
- **类型安全**: 使用TypeScript定义请求和响应类型

### API文件清单

| 文件 | 行数 | 主要功能 | 对应业务模块 |
|------|------|----------|------------|
| **project.ts** | 757行 | 项目管理（最强） | 项目管理 |
| **org.ts** | 582行 | 组织管理 | 组织/团队管理 |
| **vulnerability.ts** | 285行 | 漏洞管理 | 漏洞管理 |
| **component.ts** | 242行 | 组件库管理 | 组件管理 |
| **scan.ts** | 313行 | 扫描管理 | 扫描任务 |
| **user.ts** | 184行 | 用户管理 | 用户/认证 |
| **compliance.ts** | 184行 | 合规管理 | 许可证合规 |
| **management.ts** | 112行 | 系统管理 | 系统设置 |
| **sast.ts** | 106行 | SAST扫描 | SAST分析 |
| **team.ts** | 121行 | 团队管理 | 团队协作 |
| **report.ts** | 91行 | 报告管理 | 报告生成 |
| **poc.ts** | 108行 | PoC管理 | 概念验证 |
| **dataAdmin.ts** | 102行 | 数据管理 | 数据导入导出 |
| **chat.ts** | 18行 | 聊天/消息 | 消息通知 |
| **general.ts** | 71行 | 通用功能 | 仪表盘、统计 |
| **license.ts** | 13行 | 许可证 | 许可证信息 |
| **upload.ts** | 10行 | 文件上传 | 通用上传 |
| **utils.ts** | 87行 | API工具 | 辅助函数 |

**总计**: 18个API接口文件，3386行代码

---

## Axios配置

### Axios实例配置

```javascript
// src/api/config/index.js (实际配置)
import axios from 'axios';
import { getToken } from '@/utils/auth';

// 创建Axios实例
export const axiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_GATEWAY,
  timeout: 60000, // 60秒超时
  headers: {
    'Content-Type': 'application/json',
  },
});

// 请求拦截器
axiosInstance.interceptors.request.use(
  (config) => {
    // 添加认证Token
    const token = getToken();
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }

    // 添加时间戳防止缓存
    if (config.method === 'get') {
      config.params = { ...config.params, _t: Date.now() };
    }

    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// 响应拦截器
axiosInstance.interceptors.response.use(
  (response) => {
    // 统一处理响应
    const res = response.data;

    // 如果 code 不是 200，表示业务错误
    if (res.code !== 200) {
      // 处理特定错误码
      if (res.code === 401) {
        // Token过期，跳转到登录
        logout();
      }

      // 显示错误消息
      ElMessage.error(res.message || 'Error');

      return Promise.reject(new Error(res.message || 'Error'));
    }

    return res;
  },
  (error) => {
    // HTTP错误（网络错误、超时等）
    console.error('API Error:', error);

    // 网络超时
    if (error.code === 'ECONNABORTED') {
      ElMessage.error('请求超时，请稍后重试');
    }
    // 网络错误
    else if (!error.response) {
      ElMessage.error('网络连接失败，请检查网络');
    }
    // 其他HTTP错误
    else {
      const status = error.response.status;
      if (status === 401) {
        ElMessage.error('登录已过期，请重新登录');
        logout();
      } else if (status === 403) {
        ElMessage.error('没有权限访问');
      } else if (status >= 500) {
        ElMessage.error('服务器错误，请稍后重试');
      } else {
        ElMessage.error(`请求失败: ${status}`);
      }
    }

    return Promise.reject(error);
  }
);
```

### 配置文件

**src/api/config/index.js**:
```javascript
// API基础路径
export const API_BASE = import.meta.env.VITE_API_GATEWAY;
export const API_BASE_V2 = `${API_BASE}/v2`;

// 标准请求配置
export const CONFIG = () => ({
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${localStorage.getItem('token')}`,
  },
  withCredentials: true,
});

// 文件上传配置
export const FILE_CONFIG = () => ({
  headers: {
    'Content-Type': 'multipart/form-data',
    'Authorization': `Bearer ${localStorage.getItem('token')}`,
  },
  timeout: 300000, // 5分钟超时（文件上传需要更长时间）
});

// 删除配置
export const DELETE_CONFIG = () => ({
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${localStorage.getItem('token')}`,
  },
  data: {}, // DELETE请求需要data时添加
});

// 导出配置
export const EXPORT_CONFIG = () => ({
  responseType: 'blob', // 导出文件需要blob类型
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('token')}`,
  },
});
```

### API函数调用模式（Callback模式）

项目中的API函数采用callback模式设计，支持传统的回调函数调用方式：

```javascript
// API函数签名
export default {
  // 通用模式：cb(成功回调), errorCb(错误回调), payload(请求参数)
  functionName(cb, errorCb, payload) {
    // 实现逻辑
  }
}

// 使用示例
import projectAPI from '@/api/project';

// 1. 使用回调函数
projectAPI.getProject(
  (response) => {
    // 成功回调
    console.log('成功:', response.data);
  },
  (error) => {
    // 错误回调
    console.error('失败:', error);
  },
  { project_id: 123 }
);

// 2. 使用Promise包装（推荐）
function getProjectPromise(payload) {
  return new Promise((resolve, reject) => {
    projectAPI.getProject(resolve, reject, payload);
  });
}

// 3. 使用async/await（现代用法）
async function loadProject() {
  try {
    const response = await getProjectPromise({ project_id: 123 });
    console.log('项目数据:', response.data);
  } catch (error) {
    console.error('加载失败:', error);
  }
}
```

**callback模式的优势**：
- 兼容性：支持传统的回调函数调用方式
- 灵活性：可以灵活处理成功和失败回调
- 可包装：可以轻松包装为Promise/async-await模式
- 错误处理：每个API调用都可以单独处理错误

---

## API模块总览

### 模块对应关系

```
API文件              业务模块              Store模块                  Views目录
────────────────────────────────────────────────────────────────────────────────────
project.ts       →   项目管理       →   stores/project/      →   views/project/
org.ts           →   组织管理       →   stores/org/          →   views/management/
vulnerability.ts →   漏洞管理       →   stores/vulnerability/→   views/vulnerability/
component.ts     →   组件库         →   stores/component/    →   views/component/
compliance.ts    →   合规管理       →   stores/compliance/   →   views/compliance/
scan.ts          →   扫描管理       →   stores/scan/         →   views/scan/
report.ts        →   报告管理       →   stores/report/       →   views/report/
user.ts          →   用户管理       →   stores/user/         →   views/login/
management.ts    →   系统管理       →   stores/management/   →   views/admin/
sast.ts          →   SAST扫描       →   stores/sast/         →   views/scan/
```

### API函数命名规范

| 操作类型 | 命名前缀 | 示例 |
|----------|----------|------|
| 获取列表 | `get` / `list` | `getProjects()`, `listComponents()` |
| 获取详情 | `get` | `getProject(projectId)` |
| 创建 | `create` / `add` | `createProject()`, `addProject()` |
| 更新 | `update` / `patch` | `updateProject()`, `patchProject()` |
| 删除 | `delete` / `remove` | `deleteProject()`, `removeUser()` |
| 上传 | `upload` | `uploadFile()`, `postProjectUpload()` |
| 导出 | `export` | `exportData()`, `downloadReport()` |

---

## 核心API详解

### 1. Project API（项目管理）

**文件**: `src/api/project.ts`

**功能列表**:

```typescript
// 项目CRUD
export default {
  // 创建项目
  addProject(cb, errorCb, payload) { ... }

  // 获取项目详情
  getProject(cb, errorCb, payload) { ... }

  // 获取项目列表
  getLargeProject(cb, errorCb, payload) { ... }
  getLatestProject(cb, errorCb, payload) { ... }

  // 更新项目
  updateProject(cb, errorCb, payload) { ... }
  updateProjectSetting(cb, errorCb, payload) { ... }

  // 删除项目
  deleteProject(cb, errorCb, payload) { ... }

  // 项目扫描
  codeUpload(cb, errorCb, payload) { ... }
  postProjectUpload(cb, errorCb, project_id, payload) { ... }

  // 扫描策略
  getProjectScanPolicy(cb, errorCb, payload) { ... }
  updateProjectScanPolicySetting(cb, errorCb, payload) { ... }

  // 项目导入导出
  postProjectImportUrl(cb, errorCb, payload) { ... }
  exportProjectPDF(cb, errorCb, payload) { ... }
  exportProjectExcel(cb, errorCb, payload) { ... }

  // 项目依赖
  getProjectDependency(cb, errorCb, payload) { ... }
  getProjectDependencyDirect(cb, errorCb, payload) { ... }

  // 项目统计
  getProjectVulnerability(cb, errorCb, payload) { ... }
  getProjectIssues(cb, errorCb, payload) { ... }
  getProjectCount(cb, errorCb, payload) { ... }

  // 项目设置
  updateProjectScanningSetting(cb, errorCb, payload) { ... }
  updateProjectData(cb, errorCb, payload) { ... }
  updateProjectCustomFields(cb, errorCb, payload) { ... }

  // 团队管理
  getProjectTeam(cb, errorCb, payload) { ... }
  createProjectTeam(cb, errorCb, payload) { ... }
  patchProjectTeam(cb, errorCb, payload) { ... }
  deleteProjectTeam(cb, errorCb, payload) { ... }

  // 分支管理
  getProjectBranches(cb, errorCb, payload) { ... }
  createProjectBranch(cb, errorCb, payload) { ... }
  updateProjectBranch(cb, errorCb, payload) { ... }
  deleteProjectBranch(cb, errorCb, payload) { ... }

  // 项目关联
  createJiraAssociationOfProject(cb, errorCb, payload) { ... }
  getGitlabIntegration(cb, errorCb, payload) { ... }
  getGitlabProjects(cb, errorCb, payload) { ... }

  // CI/CD集成
  getProjectCicdConfig(cb, errorCb, payload) { ... }
  postProjectCicdConfig(cb, errorCb, payload) { ... }
  updateProjectCicdConfig(cb, errorCb, payload) { ... }
}
```

**总函数数量**: 50+

---

### 2. Org API（组织管理）

**文件**: `src/api/org.ts`

```typescript
export default {
  // 组织CRUD
  getOrgs(cb, errorCb, payload) { ... }
  createOrg(cb, errorCb, payload) { ... }
  updateOrg(cb, errorCb, payload) { ... }
  deleteOrg(cb, errorCb, payload) { ... }

  // 组织详情
  getOrg(cb, errorCb, payload) { ... }
  getOrgSubscriptionUsage(cb, errorCb, payload) { ... }
  getOrgDirectory(cb, errorCb, payload) { ... }

  // 成员管理
  getOrgMembers(cb, errorCb, payload) { ... }
  postOrgMember(cb, errorCb, payload) { ... }
  patchOrgMember(cb, errorCb, payload) { ... }
  deleteOrgMember(cb, errorCb, payload) { ... }

  // 组织设置
  getOrgSetting(cb, errorCb, payload) { ... }
  updateOrgSetting(cb, errorCb, payload) { ... }
  uploadOrgLogo(cb, errorCb, payload) { ... }

  // 集成配置
  getIntegrationConfigs(cb, errorCb, payload) { ... }
  createIntegrationConfig(cb, errorCb, payload) { ... }
  patchIntegrationConfig(cb, errorCb, payload) { ... }
  deleteIntegrationConfig(cb, errorCb, payload) { ... }

  // Jira集成
  getJiraIntegration(cb, errorCb, payload) { ... }
  createJiraIntegration(cb, errorCb, payload) { ... }
  updateJiraIntegration(cb, errorCb, payload) { ... }
  deleteJiraIntegration(cb, errorCb, payload) { ... }
  getJiraProjects(cb, errorCb, payload) { ... }

  // 许可证
  getOrgLicenses(cb, errorCb, payload) { ... }
  createOrgLicense(cb, errorCb, payload) { ... }
  deleteOrgLicense(cb, errorCb, payload) { ... }
}
```

---

### 3. Vulnerability API（漏洞管理）

**文件**: `src/api/vulnerability.ts`

```typescript
export default {
  // 漏洞列表
  getVulnerabilities(cb, errorCb, payload) { ... }
  getAllVulnerabilities(cb, errorCb, payload) { ... }
  getLargeVulnerabilities(cb, errorCb, payload) { ... }

  // 漏洞详情
  getVulnerability(cb, errorCb, payload) { ... }
  getVulnerabilityDetail(cb, errorCb, payload) { ... }

  // 漏洞分类
  getVulnerabilityClasses(cb, errorCb, payload) { ... }
  getVulnerabilityCategories(cb, errorCb, payload) { ... }

  // 漏洞统计
  getVulnerabilitySeverityCount(cb, errorCb, payload) { ... }
  getVulnerabilityTrend(cb, errorCb, payload) { ... }

  // 漏洞操作
  updateVulnerabilityStatus(cb, errorCb, payload) { ... }
  batchUpdateVulnerability(cb, errorCb, payload) { ... }

  // 导出
  exportVulnerabilities(cb, errorCb, payload) { ... }
}
```

---

## 请求/响应标准

### 请求结构

#### GET请求

```typescript
// 带查询参数
axios.get(`${API_BASE}/projects`, {
  params: {
    page: 1,
    page_size: 10,
    search: 'keyword',
    status: 'active'
  }
})

// 带路径参数
axios.get(`${API_BASE}/projects/${projectId}`, CONFIG())
```

#### POST/PUT/PATCH请求

```typescript
// 创建项目
axios.post(
  `${API_BASE}/projects`,
  JSON.stringify({
    name: 'New Project',
    description: 'Project description',
    settings: { ... }
  }),
  CONFIG()
)

// 更新项目
axios.patch(
  `${API_BASE}/projects/${projectId}`,
  JSON.stringify({
    name: 'Updated Name',
    status: 'active'
  }),
  CONFIG()
)
```

### 响应结构（标准）

```typescript
// 成功响应
interface ApiResponse<T = any> {
  code: number;        // 业务状态码 (200 = 成功)
  message: string;     // 消息提示
  data: T;            // 业务数据
  total?: number;     // 分页时返回总数
  count?: number;     // 数据条数
}

// 示例
{
  "code": 200,
  "message": "success",
  "data": {
    "id": 123,
    "name": "Project Name",
    "created_at": "2025-11-26T10:00:00Z"
  },
  "total": 100
}
```

### 响应结构（列表查询）

```typescript
{
  "code": 200,
  "message": "success",
  "data": [
    { "id": 1, "name": "Project A" },
    { "id": 2, "name": "Project B" }
  ],
  "total": 2,
  "count": 2
}
```

### 响应结构（错误）

```typescript
// 业务错误（code !== 200）
{
  "code": 400,
  "message": "参数错误：项目名称不能为空",
  "data": null
}

// HTTP错误（由拦截器统一处理）
// 401 Unauthorized
// 403 Forbidden
// 404 Not Found
// 500 Internal Server Error
```

---

## 错误处理机制

### 1. Axios拦截器统一处理

```typescript
// 响应拦截器（伪代码）
axiosInstance.interceptors.response.use(
  (response) => {
    const res = response.data;

    // 1. 检查业务状态码
    if (res.code !== 200) {
      // 1.1 处理认证错误
      if (res.code === 401) {
        // Token过期，清除token并跳转登录
        localStorage.removeItem('token');
        window.location.href = '/login';
      }

      // 1.2 显示错误消息
      ElMessage.error(res.message || '请求失败');

      // 1.3 返回Promise拒绝
      return Promise.reject(new Error(res.message || 'Error'));
    }

    // 2. 业务成功，返回数据
    return res;
  },

  (error) => {
    // 3. HTTP错误（网络错误、超时等）
    console.error('API Error:', error);

    // 3.1 网络超时
    if (error.code === 'ECONNABORTED') {
      ElMessage.error('请求超时，请检查网络连接');
    }
    // 3.2 无响应（网络断开）
    else if (!error.response) {
      ElMessage.error('网络连接失败，请检查网络');
    }
    // 3.3 HTTP状态错误
    else {
      const status = error.response.status;
      switch (status) {
        case 401:
          ElMessage.error('登录已过期，请重新登录');
          logout();
          break;
        case 403:
          ElMessage.error('没有权限访问此资源');
          break;
        case 404:
          ElMessage.error('请求的资源不存在');
          break;
        case 500:
        case 502:
        case 503:
          ElMessage.error('服务器错误，请稍后重试');
          break;
        default:
          ElMessage.error(`请求失败: ${status}`);
      }
    }

    return Promise.reject(error);
  }
);
```

### 2. 组件中的错误处理

```typescript
// ✅ 推荐：使用 try/catch 捕获错误
try {
  loading.value = true;
  const response = await getProject({ project_id: 123 });
  project.value = response.data;
} catch (error) {
  // 错误已被拦截器处理和提示
  // 这里可以做额外的处理（如日志记录）
  console.error('获取项目失败:', error);
} finally {
  loading.value = false;
}

// ❌ 不推荐：不处理错误
getProject({ project_id: 123 })
  .then(res => {
    project.value = res.data;
  });
// 错误没有捕获，会被拦截器处理，但无法在组件层面做清理工作
```

### 3. 错误处理最佳实践

```typescript
// 1. 总是使用 try/catch
async function loadData() {
  try {
    loading.value = true;
    const response = await getProjects({ page: 1 });
    projects.value = response.data;
  } catch (error) {
    // 记录错误日志
    console.error('Load error:', error);

    // 额外的错误处理（如回滚状态）
    projects.value = [];
  } finally {
    loading.value = false;
  }
}

// 2. 批量操作错误处理
async function batchUpdate() {
  try {
    await Promise.all([
      updateProject({ id: 1, name: 'A' }),
      updateProject({ id: 2, name: 'B' }),
      updateProject({ id: 3, name: 'C' })
    ]);
    ElMessage.success('批量更新成功');
  } catch (error) {
    ElMessage.warning('部分更新失败，请重试');
    // 部分成功的处理...
  }
}

// 3. 表单提交错误处理
async function handleSubmit(formData) {
  try {
    submitting.value = true;
    await validateForm(formData); // 前端验证
    await createProject(formData);
    ElMessage.success('创建成功');
    router.push('/projects');
  } catch (error) {
    if (error.name === 'ValidationError') {
      // 验证错误
      validationErrors.value = error.errors;
    } else {
      // API错误（已被拦截器提示）
      console.error('Submit error:', error);
    }
  } finally {
    submitting.value = false;
  }
}
```

---

## API调用示例

### 示例 1: 获取项目列表（带分页和筛选）

```typescript
// API调用
import { getLargeProject } from '@/api/project';

// 组件中使用
async function loadProjects() {
  try {
    loading.value = true;

    const response = await getLargeProject(
      (data) => {
        // 成功回调
        projects.value = data.results;
        total.value = data.count;
      },
      (error) => {
        // 错误回调
        console.error('加载失败:', error);
      },
      {
        org_id: currentOrgId,
        search: searchKeyword,      // 搜索关键词
        ordering: '-created_at',     // 排序（最新在前）
        page: currentPage,           // 当前页
        page_size: pageSize,         // 每页数量
        status: filters.status,      // 状态筛选
        visibility: filters.visibility // 可见性筛选
      }
    );
  } catch (error) {
    console.error('API Error:', error);
  } finally {
    loading.value = false;
  }
}

// 响应数据格式
{
  "code": 200,
  "message": "success",
  "data": {
    "count": 125,
    "results": [
      {
        "id": 1,
        "name": "My Project",
        "description": "Project description",
        "created_at": "2025-11-26T10:00:00Z",
        "updated_at": "2025-11-26T15:30:00Z",
        "status": "active",
        "visibility": "private",
        "owner": {
          "id": 101,
          "username": "john_doe",
          "email": "john@example.com"
        },
        "stats": {
          "vulnerabilities": 15,
          "components": 234,
          "last_scan": "2025-11-26T14:00:00Z"
        }
      }
    ]
  }
}
```

### 示例 2: 创建项目（表单提交）

```typescript
// API调用
import { addProject } from '@/api/project';

// 组件中使用
async function handleCreateProject() {
  try {
    submitting.value = true;

    // 准备数据
    const payload = {
      activeOrgId: currentOrgId,
      name: formData.name,
      description: formData.description,
      visibility: formData.visibility,
      settings: {
        auto_scan: formData.autoScan,
        scan_policy: formData.scanPolicy,
        notification_enabled: formData.notificationEnabled
      },
      custom_fields: formData.customFields
    };

    await addProject(
      (response) => {
        // 成功
        ElMessage.success('项目创建成功');
        const projectId = response.data.id;

        // 跳转到项目详情页
        router.push(`/org/${currentOrgId}/project/${projectId}`);
      },
      (error) => {
        // 错误（已被拦截器处理）
        console.error('创建失败:', error);
      },
      payload
    );
  } catch (error) {
    console.error('API Error:', error);
  } finally {
    submitting.value = false;
  }
}
```

### 示例 3: 更新项目（部分更新）

```typescript
// API调用
import { updateProject } from '@/api/project';

// 组件中使用
async function handleUpdateProject() {
  try {
    updating.value = true;

    // 1. 获取原始数据
    const originalProject = await getProject({ project_id: projectId });

    // 2. 准备更新数据（只更新变化的字段）
    const updateData = {
      activeOrgId: currentOrgId,
      projectId: projectId,
      name: formData.name,
      description: formData.description
    };

    // 3. 发送更新请求
    await updateProject(
      (response) => {
        ElMessage.success('更新成功');

        // 更新本地状态
        project.value = { ...project.value, ...response.data };
      },
      (error) => {
        // 错误处理
        console.error('更新失败:', error);

        // 回滚表单数据
        resetForm();
      },
      updateData
    );
  } finally {
    updating.value = false;
  }
}
```

### 示例 4: 删除项目（带确认）

```typescript
// API调用
import { deleteProject } from '@/api/project';

// 组件中使用
async function handleDeleteProject(projectId: number) {
  try {
    // 1. 确认删除
    const confirmed = await ElMessageBox.confirm(
      '确定要删除这个项目吗？此操作不可恢复。',
      '警告',
      {
        confirmButtonText: '删除',
        cancelButtonText: '取消',
        type: 'warning',
        confirmButtonClass: 'el-button--danger'
      }
    );

    if (!confirmed) return;

    deleting.value = true;

    // 2. 发送删除请求
    await deleteProject(
      (response) => {
        ElMessage.success('项目已删除');

        // 从列表中移除
        projects.value = projects.value.filter(p => p.id !== projectId);

        // 更新总数
        total.value -= 1;
      },
      (error) => {
        console.error('删除失败:', error);
        // 错误已被拦截器提示
      },
      {
        org_id: currentOrgId,
        id: projectId
      }
    );
  } catch (error) {
    // 用户取消删除
    if (error === 'cancel') {
      ElMessage.info('已取消删除');
    }
  } finally {
    deleting.value = false;
  }
}
```

### 示例 5: 批量操作（批量更新漏洞状态）

```typescript
// API调用
import { batchUpdateVulnerability } from '@/api/vulnerability';

// 组件中使用
async function handleBatchUpdateStatus(
  vulnerabilityIds: number[],
  newStatus: 'active' | 'resolved' | 'ignored'
) {
  try {
    batchUpdating.value = true;

    // 准备批量更新数据
    const payload = {
      ids: vulnerabilityIds,
      status: newStatus,
      updated_by: currentUserId,
      updated_at: new Date().toISOString()
    };

    // 发送批量更新请求
    await batchUpdateVulnerability(
      (response) => {
        // 部分成功/失败的提示
        const data = response.data;
        if (data.failures && data.failures.length > 0) {
          ElMessage.warning(`成功更新 ${data.success} 个，失败 ${data.failures.length} 个`);

          // 显示失败详情
          showFailureDetails(data.failures);
        } else {
          ElMessage.success(`成功更新 ${data.success} 个漏洞`);
        }

        // 刷新列表
        loadVulnerabilities();

        // 清除选择
        selectedVulnerabilities.value = [];
      },
      (error) => {
        console.error('批量更新失败:', error);
      },
      payload
    );
  } catch (error) {
    console.error('API Error:', error);
  } finally {
    batchUpdating.value = false;
  }
}

// 批量选择处理
function handleSelectAll(selection: Vulnerability[]) {
  selectedVulnerabilities.value = selection.map(item => item.id);
}

function handleSelectChange(selection: Vulnerability[]) {
  selectedVulnerabilities.value = selection.map(item => item.id);
}
```

### 示例 6: 带依赖关系的请求（获取组织信息后再获取项目）

```typescript
// API调用
import { getOrg } from '@/api/org';
import { getLargeProject } from '@/api/project';

// 组件中使用（使用async/await简化回调）
async function loadDashboardData() {
  try {
    loading.value = true;

    // 1. 获取当前组织信息
    const orgResponse = await getOrg({
      activeOrgId: currentOrgId
    });

    const orgData = orgResponse.data;

    // 2. 根据组织层级获取项目（依赖组织数据）
    const projectResponse = await getLargeProject(
      undefined, // 使用 Promise 方式而非回调
      undefined,
      {
        org_id: currentOrgId,
        visibility: orgData.is_admin ? 'all' : 'member_of', // 根据权限筛选
        page: 1,
        page_size: 10
      }
    );

    // 3. 获取组织统计信息
    const usageResponse = await getOrgSubscriptionUsage({
      org_id: currentOrgId
    });

    // 4. 更新状态
    org.value = orgData;
    projects.value = projectResponse.data.results;
    usageStats.value = usageResponse.data;

  } catch (error) {
    console.error('加载仪表板数据失败:', error);
    ElMessage.error('加载数据失败');
  } finally {
    loading.value = false;
  }
}

// 或者使用 Promise.all 并行请求（无依赖关系时）
async function loadParallelData() {
  try {
    loading.value = true;

    // 并行执行无依赖的请求
    const [orgRes, projectsRes, statsRes] = await Promise.all([
      getOrg({ activeOrgId: currentOrgId }),
      getLargeProject(undefined, undefined, { org_id: currentOrgId }),
      getOrgStats({ org_id: currentOrgId })
    ]);

    org.value = orgRes.data;
    projects.value = projectsRes.data.results;
    stats.value = statsRes.data;

  } catch (error) {
    console.error('加载失败:', error);
  } finally {
    loading.value = false;
  }
}
```

---

## 文件上传处理

### 上传代码包（项目扫描）

```typescript
// API调用
import { postProjectUpload } from '@/api/project';

// 组件中使用
async function handleFileUpload(file: File) {
  try {
    uploading.value = true;

    // 1. 创建FormData
    const formData = new FormData();
    formData.append('file', file);
    formData.append('filename', file.name);
    formData.append('filesize', file.size.toString());

    // 2. 上传文件
    const response = await postProjectUpload(
      undefined,
      undefined,
      projectId,
      formData
    );

    // 3. 处理响应
    if (response.data.code === 200) {
      ElMessage.success('上传成功');

      // 开始扫描
      await startScan(projectId);
    }
  } catch (error) {
    console.error('上传失败:', error);
    ElMessage.error('文件上传失败');
  } finally {
    uploading.value = false;
  }
}

// 使用 Element Plus Upload 组件
<template>
  <el-upload
    :action="uploadUrl"
    :on-success="handleUploadSuccess"
    :on-error="handleUploadError"
    :before-upload="beforeUpload"
    :headers="uploadHeaders"
    accept=".zip,.tar.gz"
  >
    <el-button type="primary">选择文件</el-button>
  </el-upload>
</template>

<script setup>
const uploadUrl = `${API_BASE}/projects/${projectId}/upload`;

const uploadHeaders = {
  Authorization: `Bearer ${localStorage.getItem('token')}`
};

function beforeUpload(file: File) {
  const maxSize = 100 * 1024 * 1024; // 100MB
  if (file.size > maxSize) {
    ElMessage.error('文件大小不能超过 100MB');
    return false;
  }
  return true;
}

function handleUploadSuccess(response: any) {
  ElMessage.success('上传成功');
  loadProject(); // 刷新项目信息
}

function handleUploadError(error: any) {
  console.error('上传失败:', error);
  ElMessage.error('文件上传失败');
}
</script>
</script>
```

---

## 总结

API接口层是前端与后端交互的核心，具有以下特点：

1. **模块化设计**: 按业务模块组织API，便于维护和查找
2. **统一配置**: 通过Axios实例统一管理baseURL、超时、拦截器等
3. **标准命名**: 统一的CRUD命名规范，提高代码可读性
4. **类型安全**: TypeScript类型定义，减少错误
5. **错误统一处理**: 拦截器集中处理错误，避免重复代码
6. **回调/Promise双支持**: 既支持传统回调，也支持现代Promise/async-await

### 快速参考

```typescript
// 导入API
import { getProject, createProject, updateProject, deleteProject } from '@/api/project';

// 使用async/await（推荐）
const loadData = async () => {
  try {
    const response = await getProject({ project_id: 1 });
    return response.data;
  } catch (error) {
    console.error(error);
  }
};

// 使用回调函数
getProject(
  (response) => { console.log('Success:', response.data); },
  (error) => { console.error('Error:', error); },
  { project_id: 1 }
);
```

---

## 相关文档

- [Pinia状态管理](./stores_guide.md) - 如何使用Store管理API返回的数据
- [模块开发指南](./module_development_guide.md) - 如何为新增的模块创建API文件
- [AI_Coding_Context.md](./AI_Coding_Context.md) - 快速了解API使用场景
- [多品牌配置系统](./multibrand_config.md) - API网关配置说明

---

## 附录

### A. API配置文件（完整示例）

```typescript
// src/api/config.ts

// API网关地址（从环境变量读取）
export const API_BASE = import.meta.env.VITE_API_GATEWAY;
export const API_BASE_V2 = `${API_BASE}/v2`;

// 特殊路径前缀
export const TTS_API_BASE = `${API_BASE}/tts`;

// 标准请求配置（JSON）
export const CONFIG = () => ({
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${localStorage.getItem('token')}`,
  },
  withCredentials: true,
  timeout: 60000,
});

// 文件上传配置（multipart/form-data）
export const FILE_CONFIG = () => ({
  headers: {
    'Content-Type': 'multipart/form-data',
    'Authorization': `Bearer ${localStorage.getItem('token')}`,
  },
  withCredentials: true,
  timeout: 300000, // 5分钟超时
});

// 删除配置（某些DELETE请求需要data体）
export const DELETE_CONFIG = () => ({
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${localStorage.getItem('token')}`,
  },
  withCredentials: true,
});

// 导出文件配置（blob）
export const EXPORT_CONFIG = () => ({
  responseType: 'blob',
  headers: {
    'Authorization': `Bearer ${localStorage.getItem('token')}`,
  },
  withCredentials: true,
});
```

### B. 通用工具函数

```typescript
// src/api/utils.ts

// 分页参数过滤器
export function paginationFilter(pagination: {
  page: number;
  pageSize: number;
}): string {
  return `page=${pagination.page}&page_size=${pagination.pageSize}`;
}

// 额外过滤器（用于筛选条件）
export function additionalFilters(filters: Record<string, any>): string {
  return Object.entries(filters)
    .filter(([_, value]) => value !== null && value !== undefined && value !== '')
    .map(([key, value]) => `${key}=${encodeURIComponent(value)}`)
    .join('&');
}

// 构建完整查询URL
export function buildQueryUrl(base: string, params: Record<string, any>): string {
  const query = additionalFilters(params);
  return query ? `${base}?${query}` : base;
}
```

---

**文档版本**: v1.0
**最后更新**: 2025-11-26
**维护者**: AI Assistant (Claude Sonnet 4.5)

---

## 文档维护记录

### 2025-11-26
- 初始版本创建
- 包含18个API模块的完整说明
- 提供6个典型API调用示例
- 包含错误处理机制和最佳实践
- 提供文件上传处理示例

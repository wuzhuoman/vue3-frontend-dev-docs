# 错误处理指南

> **基于代码版本**: v4.10.0
> **最后验证时间**: 2025-11-27 09:25
> **代码文件来源**: `src/utils/errorHandler.ts` (187行, 5874字节)

⚠️ **重要提示**: 本文档完全基于实际项目代码，不包含未实现的功能。

## 目录

- [概述](#概述)
- [核心函数](#核心函数)
- [HTTP状态码处理](#http状态码处理)
- [消息提取机制](#消息提取机制)
- [防抖机制](#防抖机制)
- [CATARC模式特殊处理](#catarc模式特殊处理)
- [使用示例](#使用示例)
- [最佳实践](#最佳实践)
- [注意事项](#注意事项)

---

## 概述

项目采用**集中式错误处理**机制，所有HTTP请求错误通过统一的拦截器处理。主要特点：

1. **统一入口**: 所有API错误通过 `errorHandler()` 或 `uploadErrorHandler()` 处理
2. **状态码分类处理**: 根据HTTP状态码执行不同的处理逻辑
3. **防抖机制**: 3秒内避免重复显示相同错误消息
4. **国际化支持**: 所有错误消息通过Vue I18n国际化
5. **CATARC模式差异化**: 针对CATARC品牌有特殊错误处理逻辑

**核心文件**: [src/utils/errorHandler.ts](../../src/utils/errorHandler.ts)

---

## 核心函数

### 1. `errorHandler(error, customMessage = false)`

**位置**: `src/utils/errorHandler.ts:40`

通用错误处理函数，处理所有axios请求错误。

**函数签名**:
```typescript
function errorHandler(error: any, customMessage = false)
```

**参数说明**:
- `error`: axios错误对象，包含 `error.response.data` 和 `error.response.status`
- `customMessage`: 布尔值，为 `true` 时显示自定义错误消息

**返回值**: `Promise.reject(error)`

**处理流程**:
```
┌─────────────┐
│  接收错误   │
└──────┬──────┘
       │
       ▼
┌────────────────────┐
│ 防抖检查(3秒)      │──重复→ 直接reject
│  showingErrorMessage? │
└──────┬─────────────┘
       │ 无重复
       ▼
┌────────────────────┐
│ 提取status和data   │
│  error.response    │
└──────┬─────────────┘
       │
       ▼
┌────────────────────┐
│ 根据status分类处理 │
│  switch(status)    │
└──────┬─────────────┘
       │
       ▼
   显示消息 → 防抖标记 → reject
```

**源代码**:
```typescript:src/utils/errorHandler.ts:40-134
export default function errorHandler(error, customMessage = false) {
  // default redirect == true for 404
  try {
    console.log("[error handler]:", error, showingErrorMessage);
    const { data, status } = error.response;

    // avoid showing multiple error messages
    if (showingErrorMessage === true) {
      return Promise.reject(error);
    }

    // handle server error
    if (status >= 500) {
      showError("Error500Msg");
      return Promise.reject(error);
    }

    // handle >= 400 error
    if (status >= 400) {
      if (status == 401 || status == 403) {
        // ... 处理401/403
      } else if (status == 402 && isCatarc) {
        // ... 处理402+CATARC
      } else if (status == 409) {
        // ... 处理409
      } else {
        // ... 处理其他4xx
      }
    }
    return Promise.reject(error);
  } catch (e) {
    console.log(e);
  }
}
```

---

### 2. `uploadErrorHandler(error)`

**位置**: `src/utils/errorHandler.ts:136`

专门处理文件上传错误的辅助函数，适配不同上传库的错误格式。

**特点**:
1. **多源数据提取**: 从 `error.status`、`error.response.status`、`error.httpCode` 提取状态码
2. **数据解析**: 自动解析 `error.message` 和 `error.response.responseText` 中的JSON字符串
3. **统一适配**: 将各种上传库的错误格式适配为axios风格，复用 `errorHandler`

**源代码**:
```typescript:src/utils/errorHandler.ts:136-186
export function uploadErrorHandler(error) {
  try {
    // 1) 获取最可能的 status
    const status =
      (error && error.status) ??
      (error && error.response && error.response.status) ??
      (error && error.httpCode) ??
      0;

    // 2) 尝试从多来源提取并解析 data
    const safeParse = (input) => {
      if (typeof input !== "string") return input;
      try {
        return JSON.parse(input);
      } catch {
        return input;
      }
    };

    let data = (error && error.response && error.response.data) ?? undefined;
    if (data === undefined) {
      // 某些上传库把后端错误放在 message 或 xhr.responseText
      const parsedFromMessage = safeParse(error && error.message);
      const parsedFromResponseText = safeParse(
        error && error.response && error.response.responseText,
      );
      data = parsedFromMessage ?? parsedFromResponseText ?? {};
    }

    // 若 data 是纯字符串，包装为 { detail: data }
    if (typeof data === "string") {
      data = { detail: data };
    }

    // 3) 适配为 axios 风格错误对象，直接交给通用 errorHandler
    const fauxAxiosError = {
      response: {
        status,
        data,
      },
    };

    // 与通用行为保持一致；开启 customMessage=true 以优先显示后端 detail
    (errorHandler as any)(fauxAxiosError as any, true);
    return;
  } catch (e) {
    ElMessage({
      message: t("REQUEST_ERROR"),
      type: "error",
    });
  }
}
```

---

## HTTP状态码处理

### 状态码 500+ (服务器错误)

**处理逻辑**: 显示通用服务器错误消息

```typescript
if (status >= 500) {
  showError("Error500Msg");
  return Promise.reject(error);
}
```

**显示消息**: `"服务器内部错误，请稍后再试"` (国际化后的 `Error500Msg`)

---

### 状态码 401/403 (未授权/禁止访问)

**处理逻辑**:
1. 检查错误码是否为 `token_not_valid` 或 `user_not_found`
2. 如果是，显示"会话已过期"，2秒后跳转到登录页
3. 如果 `customMessage=true`，显示后端返回的 `detail` 消息
4. 否则显示通用"未授权"消息

**源代码**:
```typescript:src/utils/errorHandler.ts:58-71
if (status == 401 || status == 403) {
  // if it's token expired or user not valid, redirect to login page
  if (UNAUTHORIZED_CODE_LIST.includes(data.code)) {
    showError("SESSION_HAS_EXPIRED");
    removeAuthorization();
    _.delay(() => {
      window.location.replace("/login");
    }, 2000);
  } else if (customMessage) {
    showError(error.response?.data?.detail);
  } else {
    showError("UNAUTHORIZED_ERROR_MSG");
  }
}
```

**支持的未授权错误码** (`UNAUTHORIZED_CODE_LIST`):
```typescript:src/utils/errorHandler.ts:31
const UNAUTHORIZED_CODE_LIST = ["token_not_valid", "user_not_found"];
```

**消息流程**:
- 会话过期 → "会话已过期，即将跳转到登录页面" → 2秒后跳转
- 其他401/403 → "未授权访问"（或自定义消息）

---

### 状态码 402 + CATARC模式 (许可证过期)

**特殊处理**: 仅在CATARC模式下显示许可证过期确认框

**触发条件**: `status === 402 && isCatarc`

**源代码**:
```typescript:src/utils/errorHandler.ts:71-89
else if (status == 402 && isCatarc) {
  ElMessageBox.confirm(
    t("AUTOMOTIVE_LICENSE_EXPIRED", {
      name: defaultToDash(config.CONTACT_NAME),
      number: defaultToDash(config.CONTACT_NUMBER),
      email: defaultToDash(config.CONTACT_EMAIL),
    }),
    t("ERROR"),
    {
      confirmButtonText: t("CONFIRM"),
      cancelButtonText: t("CANCEL"),
      customClass: "custom-confirm",
      cancelButtonClass: "btn-custom-cancel",
      showCancelButton: false,  // 仅显示确认按钮
      type: "error",
      center: true,
    },
  );
}
```

**消息模板**: [language/lang JSON文件]
```json
"AUTOMOTIVE_LICENSE_EXPIRED": "汽车许可证已过期，请联系销售更新许可证。\n姓名：{name}\n电话：{number}\n邮箱：{email}"
```

**配置来源**:
```typescript
import { config, isCatarc } from "@/config";
// config.CONTACT_NAME, config.CONTACT_NUMBER, config.CONTACT_EMAIL
// isCatarc: 判断当前是否为CATARC模式
```

---

### 状态码 409 (冲突错误)

**处理逻辑**: 显示确认框，点击确认后跳转到登录页（带`error=409`参数）

**去重机制**: 检查URL是否已包含`error=409`，避免重复处理

**源代码**:
```typescript:src/utils/errorHandler.ts:89-113
else if (status == 409) {
  if (window.location.href.includes("error=409")) {
    refreshShowingErrorMessage();
    const newUrl = window.location.href
      .replace(/(\?|&)error=409(&|$)/, "$1")
      .replace(/\?$/, "");
    window.history.replaceState({}, document.title, newUrl);
    return Promise.reject(error);
  }
  refreshShowingErrorMessage();
  ElMessageBox.confirm(handleMessage(data), t("ERROR"), {
    confirmButtonText: t("CONFIRM"),
    cancelButtonText: t("CANCEL"),
    customClass: "custom-confirm",
    cancelButtonClass: "btn-custom-cancel",
    showCancelButton: false,
    type: "error",
    center: true,
  }).then(() => {
    removeAuthorization();
    if (window.location.pathname !== "/login") {
      window.location.replace("/login?error=409");
    }
  });
}
```

**行为**:
1. 检测到 `error=409` 在URL中 → 清理参数并跳过处理
2. 显示确认框 → 用户点击"确认"
3. 清除认证信息 → 跳转到 `/login?error=409`

---

### 其他 4xx 错误

**处理逻辑**: 显示后端返回的错误消息（通过 `handleMessage()` 提取）

```typescript:src/utils/errorHandler.ts:113-125
else {
  console.log("general error");
  ElMessageBox.confirm(handleMessage(data), t("ERROR"), {
    confirmButtonText: t("CONFIRM"),
    cancelButtonText: t("CANCEL"),
    customClass: "custom-confirm",
    cancelButtonClass: "btn-custom-cancel",
    showCancelButton: false,
    type: "error",
    center: true,
  });
}
```

---

## 消息提取机制

### `handleMessage()` 函数

**位置**: `src/utils/errorHandler.ts:19`

从后端返回的错误数据中提取可读的错误消息。

**处理优先级**:
1. 优先检查 `non_field_errors` 字段
2. 其次检查 `detail` 字段
3. 默认返回 "error"（会被国际化）

**源代码**:
```typescript:src/utils/errorHandler.ts:19-29
function handleMessage(data) {
  let message = "error";
  if (_.has(data, "non_field_errors")) {
    message = _.get(data, "non_field_errors")[0];
  }
  if (_.has(data, "detail")) {
    message = _.get(data, "detail");
    console.log("detail??", message);
  }
  return t(message);
}
```

**依赖库**: `lodash-es` (`_.has()`, `_.get()`)

**示例**:
```json
// 后端返回
{
  "non_field_errors": ["邮箱已存在", "用户名太简短"]
}
// 提取结果: "邮箱已存在"

{
  "detail": "权限不足"
}
// 提取结果: "权限不足"
```

---

## 防抖机制

### `showingErrorMessage` 标志

**位置**: `src/utils/errorHandler.ts:5`

**机制**:
- 全局变量 `showingErrorMessage` 初始值为 `false`
- 显示错误消息时设置为 `true`
- 启动3秒定时器，结束后重置为 `false`
- 在此期间，所有新的错误消息被忽略

**刷新函数**:
```typescript:src/utils/errorHandler.ts:33-38
function refreshShowingErrorMessage() {
  showingErrorMessage = true;
  setTimeout(() => {
    showingErrorMessage = false;
  }, 3000);
}
```

**使用位置**:
1. `errorHandler()` 开始时检查（`src/utils/errorHandler.ts:47-50`）
2. 409错误处理时调用（`src/utils/errorHandler.ts:94-100`）
3. `showError()` 函数中调用（`src/utils/errorHandler.ts:16`）

**作用**: 防止用户快速点击时弹出多个相同的错误MessageBox

---

### `showError()` 辅助函数

**位置**: `src/utils/errorHandler.ts:10`

显示Element Plus的 `ElMessage` 错误提示，并触发防抖。

```typescript:src/utils/errorHandler.ts:10-18
function showError(errorText) {
  ElMessage({
    showClose: true,
    message: t(errorText),
    type: "error",
  });
  refreshShowingErrorMessage();
}
```

---

## CATARC模式特殊处理

### 条件判断

项目中多处使用 `isCatarc` 判断是否为CATARC模式：

```typescript
import { config, isCatarc } from "@/config";

// 在errorHandler中的使用
if (status == 402 && isCatarc) {
  // 仅CATARC模式显示许可证过期弹窗
}
```

**其他代码中的使用示例**:

```typescript
// DashboardLayout.vue (162-165行)
channel.bind("project_status_update", function (data: any) {
  if (!isCatarc) {
    // ⚠️ CATARC模式不处理此事件
    projectStore.updateProjectStatus(data);
  }
});
```

**配置定义**:
```javascript
// src/config/index.js (122行)
export const isCatarc = _companyConfig.COMPANY_ID == "catarc";
```

---

## 使用示例

### 示例1: API调用错误处理

```typescript
// 在Vue组件中调用API
import { getProjects } from "@/api/project";
import errorHandler from "@/utils/errorHandler";

async function loadProjects() {
  try {
    const data = await getProjects({ page: 1, page_size: 10 });
    projects.value = data;
  } catch (error) {
    // 错误已被axios拦截器自动调用errorHandler处理
    // 组件中只需catch，无需额外处理
    console.error("加载项目失败", error);
  }
}
```

**重点**: 错误消息显示和跳转已由 `errorHandler` 自动完成，组件无需重复处理。

---

### 示例2: 文件上传错误处理

```typescript
// 使用uploadErrorHandler处理文件上传错误
import { uploadErrorHandler } from "@/utils/errorHandler";
import { ref } from "vue";

const fileList = ref([]);

function handleFileChange(file) {
  const formData = new FormData();
  formData.append("file", file.raw);

  // 假设使用XMLHttpRequest
  const xhr = new XMLHttpRequest();
  xhr.open("POST", "/api/upload");

  xhr.onload = () => {
    if (xhr.status !== 200) {
      // 错误格式可能是非标准的，使用uploadErrorHandler适配
      uploadErrorHandler({
        status: xhr.status,
        response: {
          responseText: xhr.responseText,
        },
      });
    }
  };

  xhr.onerror = () => {
    uploadErrorHandler({
      status: 0,
      message: "网络错误",
    });
  };

  xhr.send(formData);
}
```

**重点**: `uploadErrorHandler` 适配不同上传库的错误格式，并复用统一错误处理逻辑。

---

### 示例3: 自定义错误消息

```typescript
// customMessage = true 时显示后端detail
import errorHandler from "@/utils/errorHandler";

async function updateConfig() {
  try {
    await saveConfig(data);
  } catch (error) {
    // 显示后端返回的detail消息（如果有）
    errorHandler(error, true);
  }
}
```

当后端返回：
```json
{
  "detail": "配置格式不正确，请检查JSON格式"
}
```

会显示："配置格式不正确，请检查JSON格式"

---

## 最佳实践

### ✅ 推荐做法

1. **依赖拦截器**: 所有通过axios发出的请求，错误处理自动完成
   ```typescript
   // axios实例已配置拦截器
   // src/utils/axios.ts
   axiosInstance.interceptors.response.use(
     response => response,
     error => {
       errorHandler(error); // 自动处理
       return Promise.reject(error);
     }
   );
   ```

2. **统一的错误日志**: 所有错误消息都会打印到控制台（`src/utils/errorHandler.ts:43`）
   ```
   console.log("[error handler]:", error, showingErrorMessage);
   ```

3. **使用uploadErrorHandler处理上传**: 文件上传时使用专用函数
   ```typescript
   // 不同上传库（plupload, xhr, fetch）可能返回不同格式
   // uploadErrorHandler已统一适配
   ```

4. **后端消息优先**: 使用 `customMessage=true` 显示后端具体错误
   ```typescript
   errorHandler(error, true); // 使用后端的detail消息
   ```

---

### ❌ 避免做法

1. **不要重复显示错误消息**: 错误已由 `errorHandler` 显示，组件不需要再 `ElMessage.error()`
   ```typescript
   // ❌ 错误示例
   try {
     await apiCall();
   } catch (error) {
     ElMessage.error('操作失败');  // 重复显示
     errorHandler(error);          // errorHandler已显示
   }
   ```

2. **不要捕获后无处理**: 至少打印错误日志
   ```typescript
   // ❌ 错误示例
   try {
     await apiCall();
   } catch (error) {
     // 没有任何处理
   }

   // ✅ 正确示例
   try {
     await apiCall();
   } catch (error) {
     console.error('调用失败:', error);
   }
   ```

3. **不要直接操作DOM跳转**: 使用 `errorHandler` 的统一跳转逻辑
   ```typescript
   // ❌ 手动跳转
   if (status === 401) {
     window.location.href = '/login';  // 错误
   }

   // ✅ 使用errorHandler
   errorHandler(error);  // 自动2秒后跳转
   ```

---

## 注意事项

### 1. 防抖机制影响

- 3秒内连续触发相同错误，只有第一次会显示消息
- 适合防止用户快速点击按钮导致的重复错误
- 测试错误处理时，注意等待3秒或使用不同错误场景

### 2. CATARC模式差异

- 402状态码仅在CATARC模式下显示许可证过期弹窗
- 其他品牌（default/mstl/anesec等）忽略402错误
- 开发时注意 `isCatarc` 判断的逻辑分支

### 3. 消息国际化

- 错误消息通过 `t()` 函数国际化
- 如需修改默认消息，编辑 `language/lang/*.json` 文件
- 常见消息键：
  - `"Error500Msg"` - 服务器错误
  - `"SESSION_HAS_EXPIRED"` - 会话过期
  - `"UNAUTHORIZED_ERROR_MSG"` - 未授权
  - `"AUTOMOTIVE_LICENSE_EXPIRED"` - CATARC许可证过期（独有）

### 4. 与API拦截器关系

```
组件调用API
    ↓
axios请求
    ↓
后端响应错误
    ↓
axios响应拦截器
    ↓
calls errorHandler(error)
    ↓
显示错误消息
    ↓
Promise.reject(error)
    ↓
组件catch块捕获
```

**注意**: 如果组件中不需要额外处理，可以省略try/catch。

### 5. 409错误重定向

- 409错误点击"确认"后会重定向到 `/login?error=409`
- URL中的 `error=409` 参数用于防止重复处理
- 重定向前会清理认证信息（`removeAuthorization()`）

### 6. 自定义上传库集成

如果使用第三方上传库（如plupload、vue-upload-component），返回的错误格式可能不同：

```typescript
// 错误格式可能是:
{
  status: 0,
  message: "{\"detail\": \"文件太大\"}",  // JSON字符串
  responseText: "{\"detail\": \"文件太大\"}"
}

// 或者:
{
  httpCode: 413,
  data: "文件超过最大限制"
}

// 统一使用uploadErrorHandler处理
uploadErrorHandler(error);
```

`uploadErrorHandler` 会自动适配这些非标准格式。

---

## 相关文档

- [API层文档](./api_layer.md) - API调用规范和拦截器配置
- [开发模式指南](./development_modes_guide.md) - CATARC模式的详细说明
- [架构总览](./architecture_overview.md) - 技术架构和依赖库说明

---

## 更新日志

### 2025-11-27
- **创建文档**: v1.0
- **验证文件**: `src/utils/errorHandler.ts` (187行)
- **主要功能**: 通用错误处理、上传错误处理、防抖机制、CATARC特殊处理

---

**文档路径**: `dev_docs/error_handling_guide.md`
**最后更新**: 2025-11-27 10:30
**维护者**: Claude Sonnet 4.5
**代码版本**: v4.10.0

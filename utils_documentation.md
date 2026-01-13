# 工具函数文档

本文档汇总项目中所有工具函数，按功能分类并提供使用示例。

## 目录

- [日期处理](#日期处理)
- [格式化函数](#格式化函数)
- [助手函数](#助手函数)
- [图片处理](#图片处理)
- [消息盒子](#消息盒子)
- [权限辅助](#权限辅助)
- [正则表达式](#正则表达式)
- [扫描辅助](#扫描辅助)
- [排序处理](#排序处理)
- [第三方集成](#第三方集成)
- [树形数据](#树形数据结构)
- [WebSocket](#websocket工具)

---

## 日期处理

### 文件位置

```
src/utils/dateHelper.ts
```

### 函数清单

#### `time`

**描述：** 将日期时间格式化为本地时间字符串

**参数：**
```typescript
time(datetime: string): string
```

**使用示例：**
```typescript
import { time } from "@/utils/dateHelper"

const isoDate = "2024-01-15T08:30:00Z"
const formattedDate = time(isoDate)
// 输出: "2024-01-15, 16:30:00" (新加坡时间)
```

#### `date`

**描述：** 格式化日期为YYYY-MM-DD格式

**参数：**
```typescript
date(date: string): string
```

**使用示例：**
```typescript
import { date } from "@/utils/dateHelper"

const isoDate = "2024-01-15T08:30:00Z"
const formattedDate = date(isoDate)
// 输出: "2024-01-15"
```

#### `duration`

**描述：** 计算两个日期之间的持续时间

**参数：**
```typescript
duration(start: string, end: string): string
```

**使用示例：**
```typescript
import { duration } from "@/utils/dateHelper"

const start = "2024-01-15T08:30:00Z"
const end = "2024-01-16T10:45:00Z"
const result = duration(start, end)
// 输出: "1 D 2 h 15 m"
```

#### `timeFromNow`

**描述：** 获取相对时间（如"2小时前"）

**参数：**
```typescript
timeFromNow(_date: string): string
```

**使用示例：**
```typescript
import { timeFromNow } from "@/utils/dateHelper"

const date = "2024-01-15T08:30:00Z"
const relativeTime = timeFromNow(date)
// 输出: "2小时前" (根据当前时间计算)
```

#### `convertDurationInSec`

**描述：** 将秒数转换为易读的时间格式

**参数：**
```typescript
convertDurationInSec(duration: number): string
```

**使用示例：**
```typescript
import { convertDurationInSec } from "@/utils/dateHelper"

const seconds = 3725
const formatted = convertDurationInSec(seconds)
// 输出: "1h 2m 5s"
```

---

## 格式化函数

### 文件位置

```
src/utils/helperFunctions.ts
```

### 函数清单

#### `formatLineNumber`

**描述：** 格式化代码行号为指定位数

**参数：**
```typescript
formatLineNumber(lineNumber: number, totalLines: number): string
```

**使用示例：**
```typescript
import { formatLineNumber } from "@/composables/formatter"

const line1 = formatLineNumber(1, 100)
// 输出: "001"

const line50 = formatLineNumber(50, 1000)
// 输出: "0050"
```

#### `formatTimeWithMilliseconds`

**描述：** 将毫秒格式化为带毫秒的时间字符串

**参数：**
```typescript
formatTimeWithMilliseconds(time: number): string
```

**使用示例：**
```typescript
import { formatTimeWithMilliseconds } from "@/composables/formatter"

const time1 = formatTimeWithMilliseconds(1600)
// 输出: "1.6s"

const time2 = formatTimeWithMilliseconds(150)
// 输出: "150ms"
```

#### `formatChartXAxis`

**描述：** 格式化图表X轴时间标签

**参数：**
```typescript
formatChartXAxis(timestampList: number[]): number[]
```

**使用示例：**
```typescript
import { formatChartXAxis } from "@/composables/formatter"

const timestamps = [1704067200000, 1704153600000, 1704240000000]
const formatted = formatChartXAxis(timestamps)
// 返回处理后的时间戳数组，用于图表显示
```

---

## 助手函数

### 文件位置

```
src/utils/helperFunctions.ts
```

### 函数清单

#### `generateColorBySeverity`

**描述：** 根据严重级别生成对应的颜色（用于图表）

**参数：**
```typescript
generateColorBySeverity(severity: string): string
````

**严重级别与颜色：**
- Critical: "#f12c2e" (红色)
- High: "#f3951b" (橙色)
- Medium: "#fcee0a" (黄色)
- Low: "#42c66d" (绿色)
- Info: "#72b1e2" (蓝色)
- Unknown: "#666666" (灰色)

**使用示例：**
```typescript
import { generateColorBySeverity } from "@/utils/helperFunctions"

const criticalColor = generateColorBySeverity("Critical")
// 输出: "#f12c2e"

const highColor = generateColorBySeverity("High")
// 输出: "#f3951b"
```

#### `convertMapToArray`

**描述：** 将Map对象转换为数组

**参数：**
```typescript
convertMapToArray<T>(map: Map<string, T>): T[]
```

**使用示例：**
```typescript
import { convertMapToArray } from "@/utils/helperFunctions"

const map = new Map<string, any>()
map.set("item1", { id: 1, name: "Item 1" })
map.set("item2", { id: 2, name: "Item 2" })

const array = convertMapToArray(map)
// 输出: [{ id: 1, name: "Item 1" }, { id: 2, name: "Item 2" }]
```

#### `truncateHashedLicenseName`

**描述：** 截断哈希化的许可证名称（用于显示）

**参数：**
```typescript
truncateHashedLicenseName(licenseName: string, maxLength?: number): string
```

**使用示例：**
```typescript
import { truncateHashedLicenseName } from "@/utils/helperFunctions"

const longName = "This is a very long license name that needs to be truncated"
const truncated = truncateHashedLicenseName(longName, 30)
// 输出: "This is a very long lice..."
```

#### `detectIE11`

**描述：** 检测浏览器是否为IE11

**返回值：**
```typescript
boolean
```

**使用示例：**
```typescript
import { detectIE11 } from "@/utils/helperFunctions"

if (detectIE11()) {
  alert("Internet Explorer 11 is not supported. Please use a modern browser.")
}
```

---

## 图片处理

### 文件位置

```
src/utils/imgHelper.ts
```

### 函数清单

#### `dataUrlToFile`

**描述：** 将Data URL转换为File对象（用于本地图片上传）

**参数：**
```typescript
dataUrlToFile(dataUrl: string, filename: string, mimeType: string): Promise<File>
```

**使用示例：**
```typescript
import { dataUrlToFile } from "@/utils/imgHelper"

const dataUrl = "data:image/png;base64,iVBORw0KGgo..."
const file = await dataUrlToFile(dataUrl, "avatar.png", "image/png")

// 然后可以上传
uploadFile(file)
```

#### `convertLocalImageToBase64`

**描述：** 将本地图片转换为Base64

**参数：**
```typescript
convertLocalImageToBase64(file: File): Promise<string>
```

**使用示例：**
```typescript
import { convertLocalImageToBase64 } from "@/utils/imgHelper"

const input = document.querySelector('input[type="file"]')
input.addEventListener("change", async (e) => {
  const file = e.target.files[0]
  const base64 = await convertLocalImageToBase64(file)
  // base64: "data:image/jpeg;base64,/9j/4AAQSkZJRgABAQ..."
})
```

---

## 消息盒子

### 文件位置

```
src/utils/messageBoxes.ts
```

### 函数清单

#### `confirmBox`

**描述：** 确认对话框（带取消按钮）

**参数：**
```typescript
confirmBox(
  message: string,
  title?: string,
  confirmButtonText?: string,
  cancelButtonText?: string,
  type?: "success" | "warning" | "info" | "error"
): Promise<boolean>
```

**使用示例：**
```typescript
import { confirmBox } from "@/utils/messageBoxes"

const result = await confirmBox(
  "此操作将永久删除该项目，是否继续？",
  "警告",
  "确定",
  "取消",
  "warning"
)

if (result) {
  // 用户点击了"确定"
  await deleteProject()
} else {
  // 用户点击了"取消"
  console.log("取消删除")
}
```

#### `showToast`

**描述：** 显示提示消息（自动消失）

**参数：**
```typescript
showToast(
  message: string,
  type?: "success" | "warning" | "info" | "error",
  duration?: number
): void
```

**使用示例：**
```typescript
import { showToast } from "@/utils/messageBoxes"

// 成功消息
showToast("操作成功", "success", 3000)

// 错误消息
showToast("操作失败，请重试", "error")

// 警告消息
showToast("请注意：此操作有风险", "warning")
```

---

## 权限辅助

### 文件位置

```
src/utils/privilegesHelper.js
```

### 函数清单

#### `checkPrivileges`

**描述：** 检查用户是否具有指定权限

**参数：**
```javascript
checkPrivileges(requiredPrivileges, userPrivileges)
```

**使用示例：**
```javascript
import { checkPrivileges } from "@/utils/privilegesHelper"

const requiredPrivileges = ["delete_project", "edit_project"]
const userPrivileges = ["view_project", "edit_project", "create_project"]

const hasPrivileges = checkPrivileges(requiredPrivileges, userPrivileges)
// 输出: false (缺少 delete_project)
```

#### `checkUserPrivileges`

**描述：** 检查当前用户是否具有权限（使用全局状态）

**参数：**
```javascript
checkUserPrivileges(requiredPrivileges)
```

**使用示例：**
```javascript
import { checkUserPrivileges } from "@/utils/privilegesHelper"

// 在组件中
const canDelete = checkUserPrivileges(["delete_project"])

if (canDelete) {
  // 显示删除按钮
}
```

---

## 正则表达式

### 文件位置

```
src/utils/regexConstants.ts
```

### 常量清单

#### `EMAIL_REGEX`

**描述：** 邮箱验证正则表达式

**值：**
```typescript
const EMAIL_REGEX = /^[a-zA-Z0-9._-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,6}$/
```

**使用示例：**
```typescript
import { EMAIL_REGEX } from "@/utils/regexConstants"

const email = "test@example.com"
const isValid = EMAIL_REGEX.test(email)
// 输出: true
```

#### `PASSWORD_REGEX`

**描述：** 密码强度验证正则表达式

**值：**
```typescript
const PASSWORD_REGEX = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)[a-zA-Z\d@$!%*?&]{8,}$/
```

**使用示例：**
```typescript
import { PASSWORD_REGEX } from "@/utils/regexConstants"

const password = "Password123"
const isValid = PASSWORD_REGEX.test(password)
// 输出: true (包含大小写字母和数字，至少8位)
```

#### `URL_REGEX`

**描述：** URL验证正则表达式

**值：**
```typescript
const URL_REGEX = /^(https?:\/\/)?([\da-z.-]+)\.([a-z.]{2,6})([/\w .-]*)*\/?$/
```

**使用示例：**
```typescript
import { URL_REGEX } from "@/utils/regexConstants"

const url = "https://github.com/myrepo"
const isValid = URL_REGEX.test(url)
// 输出: true
```

---

## 扫描辅助

### 文件位置

```
src/utils/scanHelper.ts
```

### 函数清单

#### `getScanEngineLogo`

**描述：** 获取扫描引擎的Logo图标

**参数：**
```typescript
getScanEngineLogo(engine: string): string
```

**使用示例：**
```typescript
import { getScanEngineLogo } from "@/utils/scanHelper"

const sastLogo = getScanEngineLogo("sast")
// 输出: 引擎Logo的URL或类名
```

#### `getScanEngineTooltip`

**描述：** 获取扫描引擎的工具提示

**参数：**
```typescript
getScanEngineTooltip(engine: string): string
```

**使用示例：**
```typescript
import { getScanEngineTooltip } from "@/utils/scanHelper"

const sastTooltip = getScanEngineTooltip("sast")
// 输出: "Static Application Security Testing - 检测源代码中的安全漏洞"
```

#### `calculateScanProgress`

**描述：** 计算扫描进度百分比

**参数：**
```typescript
calculateScanProgress(scan: Scan): number
```

**使用示例：**
```typescript
import { calculateScanProgress } from "@/utils/scanHelper"

const scan = {
  status: "scanning",
  started_at: "2024-01-15T08:00:00Z",
  // ...其他属性
}

const progress = calculateScanProgress(scan)
// 输出: 45 (表示45%)
```

---

## 排序处理

### 文件位置

```
src/utils/sortChange.js
```

### 函数清单

#### `handleSortChange`

**描述：** 处理表格排序变化

**参数：**
```javascript
handleSortChange({ column, prop, order }, queryParams)
```

**使用示例：**
```javascript
import { handleSortChange } from "@/utils/sortChange"

// 在Element Plus表格组件中
@sort-change="(sort) => handleSortChange(sort, queryParams)"

// 函数会自动更新queryParams.order_by和queryParams.sort_order
```

---

## 第三方集成

### 文件位置

```
src/utils/thirdPartyIntegration.ts
```

### 函数清单

#### `initializeAnalytics`

**描述：** 初始化分析工具（如Segment）

**参数：**
```typescript
initializeAnalytics(): void
```

**使用示例：**
```typescript
import { initializeAnalytics } from "@/utils/thirdPartyIntegration"

// 在main.ts中初始化
createApp(App)
  .use(router)
  .use(pinia)
  .mount("#app")

// 初始化分析工具
initializeAnalytics()
```

#### `trackEvent`

**描述：** 跟踪用户事件

**参数：**
```typescript
trackEvent(eventName: string, properties?: Record<string, any>): void
```

**使用示例：**
```typescript
import { trackEvent } from "@/utils/thirdPartyIntegration"

// 用户点击扫描按钮
trackEvent("Scan Triggered", {
  project_id: 123,
  engines: ["sast", "sca"],
  user_id: 456
})

// 用户访问项目详情页
trackEvent("Project Viewed", { project_id: 123 })
```

---

## 树形数据结构

### 文件位置

```
src/utils/treeHelper.js
```

### 函数清单

#### `flattenTree`

**描述：** 将树形结构扁平化为数组

**参数：**
```javascript
flattenTree(tree, childrenKey = "children")
```

**使用示例：**
```javascript
import { flattenTree } from "@/utils/treeHelper"

const tree = [
  {
    id: 1,
    name: "Root",
    children: [
      { id: 2, name: "Child 1", children: [] },
      { id: 3, name: "Child 2", children: [] }
    ]
  }
]

const flatArray = flattenTree(tree)
// 输出: [
//   { id: 1, name: "Root", ... },
//   { id: 2, name: "Child 1", ... },
//   { id: 3, name: "Child 2", ... }
// ]
```

#### `findNodeInTree`

**描述：** 在树中查找指定节点

**参数：**
```javascript
findNodeInTree(tree, predicate, childrenKey = "children")
```

**使用示例：**
```javascript
import { findNodeInTree } from "@/utils/treeHelper"

const node = findNodeInTree(tree, (node) => node.id === 2)
// 输出: { id: 2, name: "Child 1", children: [] }
```

#### `buildTreeFromFlat`

**描述：** 从扁平数组构建树形结构

**参数：**
```javascript
buildTreeFromFlat(flatArray, idKey = "id", parentKey = "parent_id")
```

**使用示例：**
```javascript
import { buildTreeFromFlat } from "@/utils/treeHelper"

const flatArray = [
  { id: 1, name: "Root", parent_id: null },
  { id: 2, name: "Child 1", parent_id: 1 },
  { id: 3, name: "Child 2", parent_id: 1 }
]

const tree = buildTreeFromFlat(flatArray)
// 输出: 树形结构
```

---

## WebSocket工具

### 文件位置

```
src/utils/websocket.js
```

### 函数清单

#### `createWebSocket`

**描述：** 创建WebSocket连接

**参数：**
```javascript
createWebSocket(url, options)
```

**使用示例：**
```javascript
import { createWebSocket } from "@/utils/websocket"

const ws = createWebSocket("ws://localhost:8080/ws", {
  onMessage: (data) => {
    console.log("Received:", data)
  },
  onOpen: () => {
    console.log("WebSocket connected")
  },
  onClose: () => {
    console.log("WebSocket disconnected")
  }
})

// 发送消息
ws.send(JSON.stringify({ type: "ping" }))

// 关闭连接
ws.close()
```

#### `subscribeToScanUpdates`

**描述：** 订阅扫描状态更新

**参数：**
```javascript
subscribeToScanUpdates(scanId, callback)
```

**使用示例：**
```javascript
import { subscribeToScanUpdates } from "@/utils/websocket"

// 扫描开始
const unsubscribe = subscribeToScanUpdates(123, (status) => {
  console.log("Scan status:", status)
  // 更新UI
})

// 扫描结束或组件卸载时取消订阅
unsubscribe()
```

---

## 常量定义

### 文件位置

```
src/utils/constants.ts
```

### 常量清单

#### `SEVERITY_LEVELS`

**描述：** 漏洞严重级别定义

**值：**
```typescript
const SEVERITY_LEVELS = {
  CRITICAL: "Critical",
  HIGH: "High",
  MEDIUM: "Medium",
  LOW: "Low",
  INFO: "Info"
}
```

#### `SCAN_ENGINES`

**描述：** 扫描引擎类型

**值：**
```typescript
const SCAN_ENGINES = {
  SAST: "sast",
  SCA: "sca",
  LICENSE: "license",
  COMPLIANCE: "compliance"
}
```

#### `LICENSE_RISKS`

**描述：** 许可证风险级别

**值：**
```typescript
const LICENSE_RISKS = {
  LOW: "low",
  MEDIUM: "medium",
  HIGH: "high"
}
```

---

## 工具函数统计

### 按文件统计

| 文件 | 函数数量 | 主要功能 |
|------|----------|----------|
| constants.ts | 5+ | 常量定义 |
| dateHelper.ts | 3+ | 日期格式化 |
| helperFunctions.ts | 6+ | 通用助手函数 |
| imgHelper.ts | 2+ | 图片处理 |
| messageBoxes.ts | 2+ | 消息弹窗 |
| privilegesHelper.js | 2+ | 权限检查 |
| regexConstants.ts | 3+ | 正则表达式 |
| scanHelper.ts | 5+ | 扫描辅助 |
| sortChange.js | 1+ | 排序处理 |
| thirdPartyIntegration.ts | 2+ | 第三方集成 |
| treeHelper.js | 3+ | 树形数据操作 |
| websocket.js | 2+ | WebSocket连接 |

**总计：** 约36个常用工具函数

### 按类别统计

| 类别 | 数量 | 文件 |
|------|------|------|
| 常量定义 | 5 | constants.ts, regexConstants.ts |
| 数据处理 | 8 | helperFunctions.ts, treeHelper.js |
| 日期时间 | 3 | dateHelper.ts |
| 图片处理 | 2 | imgHelper.ts |
| 消息提示 | 2 | messageBoxes.ts |
| 权限验证 | 2 | privilegesHelper.js |
| 扫描相关 | 5 | scanHelper.ts |
| 第三方集成 | 2 | thirdPartyIntegration.ts |
| 网络通信 | 2 | websocket.js |
| 其他 | 5 | sortChange.js, formatter.ts |

---

## 使用建议

### 1. 优先使用现有工具函数

在开发新功能前，先检查是否已有现成的工具函数：

```bash
# 在utils/目录搜索相关功能
grep -r "formatDate" src/utils/
grep -r "tree" src/utils/
```

### 2. 添加新工具函数

如需添加新工具函数，请遵循以下规范：

1. **按功能分类**：添加到对应的文件中
2. **导出函数**：确保在文件末尾导出
3. **编写文档**：添加JSDoc注释
4. **补充示例**：提供使用示例
5. **更新本文档**：在本文档中登记

**示例：**

```typescript
// src/utils/newHelper.ts

/**
 * 新助手函数 - 功能说明
 * @param param1 - 参数1说明
 * @param param2 - 参数2说明
 * @returns 返回值说明
 *
 * @example
 * ```typescript
 * const result = newHelper('value1', 'value2')
 * ```
 */
export function newHelper(param1: string, param2: number): string {
  // 实现
  return "result"
}
```

### 3. 在Composables中使用

对于需要响应式的工具函数，优先使用Composables：

```typescript
// src/composables/newComposable.ts

import { ref, computed } from "vue"

export function useNewFeature() {
  const state = ref("")

  const computedValue = computed(() => {
    return state.value.toUpperCase()
  })

  function setState(value: string) {
    state.value = value
  }

  return {
    state,
    computedValue,
    setState
  }
}
```

---

## 常见问题

### Q1: 如何在组件中使用工具函数？

**A:** 直接导入使用

```vue
<script setup lang="ts">
import { formatISODateTime } from "@/utils/dateHelper"
import { showToast } from "@/utils/messageBoxes"

const date = ref("")
const displayDate = computed(() => formatISODateTime(date.value))

function handleSave() {
  showToast("保存成功", "success")
}
</script>
```

### Q2: 工具函数和Composables有什么区别？

**A:** 工具函数是纯函数，Composables可以包含响应式状态

```typescript
// 工具函数 - 纯函数
export function formatDate(date: string): string {
  return new Date(date).toLocaleString()
}

// Composables - 响应式
export function useDate() {
  const date = ref("")
  const formatted = computed(() => formatDate(date.value))

  return { date, formatted }
}
```

### Q3: 如何测试工具函数？

**A:** 使用Vitest或Jest编写单元测试

```typescript
// test/helperFunctions.test.ts
import { describe, it, expect } from "vitest"
import { generateColorBySeverity } from "@/utils/helperFunctions"

describe("generateColorBySeverity", () => {
  it("should return red for Critical", () => {
    expect(generateColorBySeverity("Critical")).toBe("#f12c2e")
  })

  it("should return orange for High", () => {
    expect(generateColorBySeverity("High")).toBe("#f3951b")
  })
})
```

---

## 扩展阅读

- [组件开发规范](./component_guide.md) - 如何在组件中使用工具函数
- [模块开发指南](./module_development_guide.md) - 工具函数在模块开发中的应用

---

## 最后更新

**最后更新日期：** 2025-11-27
**适用版本：** v4.10.0
**文档维护：** 新增工具函数时请同步更新本文档

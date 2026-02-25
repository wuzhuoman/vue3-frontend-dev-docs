# 修改密码弱口令检测功能

> **文档定位**: 本方案用于指导前端实现弱口令检测功能
> **适用版本**: vue3-frontend v4.10.0
> **创建日期**: 2026-02-25
> **需求类型**: 安全加固

---

## 📋 项目上下文（必读）

### 技术栈信息

- **框架**: Vue 3.4.15 + Vite 5.2.0 + TypeScript
- **状态管理**: Pinia
- **UI组件库**: Element Plus (自定义版本 v0.4.4-sct)
- **构建工具**: Vite

### 密码功能现状

项目当前有 **4个密码修改入口**，分别对应不同的业务场景：

| 入口         | 组件文件                       | 使用场景                  | 特点               |
| ------------ | ------------------------------ | ------------------------- | ------------------ |
| 用户下拉菜单 | `PasswordChangePopup.vue`      | 登录用户自主修改          | 需输入旧密码       |
| 强制修改弹窗 | `ForcePasswordChangeModal.vue` | 首次登录/密码过期强制修改 | 无法关闭，必须修改 |
| SCM账号重置  | `ResetPasswordModal.vue`       | 断开SCM集成时重置         | 无需旧密码         |
| 管理员重置   | `ResetPasswordModel.vue`       | 管理员重置成员密码        | 管理员操作         |

### 现有密码验证规则

当前密码验证仅检查复杂度（位于 `src/utils/constants.ts` 和 `src/utils/regexConstants.ts`）：

- 最少12个字符
- 至少1个大写字母
- 至少1个小写字母
- 至少1个数字
- 至少1个特殊字符 `- ! # % ^ & * ( ) _ + = . ? , @ | $`

### 涉及的核心文件清单

**新增文件**:

- `src/utils/weakPasswordCheck.ts` - 弱口令检测工具模块

**修改文件**:

- `src/views/layouts/content/PasswordChangePopup.vue` - 用户修改密码弹窗
- `src/components/auth/ForcePasswordChangeModal.vue` - 强制修改密码弹窗
- `src/views/login/ResetPasswordModal.vue` - SCM重置密码弹窗
- `src/views/setting/components/ResetPasswordModel.vue` - 管理员重置密码弹窗
- `src/language/cn.js` - 中文国际化
- `src/language/en.js` - 英文国际化

**参考文件**:

- `src/utils/constants.ts` - 现有密码规则常量
- `src/utils/regexConstants.ts` - 正则表达式常量
- `src/stores/user/actions.ts` - 用户状态管理
- `src/api/user.ts` - 用户API接口

### 项目全局依赖说明

**SHA256 函数**：

- **来源**：`/static/sha256/index.js`（在 `index.html` 中全局引入）
- **使用方式**：直接调用全局函数 `SHA256(string)`，无需导入
- **返回值**：字符串的 SHA256 哈希值（小写十六进制字符串）
- **示例**：`SHA256('password')` → `'5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8'`

---

## 📝 背景与问题描述

### 需求背景

当前系统密码修改功能仅检查密码复杂度（长度、大小写字母、数字、特殊字符），但无法识别与个人信息相关或常见的弱口令。这可能导致用户设置看似复杂但实际容易被猜测的密码，如 `ZhangSan@2024`、`Password123!` 等。

### 需求目标

在修改密码时增加弱口令识别功能，检测以下类型的弱口令：

1. **个人信息相关**：与姓名、身份证号相关
2. **重要日期相关**：与生日、纪念日相关
3. **常用词汇相关**：如 "password"、"123456"、"qwerty" 等
4. **内置弱口令列表**：维护常见弱口令字典

### 影响范围

- **涉及组件**：
  - [PasswordChangePopup.vue](../../src/views/layouts/content/PasswordChangePopup.vue) - 用户自主修改密码
  - [ForcePasswordChangeModal.vue](../../src/components/auth/ForcePasswordChangeModal.vue) - 强制修改密码
  - [ResetPasswordModal.vue](../../src/views/login/ResetPasswordModal.vue) - SCM账号重置密码
  - [ResetPasswordModel.vue](../../src/views/setting/components/ResetPasswordModel.vue) - 管理员重置成员密码
- **涉及API**：
  - [user.ts](../../src/api/user.ts) - `changePassword` 接口
  - [org.ts](../../src/api/org.ts) - `resetPassWord` 接口

### 优先级

**高** - 安全加固需求

---

## 🎯 解决方案

### 技术选型分析

#### 方案A：使用 zxcvbn 库（推荐）

**zxcvbn 简介**：

- 由 Dropbox 开发的开源密码强度评估库
- 通过模式匹配识别 30,000+ 常见密码、姓名、流行英语单词、日期、重复字符、键盘模式等
- 提供 0-4 的密码强度评分（0=非常弱，4=非常强）
- 返回详细的匹配模式和改进建议

**优点**：

- ✅ 功能完善，内置 3万+ 常见密码字典
- ✅ 能识别键盘模式（如 `qwerty`、`123456`）
- ✅ 能识别日期模式（如 `19900101`）
- ✅ 能识别重复字符和序列
- ✅ 支持 l33t speak 转换（如 `p@ssw0rd` → `password`）
- ✅ 社区活跃，文档完善

**缺点**：

- ❌ 包体积较大（约 800KB gzipped），主要因为内置字典
- ❌ 首次加载可能影响性能

**适用场景**：

- 对安全性要求较高的场景
- 可接受额外加载开销的场景

---

#### 方案B：使用 zxcvbn-lite 或 zxcvbn-ts

**zxcvbn-ts 简介**：

- zxcvbn 的 TypeScript 重写版本
- 支持按需加载字典，可减小包体积
- 提供更好的 TypeScript 支持

**优点**：

- ✅ 类型支持更好
- ✅ 可配置字典加载策略
- ✅ 体积可优化（按需加载）

**缺点**：

- ❌ 配置较复杂
- ❌ 社区活跃度不如原版

---

#### 方案C：自定义弱口令检测（推荐）

**实现思路**：

- 维护一个精简的弱口令列表（100-500条常见密码）
- 实现简单的模式检测（日期、连续字符、键盘模式）
- 结合用户个人信息进行匹配检测

**优点**：

- ✅ 包体积极小（< 5KB）
- ✅ 完全可控，可定制检测规则
- ✅ 无外部依赖
- ✅ 可根据业务需求灵活调整

**缺点**：

- ❌ 需要自行维护弱口令字典
- ❌ 检测能力不如 zxcvbn 全面

**适用场景**：

- 对包体积敏感的项目
- 需要高度定制化的检测规则

---

### 最终方案选择：**方案C（自定义弱口令检测）**

**选择理由**：

1. **包体积考虑**：项目使用 Vite 构建，对包体积敏感，zxcvbn 的 800KB 开销过大
2. **定制化需求**：需要检测中文姓名、身份证号等特定模式，zxcvbn 对中文支持有限
3. **维护可控性**：弱口令列表可根据国内安全规范自行维护
4. **性能考虑**：自定义实现无额外依赖，加载和执行性能更优

---

## 🔧 实施步骤

### 前置检查清单

在开始实施前，请确认以下文件可访问：

- [ ] `src/utils/constants.ts` - 检查现有密码规则
- [ ] `src/utils/regexConstants.ts` - 检查正则表达式
- [ ] `src/views/layouts/content/PasswordChangePopup.vue` - 确认组件结构
- [ ] `src/stores/user/state.ts` - 确认用户信息字段

### 步骤1：创建弱口令检测工具模块

**新建文件**：`src/utils/weakPasswordCheck.ts`

**文件作用**：提供弱口令检测的核心逻辑，被所有密码修改组件共用。

**注意事项**：

- 该文件为纯工具函数，不涉及UI渲染
- 所有检测函数不修改输入参数
- 返回结果包含可国际化的原因代码

```typescript
/**
 * 弱口令检测工具
 * 检测密码是否为弱口令（与个人信息、常见词汇、日期等相关）
 */

// 常见弱口令列表（可扩展）
export const COMMON_WEAK_PASSWORDS = [
  "password",
  "123456",
  "12345678",
  "qwerty",
  "abc123",
  "monkey",
  "letmein",
  "dragon",
  "111111",
  "baseball",
  "iloveyou",
  "trustno1",
  "sunshine",
  "princess",
  "admin",
  "welcome",
  "shadow",
  "ashley",
  "football",
  "jesus",
  "michael",
  "ninja",
  "mustang",
  "password1",
  "123456789",
  "adobe123",
  "admin123",
  "root",
  "toor",
  "guest",
  "default",
  "scantist",
  "catarc",
  "mstl",
  "anesec",
  "qwerty123",
  "1q2w3e4r",
  "zaq12wsx",
  "!@#$%^&*",
  "000000",
  "666666",
  "888888",
  "999999",
  "7777777",
];

// 键盘模式（横向）
const KEYBOARD_ROWS = ["qwertyuiop", "asdfghjkl", "zxcvbnm", "1234567890"];

// 键盘模式（横向反向）
const KEYBOARD_ROWS_REVERSE = KEYBOARD_ROWS.map((row) =>
  row.split("").reverse().join(""),
);

// 键盘模式（纵向）
const KEYBOARD_COLS = [
  "1qaz",
  "2wsx",
  "3edc",
  "4rfv",
  "5tgb",
  "6yhn",
  "7ujm",
  "8ik,",
  "9ol.",
  "0p;/",
];

// 键盘模式（纵向反向）
const KEYBOARD_COLS_REVERSE = KEYBOARD_COLS.map((col) =>
  col.split("").reverse().join(""),
);

export interface WeakPasswordCheckResult {
  isWeak: boolean;
  score: number; // 0-100，分数越低越弱
  reasons: string[]; // 弱口令原因列表
  suggestions: string[]; // 改进建议
}

/**
 * 用户信息接口
 * 基于 /user/ 接口实际返回数据：
 * {
 *   "id": 2674,
 *   "username": "wuqingyang",
 *   "email_info": { "email": "wuqingyang@catarc.ac.cn", "verified": false },
 *   "avatar_url": null,
 *   "default_org": 2833,
 *   "vcs_providers": [],
 *   "is_local_account": true
 * }
 *
 * 注意：email 字段需要从 email_info 对象中提取
 */
export interface UserInfo {
  username?: string; // 用户名
  email?: string; // 从 email_info.email 提取，如 wuqingyang@catarc.ac.cn
}

/**
 * 检查是否为常见弱口令
 */
function checkCommonWeakPassword(password: string): {
  isWeak: boolean;
  reason?: string;
} {
  const lowerPassword = password.toLowerCase();

  // 直接匹配
  if (COMMON_WEAK_PASSWORDS.includes(lowerPassword)) {
    return { isWeak: true, reason: "该密码为常见弱口令" };
  }

  // 包含常见弱口令
  for (const weakPwd of COMMON_WEAK_PASSWORDS) {
    if (lowerPassword.includes(weakPwd)) {
      return { isWeak: true, reason: `密码包含常见弱口令词汇: ${weakPwd}` };
    }
  }

  return { isWeak: false };
}

/**
 * 检查键盘模式
 * 检测横向、纵向以及反向的键盘连续字符序列
 */
function checkKeyboardPattern(password: string): {
  isWeak: boolean;
  reason?: string;
} {
  const lowerPassword = password.toLowerCase();

  // 检查横向键盘序列（连续3个及以上）
  for (const row of KEYBOARD_ROWS) {
    for (let i = 0; i <= row.length - 3; i++) {
      const pattern = row.slice(i, i + 3);
      if (lowerPassword.includes(pattern)) {
        return { isWeak: true, reason: "密码包含键盘连续字符模式" };
      }
    }
  }

  // 检查横向反向键盘序列（如 poiuyt、lkjhgf）
  for (const row of KEYBOARD_ROWS_REVERSE) {
    for (let i = 0; i <= row.length - 3; i++) {
      const pattern = row.slice(i, i + 3);
      if (lowerPassword.includes(pattern)) {
        return { isWeak: true, reason: "密码包含键盘反向连续字符模式" };
      }
    }
  }

  // 检查纵向键盘序列（连续3个及以上）
  for (const col of KEYBOARD_COLS) {
    for (let i = 0; i <= col.length - 3; i++) {
      const pattern = col.slice(i, i + 3);
      if (lowerPassword.includes(pattern)) {
        return { isWeak: true, reason: "密码包含键盘纵向连续字符模式" };
      }
    }
  }

  // 检查纵向反向键盘序列
  for (const col of KEYBOARD_COLS_REVERSE) {
    for (let i = 0; i <= col.length - 3; i++) {
      const pattern = col.slice(i, i + 3);
      if (lowerPassword.includes(pattern)) {
        return { isWeak: true, reason: "密码包含键盘纵向反向连续字符模式" };
      }
    }
  }

  return { isWeak: false };
}

/**
 * 检查日期模式
 * 检测各种日期格式，包括：
 * - 标准格式：YYYYMMDD (20240115)、MMDDYYYY (01152024)
 * - 宽松格式：YYYYMDD (2024115)、YYYYMMD (2024015) 等
 * - 分隔符格式：YYYY.MM.DD、YYYY-MM-DD、YYYY/MM/DD
 * - 中文格式：2024年01月15日、01月15日
 *
 * 注意：单独的年份数字（如 2024）不会被判定为日期
 */
function checkDatePattern(password: string): {
  isWeak: boolean;
  reason?: string;
} {
  // 匹配各种日期格式
  const datePatterns = [
    // 标准8位日期格式
    /(19|20)\d{2}(0[1-9]|1[0-2])(0[1-9]|[12]\d|3[01])/, // YYYYMMDD (如 20240115)
    /(0[1-9]|1[0-2])(0[1-9]|[12]\d|3[01])(19|20)\d{2}/, // MMDDYYYY (如 01152024)
    // 宽松日期格式（支持1-2位的月和日）
    /(19|20)\d{2}(0?[1-9]|1[0-2])(0?[1-9]|[12]\d|3[01])/, // YYYYMMDD 宽松 (如 2024111, 2024015)
    /(0?[1-9]|1[0-2])(0?[1-9]|[12]\d|3[01])(19|20)\d{2}/, // MMDDYYYY 宽松
    // 带分隔符的日期格式
    /(19|20)\d{2}[.\-/](0?[1-9]|1[0-2])[.\-/](0?[1-9]|[12]\d|3[01])/, // YYYY.MM.DD
    /(0?[1-9]|[12]\d|3[01])[.\-/](0?[1-9]|1[0-2])[.\-/](19|20)\d{2}/, // DD.MM.YYYY
    // 中文日期格式
    /\d{4}年\d{1,2}月\d{1,2}日/, // 中文年月日格式 (如 2024年1月15日, 2024年01月15日)
    /\d{1,2}月\d{1,2}日/, // 中文月日格式 (如 1月15日, 01月15日)
  ];

  for (const pattern of datePatterns) {
    if (pattern.test(password)) {
      return { isWeak: true, reason: "密码包含日期格式" };
    }
  }

  return { isWeak: false };
}

/**
 * 检查重复字符
 */
function checkRepeatedChars(password: string): {
  isWeak: boolean;
  reason?: string;
} {
  // 检查连续重复字符（3个及以上）
  const repeatedPattern = /(.)\1{2,}/;
  if (repeatedPattern.test(password)) {
    return { isWeak: true, reason: "密码包含连续重复字符" };
  }

  // 检查顺序序列（如 abc, 123, qwe）
  const sequences = [
    "abcdefghijklmnopqrstuvwxyz",
    "zyxwvutsrqponmlkjihgfedcba",
    "0123456789",
    "9876543210",
  ];

  const lowerPassword = password.toLowerCase();
  for (const seq of sequences) {
    for (let i = 0; i <= seq.length - 3; i++) {
      const pattern = seq.slice(i, i + 3);
      if (lowerPassword.includes(pattern)) {
        return { isWeak: true, reason: "密码包含顺序字符序列" };
      }
    }
  }

  return { isWeak: false };
}

/**
 * 检查与个人信息相关
 * 基于 /user/ 接口实际返回字段：username, email_info.email
 */
function checkPersonalInfo(
  password: string,
  userInfo: UserInfo,
): { isWeak: boolean; reason?: string } {
  const lowerPassword = password.toLowerCase();

  // 检查用户名（单向包含：密码包含用户名即为弱口令）
  if (userInfo.username) {
    const usernameLower = userInfo.username.toLowerCase();
    // 用户名长度>=3时才检测，避免过于短的用户名导致误报
    if (usernameLower.length >= 3 && lowerPassword.includes(usernameLower)) {
      return { isWeak: true, reason: "密码与用户名相关" };
    }
  }

  // 检查邮箱前缀
  if (userInfo.email) {
    const emailPrefix = userInfo.email.split("@")[0].toLowerCase();
    // 邮箱前缀长度>=3时才检测
    if (emailPrefix.length >= 3 && lowerPassword.includes(emailPrefix)) {
      return { isWeak: true, reason: "密码与邮箱相关" };
    }
  }

  return { isWeak: false };
}

/**
 * 综合弱口令检测
 */
export function checkWeakPassword(
  password: string,
  userInfo?: UserInfo,
): WeakPasswordCheckResult {
  const reasons: string[] = [];
  const suggestions: string[] = [];
  let score = 100;

  // 1. 检查常见弱口令
  const commonCheck = checkCommonWeakPassword(password);
  if (commonCheck.isWeak) {
    reasons.push(commonCheck.reason);
    score -= 40;
    suggestions.push("避免使用常见密码词汇");
  }

  // 2. 检查键盘模式
  const keyboardCheck = checkKeyboardPattern(password);
  if (keyboardCheck.isWeak) {
    reasons.push(keyboardCheck.reason);
    score -= 20;
    suggestions.push("避免使用键盘连续字符");
  }

  // 3. 检查日期模式
  const dateCheck = checkDatePattern(password);
  if (dateCheck.isWeak) {
    reasons.push(dateCheck.reason);
    score -= 20;
    suggestions.push("避免在密码中使用日期");
  }

  // 4. 检查重复字符
  const repeatedCheck = checkRepeatedChars(password);
  if (repeatedCheck.isWeak) {
    reasons.push(repeatedCheck.reason);
    score -= 15;
    suggestions.push("避免使用连续重复或顺序字符");
  }

  // 5. 检查个人信息（如果提供了）
  if (userInfo) {
    const personalCheck = checkPersonalInfo(password, userInfo);
    if (personalCheck.isWeak) {
      reasons.push(personalCheck.reason);
      score -= 30;
      suggestions.push("避免使用与个人信息相关的内容");
    }
  }

  // 确保分数在 0-100 范围内
  score = Math.max(0, Math.min(100, score));

  return {
    isWeak: score < 60 || reasons.length > 0,
    score,
    reasons,
    suggestions: [...new Set(suggestions)], // 去重
  };
}

export default checkWeakPassword;
```

---

### 步骤2：修改 PasswordChangePopup.vue 组件

**文件路径**：`src/views/layouts/content/PasswordChangePopup.vue`

**修改原因**：这是用户最常用的密码修改入口，位于顶部导航栏用户下拉菜单中。

**具体修改内容**：

#### 2.1 添加导入和状态

```typescript
// 在 script setup 顶部添加导入
import { checkWeakPassword } from "@/utils/weakPasswordCheck";
import { computed } from "vue"; // 确保已导入

// 添加响应式状态（在 passwordForm 定义附近）
const weakPasswordWarning = ref("");
const weakPasswordSuggestions = ref<string[]>([]);

// 获取用户信息 - 基于 /user/ 接口实际返回数据
const userStore = useUserStore();
const userInfo = computed(() => ({
  username: userStore.user?.username,
  email: userStore.user?.email_info?.email, // 注意：email 在 email_info 对象中
}));

// 防抖函数（如项目中已有 lodash-es，可直接使用 _.debounce）
function debounce<T extends (...args: any[]) => void>(
  func: T,
  wait: number,
): (...args: Parameters<T>) => void {
  let timeout: ReturnType<typeof setTimeout> | null = null;
  return function (...args: Parameters<T>) {
    if (timeout) clearTimeout(timeout);
    timeout = setTimeout(() => func(...args), wait);
  };
}
```

#### 2.2 添加密码监听（带防抖）

```typescript
// 在 watch(passwordForm, ...) 中添加或新建监听
// 注意：项目密码最小长度为12，与复杂度要求保持一致
// 使用防抖优化性能，避免每次输入都触发检测
const debouncedCheckWeakPassword = debounce((newVal: string) => {
  if (newVal.length >= 12) {
    const checkResult = checkWeakPassword(newVal, userInfo.value);
    if (checkResult.isWeak) {
      weakPasswordWarning.value = checkResult.reasons.join("；");
      weakPasswordSuggestions.value = checkResult.suggestions;
    } else {
      weakPasswordWarning.value = "";
      weakPasswordSuggestions.value = [];
    }
  } else {
    weakPasswordWarning.value = "";
    weakPasswordSuggestions.value = [];
  }
}, 300); // 300ms 防抖延迟

watch(
  () => passwordForm.newPassword,
  (newVal) => {
    debouncedCheckWeakPassword(newVal);
  },
);
```

#### 2.3 修改模板添加警告提示

```vue
<el-form-item :label="t('NEW_PASSWORD')" prop="newPassword">
  <el-input
    type="password"
    show-password
    autocomplete="off"
    v-model="passwordForm.newPassword"
    :placeholder="t('ENTER_NEW_PASSWORD')"
  />
  
  <!-- 弱口令警告 - 新增 -->
  <div v-if="weakPasswordWarning" class="weak-password-warning">
    <el-alert
      :title="weakPasswordWarning"
      type="warning"
      :description="weakPasswordSuggestions.join('；')"
      show-icon
      :closable="false"
      style="margin-top: 8px;"
    />
  </div>
  
  <!-- 原有密码强度检查UI -->
  <div class="password-change-popup__validation">
    ...
  </div>
</el-form-item>
```

#### 2.4 添加样式（可选）

```scss
.weak-password-warning {
  margin-top: 8px;
}
```

---

### 步骤3：添加表单验证规则（阻止弱口令提交）

**文件路径**：`src/views/layouts/content/PasswordChangePopup.vue`

**策略说明**：

- **硬性阻止**：弱口令检测将阻止表单提交，用户必须使用非弱口令才能修改密码
- **清晰提示**：当检测到弱口令时，会显示具体的拒绝原因（如"密码与用户名相关"、"密码包含键盘连续字符模式"等），帮助用户理解为什么密码被拒绝
- **改进建议**：同时提供改进建议，指导用户如何设置更安全的密码

```typescript
// 在 formRules 中添加验证器
const formRules = reactive<FormRules>({
  newPassword: [
    {
      required: true,
      message: t("The new password field is required."),
      trigger: "blur",
    },
    { validator: currentAndNewPasswordDifferent, trigger: "blur" },
    { validator: validateWeakPassword, trigger: "blur" }, // 新增弱口令验证
  ],
});

/**
 * 弱口令验证器 - 阻止提交
 *
 * 注意：此验证器会硬性阻止弱口令提交，确保系统安全性。
 * 当密码被检测为弱口令时，会显示具体的拒绝原因，帮助用户理解问题所在。
 */
const validateWeakPassword = (rule: any, value: any, callback: any) => {
  // 密码长度不足12时不检测（由其他规则处理）
  if (!value || value.length < 12) {
    callback();
    return;
  }
  const checkResult = checkWeakPassword(value, userInfo.value);
  if (checkResult.isWeak) {
    // 显示具体的拒绝原因和改进建议
    const errorMessage = checkResult.reasons.join("；");
    callback(new Error(errorMessage));
  } else {
    callback();
  }
};
```

---

### 步骤4：同步修改其他密码修改组件

#### 4.1 ForcePasswordChangeModal.vue

**文件路径**：`src/components/auth/ForcePasswordChangeModal.vue`

**修改说明**：

- 该组件用于强制用户修改密码（如首次登录或密码过期）
- 弹窗无法关闭，用户必须完成密码修改
- 修改方式与 PasswordChangePopup.vue 类似

**关键差异**：

- 该组件已经有一个密码强度检查区域（`password-strength-checker`）
- 建议将弱口令警告整合到现有检查区域中

---

#### 4.2 ResetPasswordModal.vue

**文件路径**：`src/views/login/ResetPasswordModal.vue`

**修改说明**：

- 用于SCM账号断开连接时设置新密码
- 无需输入旧密码
- 修改方式类似，但不需要 `currentAndNewPasswordDifferent` 验证

---

#### 4.3 ResetPasswordModel.vue（管理员重置密码）

**文件路径**：`src/views/setting/components/ResetPasswordModel.vue`

**修改说明**：

- 用于管理员重置组织成员密码
- 位于组织设置 → 成员管理页面
- 需要管理员权限才能访问
- 该组件通过 `props.rowItem` 接收用户信息，**仅包含 `id`, `user_id`, `username`**
- 由于无法获取被重置用户的邮箱等敏感信息，**仅进行通用弱口令检测**（常见密码、键盘模式等）

**重要：修复现有 Bug**

该组件的密码验证逻辑中**缺少小写字母检查**，这与项目密码规则不一致。在实施弱口令检测时，需要一并修复：

```typescript
// 在 validatePass 函数中添加小写字母检查
const validatePass = (rule: any, value: any, callback: any) => {
  const regCantainNumber = /\d/;
  const regCantainUppercase = /^.*[A-Z]+.*$/;

  if (value === "") {
    callback(new Error(t("MEMBER_PASSWORD_RULE")));
  } else if (value.length < 12) {
    callback(
      new Error(t("The password field must contain at least 12 characters")),
    );
  } else if (!regCantainNumber.test(value)) {
    callback(
      new Error(t("The password field must contain at least 1 numeric letter")),
    );
  } else if (!regCantainUppercase.test(value)) {
    callback(
      new Error(
        t("The password field must contain at least 1 upper case letter"),
      ),
    );
  } else if (!CONTAINS_LOWERCASE_REGEX.test(value)) {
    // 🔧 修复：添加小写字母检查
    callback(
      new Error(
        t("The password field must contain at least 1 lower case letter"),
      ),
    );
  } else if (!CONTAINS_SPECIAL_CHAR_REGEX.test(value)) {
    callback(
      new Error(
        t("The password field must contain at least 1 special character"),
      ),
    );
  } else {
    callback();
  }
  // ...
};

// 同时在 handleInput 函数中添加小写字母的 UI 状态更新
const handleInput = () => {
  const val = ResetPasswordForm.password;
  // ... 其他检查 ...

  // 检查是否包含小写字母
  if (CONTAINS_LOWERCASE_REGEX.test(val)) {
    isContainLowercase.value = true;
  } else {
    isContainLowercase.value = false;
  }
  // ...
};
```

**与其他组件的差异**：

```typescript
// 该组件仅接收有限的用户信息
interface IrowItem {
  id: string | number;
  user_id: string | number;
  username: string;
}

// 弱口令检测仅使用通用规则，不检测个人信息相关
const checkResult = checkWeakPassword(password, {
  username: props.rowItem?.username,
});
```

---

### 步骤5：添加国际化支持

**中文文件**：`src/language/cn.js`

在文件中找到合适的位置（建议在其他密码相关翻译附近）添加：

```javascript
// 在 cn.js 中添加 - 使用扁平化 key 风格，与项目保持一致
WEAK_PASSWORD_TITLE: '弱口令警告',
WEAK_PASSWORD_COMMON: '该密码为常见弱口令',
WEAK_PASSWORD_COMMON_CONTAINS: '密码包含常见弱口令词汇：%{word}',
WEAK_PASSWORD_KEYBOARD_PATTERN: '密码包含键盘连续字符模式',
WEAK_PASSWORD_KEYBOARD_PATTERN_REVERSE: '密码包含键盘反向连续字符模式',
WEAK_PASSWORD_KEYBOARD_PATTERN_COL: '密码包含键盘纵向连续字符模式',
WEAK_PASSWORD_KEYBOARD_PATTERN_COL_REVERSE: '密码包含键盘纵向反向连续字符模式',
WEAK_PASSWORD_DATE_PATTERN: '密码包含日期格式',
WEAK_PASSWORD_REPEATED_CHARS: '密码包含连续重复字符',
WEAK_PASSWORD_SEQUENCE_CHARS: '密码包含顺序字符序列',
WEAK_PASSWORD_RELATED_TO_USERNAME: '密码与用户名相关',
WEAK_PASSWORD_RELATED_TO_EMAIL: '密码与邮箱相关',
WEAK_PASSWORD_SUGGESTION_AVOID_COMMON: '避免使用常见密码词汇',
WEAK_PASSWORD_SUGGESTION_AVOID_KEYBOARD: '避免使用键盘连续字符',
WEAK_PASSWORD_SUGGESTION_AVOID_DATE: '避免在密码中使用日期',
WEAK_PASSWORD_SUGGESTION_AVOID_SEQUENCE: '避免使用连续重复或顺序字符',
WEAK_PASSWORD_SUGGESTION_AVOID_PERSONAL: '避免使用与个人信息相关的内容',
```

**英文文件**：`src/language/en.js`

```javascript
// 在 en.js 中添加 - 使用扁平化 key 风格，与项目保持一致
WEAK_PASSWORD_TITLE: 'Weak Password Warning',
WEAK_PASSWORD_COMMON: 'This is a commonly used weak password',
WEAK_PASSWORD_COMMON_CONTAINS: 'Password contains common weak password word: %{word}',
WEAK_PASSWORD_KEYBOARD_PATTERN: 'Password contains keyboard sequential pattern',
WEAK_PASSWORD_KEYBOARD_PATTERN_REVERSE: 'Password contains keyboard reverse sequential pattern',
WEAK_PASSWORD_KEYBOARD_PATTERN_COL: 'Password contains keyboard vertical sequential pattern',
WEAK_PASSWORD_KEYBOARD_PATTERN_COL_REVERSE: 'Password contains keyboard vertical reverse sequential pattern',
WEAK_PASSWORD_DATE_PATTERN: 'Password contains date format',
WEAK_PASSWORD_REPEATED_CHARS: 'Password contains repeated characters',
WEAK_PASSWORD_SEQUENCE_CHARS: 'Password contains sequential characters',
WEAK_PASSWORD_RELATED_TO_USERNAME: 'Password is related to username',
WEAK_PASSWORD_RELATED_TO_EMAIL: 'Password is related to email',
WEAK_PASSWORD_SUGGESTION_AVOID_COMMON: 'Avoid using common password words',
WEAK_PASSWORD_SUGGESTION_AVOID_KEYBOARD: 'Avoid using keyboard sequential characters',
WEAK_PASSWORD_SUGGESTION_AVOID_DATE: 'Avoid using dates in password',
WEAK_PASSWORD_SUGGESTION_AVOID_SEQUENCE: 'Avoid using repeated or sequential characters',
WEAK_PASSWORD_SUGGESTION_AVOID_PERSONAL: 'Avoid using personal information in password',
```

**使用方式**：

```typescript
// 在组件中使用
const { t } = i18n.global;
t("WEAK_PASSWORD_COMMON");
```

**weakPasswordCheck.ts 中使用国际化的示例**：

```typescript
import i18n from "@/language/lang";

const { t } = i18n.global;

function checkCommonWeakPassword(password: string): {
  isWeak: boolean;
  reason?: string;
} {
  const lowerPassword = password.toLowerCase();

  if (COMMON_WEAK_PASSWORDS.includes(lowerPassword)) {
    return { isWeak: true, reason: t("WEAK_PASSWORD_COMMON") };
  }

  for (const weakPwd of COMMON_WEAK_PASSWORDS) {
    if (lowerPassword.includes(weakPwd)) {
      return {
        isWeak: true,
        reason: t("WEAK_PASSWORD_COMMON_CONTAINS", { word: weakPwd }),
      };
    }
  }

  return { isWeak: false };
}
```

---

### 影响评估

#### 影响的模块/组件

| 组件                         | 影响类型 | 说明                 |
| ---------------------------- | -------- | -------------------- |
| PasswordChangePopup.vue      | 修改     | 添加弱口令检测和提示 |
| ForcePasswordChangeModal.vue | 修改     | 添加弱口令检测和提示 |
| ResetPasswordModal.vue       | 修改     | 添加弱口令检测和提示 |
| ResetPasswordModel.vue       | 修改     | 添加弱口令检测和提示 |
| weakPasswordCheck.ts         | 新增     | 弱口令检测工具模块   |

#### 多品牌兼容性

- ✅ 无品牌特定代码，所有品牌通用
- ✅ 国际化支持完善

#### 性能影响

- ✅ 弱口令检测为纯前端计算，无额外网络请求
- ✅ 检测逻辑简单，执行耗时 < 1ms
- ✅ 新增代码体积约 5KB（gzip 后约 1.5KB）

---

## ✅ 验证计划

### 功能测试用例

#### 弱口令检测功能

| 测试项     | 测试输入                                    | 预期结果                               |
| ---------- | ------------------------------------------- | -------------------------------------- |
| 常见弱口令 | `password`                                  | 提示"该密码为常见弱口令"               |
| 常见弱口令 | `123456`                                    | 提示"该密码为常见弱口令"               |
| 常见弱口令 | `MyPassword123!`                            | 提示"密码包含常见弱口令词汇"           |
| 键盘模式   | `qwerty123!`                                | 提示"密码包含键盘连续字符模式"         |
| 键盘模式   | `poiuyt123!`                                | 提示"密码包含键盘反向连续字符模式"     |
| 键盘模式   | `1qaz2wsx!`                                 | 提示"密码包含键盘纵向连续字符模式"     |
| 键盘模式   | `zaq12wsx!`                                 | 提示"密码包含键盘纵向反向连续字符模式" |
| 日期模式   | `19900101Ab!`                               | 提示"密码包含日期格式"                 |
| 日期模式   | `2024115Ab!`                                | 提示"密码包含日期格式"（宽松格式）     |
| 日期模式   | `2024.01.15Ab!`                             | 提示"密码包含日期格式"（分隔符格式）   |
| 日期模式   | `2024年1月15日Ab!`                          | 提示"密码包含日期格式"（中文格式）     |
| 日期模式   | `My2024Ab!`                                 | **不提示**（单独年份不视为日期）       |
| 重复字符   | `aaaBbb123!`                                | 提示"密码包含连续重复字符"             |
| 顺序序列   | `abc123Def!`                                | 提示"密码包含顺序字符序列"             |
| 顺序序列   | `cba123Def!`                                | 提示"密码包含顺序字符序列"（反向）     |
| 用户名相关 | 用户名`zhangsan`，密码`zhangsan2024!`       | 提示"密码与用户名相关"                 |
| 邮箱相关   | 邮箱`zhangsan@company.com`，密码`zhangsan!` | 提示"密码与邮箱相关"                   |
| 邮箱相关   | 邮箱`ab@company.com`，密码`ab123!`          | **不提示**（短邮箱前缀不检测）         |

#### 回归测试

| 测试项         | 验证内容                                 |
| -------------- | ---------------------------------------- |
| 原有密码复杂度 | 密码`Short1!`（8位）应提示长度不足       |
| 原有密码复杂度 | 密码`nouppercase123!` 应提示缺少大写字母 |
| 原有密码复杂度 | 密码`NOLOWERCASE123!` 应提示缺少小写字母 |
| 原有密码复杂度 | 密码`NoNumbers!@#` 应提示缺少数字        |
| 原有密码复杂度 | 密码`NoSpecial123` 应提示缺少特殊字符    |
| 密码修改流程   | 正常密码修改流程应成功                   |
| 表单验证       | 表单验证错误提示应正常显示               |

### 多品牌环境测试

- [ ] **CATARC** - 验证弱口令检测正常显示
- [ ] **MSTL** - 验证弱口令检测正常显示
- [ ] **ANESEC** - 验证弱口令检测正常显示
- [ ] **OSREDM** - 验证弱口令检测正常显示
- [ ] **DEFAULT** - 验证弱口令检测正常显示

### 浏览器兼容性测试

- [ ] Chrome
- [ ] Firefox
- [ ] Edge
- [ ] Safari（如支持）

---

## � 相关文件速查

### 本方案涉及的所有文件

| 类型     | 文件路径                                              | 操作 | 说明               |
| -------- | ----------------------------------------------------- | ---- | ------------------ |
| **新增** | `src/utils/weakPasswordCheck.ts`                      | 创建 | 弱口令检测工具模块 |
| **修改** | `src/views/layouts/content/PasswordChangePopup.vue`   | 编辑 | 用户修改密码弹窗   |
| **修改** | `src/components/auth/ForcePasswordChangeModal.vue`    | 编辑 | 强制修改密码弹窗   |
| **修改** | `src/views/login/ResetPasswordModal.vue`              | 编辑 | SCM重置密码弹窗    |
| **修改** | `src/views/setting/components/ResetPasswordModel.vue` | 编辑 | 管理员重置密码弹窗 |
| **修改** | `src/language/cn.js`                                  | 编辑 | 中文国际化         |
| **修改** | `src/language/en.js`                                  | 编辑 | 英文国际化         |

### 参考文件（只读）

| 文件路径                      | 用途                                                    |
| ----------------------------- | ------------------------------------------------------- |
| `src/utils/constants.ts`      | 现有密码规则常量（passwordChecks, SPECIAL_CHAR_STRING） |
| `src/utils/regexConstants.ts` | 正则表达式常量（CONTAINS_NUM_REGEX 等）                 |
| `src/stores/user/state.ts`    | 用户信息状态定义                                        |
| `src/stores/user/actions.ts`  | 用户状态管理 actions                                    |
| `src/api/user.ts`             | 用户API接口（changePassword）                           |
| `src/api/org.ts`              | 组织API接口（resetPassWord）                            |

---

## �� 参考资料

### 第三方库参考

- [zxcvbn GitHub](https://github.com/dropbox/zxcvbn) - Dropbox 密码强度评估库
- [zxcvbn-ts GitHub](https://github.com/zxcvbn-ts/zxcvbn) - TypeScript 版本

### 安全规范参考

- [常见弱口令列表 - SecLists](https://github.com/danielmiessler/SecLists/tree/master/Passwords)
- [中国网络安全等级保护 2.0 密码要求](http://www.cac.gov.cn/)
- [OWASP 密码安全指南](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)

### 项目内部文档

- [API层文档](../../api_layer.md)
- [模块开发规范](../../module_development_guide.md)
- [多品牌配置系统](../../multibrand_config.md)

---

## 📝 备注

### 弱口令字典维护建议

1. **定期更新**：建议每季度更新一次常见弱口令列表
2. **来源渠道**：
   - 公开的弱口令字典（如 SecLists）
   - 安全厂商发布的年度弱口令报告
   - 内部安全审计发现的弱口令
3. **自定义扩展**：可根据业务特点添加行业特定弱口令

### 后续优化方向

1. **后端联动**：可考虑后端也进行弱口令检测，防止绕过前端
2. **泄露检测**：集成 Have I Been Pwned API 检测密码是否已泄露
3. **智能提示**：根据检测结果提供更智能的密码改进建议

---

## ❓ 常见问题

### Q1: 为什么不用 zxcvbn 库？

**A**: zxcvbn 库虽然功能强大，但包体积约 800KB，对前端项目负担过重。本项目采用自定义实现，体积仅约 5KB，且更易于定制中文相关检测规则。

### Q2: 弱口令检测是否阻止表单提交？

**A**: 是的，本方案采用**硬性阻止**策略。当密码被检测为弱口令时，表单验证会失败，用户无法提交密码修改请求。同时，系统会显示具体的拒绝原因（如"密码与用户名相关"、"密码包含键盘连续字符模式"等），帮助用户理解问题所在并改进密码。

### Q3: 如何添加新的弱口令？

**A**: 编辑 `src/utils/weakPasswordCheck.ts` 中的 `COMMON_WEAK_PASSWORDS` 数组，添加新的弱口令字符串即可。

### Q4: 个人信息检测需要哪些字段？

**A**: 当前支持检测的字段包括：username（用户名）、email（邮箱）。根据实际业务需要，可在 `UserInfo` 接口中添加更多字段。

### Q5: 密码中包含年份（如 2024）会被判定为弱口令吗？

**A**: 不会。日期检测只匹配完整的日期格式（如 `20240115`、`2024年01月15日`），单独的4位年份数字不会被判定为日期。这样可以避免误判像 `MyCompany@2024` 这样的合理密码。

### Q6: 多品牌环境是否需要特殊处理？

**A**: 不需要。弱口令检测逻辑对所有品牌通用，国际化文本已通过 `src/language/` 下的文件支持多语言。

# 表单验证示例

> **基于代码版本**: v4.10.0
> **最后验证时间**: 2025-11-27 20:00
> **代码文件来源**:
> - `ForgetPasswordModal.vue` (220行) ✅
> - `CreateProjectModal.vue` (678行) ✅
> - `ResetPasswordModal.vue` (300行) ✅
> - `LoginPage.vue` (1190行) ✅
> - `regexConstants.ts` (38行) ✅
> - `constants.ts` (密码验证配置) ✅

⚠️ **重要提示**: 本文档完全基于实际项目代码，所有验证规则和示例均来自真实组件。

## 目录

- [概述](#概述)
- [Element Plus验证基础](#element-plus验证基础)
- [验证示例1: 邮箱验证](#验证示例1-邮箱验证)
- [验证示例2: 密码验证](#验证示例2-密码验证)
- [验证示例3: 密码复杂性检查](#验证示例3-密码复杂性检查)
- [验证示例4: 项目创建表单](#验证示例4-项目创建表单)
- [验证示例5: 自定义文件验证](#验证示例5-自定义文件验证)
- [验证示例6: 多邮箱验证](#验证示例6-多邮箱验证)
- [常用正则表达式](#常用正则表达式)
- [验证最佳实践](#验证最佳实践)
- [常见问题](#常见问题)

---

## 概述

项目使用 **Element Plus** 的表单验证系统，支持同步和异步验证、自定义验证器、字段间联动验证。

**验证核心文件**:
- `FormInstance` - 表单实例类型
- `FormRules` - 验证规则类型
- `rules` 属性 - 绑定规则
- `validate()` 方法 - 手动触发验证
- `resetFields()` 方法 - 重置表单和验证

---

## Element Plus验证基础

### 标准规则结构

```typescript
// 基础规则类型
const rules = reactive<FormRules>({
  fieldName: [
    { required: boolean, message: string, trigger: string },
    { min: number, max: number, message: string, trigger: string },
    { type: "email", message: string, trigger: string },
    { validator: Function, trigger: string }
  ]
})
```

**触发方式**:
- `"blur"` - 失焦时触发
- `"change"` - 值变化时触发
- `["blur", "change"]` - 两种方式都触发

**规则优先级**: 从上到下依次验证，前一个验证失败则不再继续。

---

### 在模板中使用

```vue
<el-form
  ref="ruleFormRef"
  :model="form"
  :rules="rules"
  label-width="120px"
  @submit.native.prevent
>
  <el-form-item label="邮箱" prop="email">
    <el-input v-model="form.email" />
  </el-form-item>
</el-form>
```

---

### 在JavaScript中使用

```typescript
// 1. 提交时验证
await formEl.validate((valid, fields) => {
  if (valid) {
    // 验证通过，提交表单
    submitForm()
  } else {
    // 验证失败，fields包含错误信息
    console.log("error submit!", fields)
  }
})

// 2. 验证单个字段
await formEl.validateField("email")

// 3. 重置表单
formEl.resetFields()
```

---

## 验证示例1: 邮箱验证

**文件**: ForgetPasswordModal.vue (105-118行)

### 使用自定义验证器

```typescript
const form = reactive({
  email: "",
})

const validateEmail = (rule: any, value: any, callback: any) => {
  const regEmail = /\w[-\w.+]*@([A-Za-z0-9][-A-Za-z0-9]+\.)+[A-Za-z]{2,14}/

  if (value === "") {
    callback(new Error(t("MEMBER_EMAIL_RULE")))  // "请输入邮箱"
  } else if (!regEmail.test(value)) {
    callback(new Error(t("The invitee field must be a valid email.")))  // "请输入有效的邮箱地址"
  } else {
    callback()  // 验证通过
  }
}

const rules = reactive<FormRules>({
  email: [
    {
      required: true,
      validator: validateEmail,  // 使用自定义验证器
      trigger: "blur"
    }
  ]
})
```

**正则表达式详解**:
```regex
/\w[-\w.+]*@([A-Za-z0-9][-A-Za-z0-9]+\.)+[A-Za-z]{2,14}/
├─ \w                   # 开头字母/数字/下划线
├─ [-\w.+]*            # 中间字符（允许- . +）
├─ @                    # @符号
├─ ([A-Za-z0-9]        # 域名开头
├─ [-A-Za-z0-9]+       # 域名主体
├─ \.)+                # 点号（一个或多个子域名）
└─ [A-Za-z]{2,14}      # 顶级域名（2-14个字母）
```

---

### 使用内置email类型验证

**文件**: SignUp.vue (简化示例)

```typescript
const rules = {
  email: [
    { required: true, message: "请输入邮箱", trigger: "blur" },
    { type: "email", message: "请输入有效的邮箱", trigger: "blur" }
  ]
}
```

**对比**:
- **内置email**: Element Plus的标准邮箱格式（较宽松）
- **自定义验证器**: 项目特定的严格邮箱格式

**建议**: 对安全性要求高的场景使用自定义验证器。

---

## 验证示例2: 密码验证

### 基础密码验证

**文件**: LoginPage.vue (475-481行)

```typescript
const validatePass = (rule: any, value: any, callback: any) => {
  if (value === "") {
    callback(new Error(t("PASSWORD_FIELD_REQUIRED")))  // "请输入密码"
  } else {
    callback()  // 任何非空密码都通过
  }
}

const rules = reactive<FormRules>({
  password: [
    { required: true, validator: validatePass, trigger: "blur" }
  ]
})
```

**特点**: 登录表单只验证密码不为空，不检查复杂度。

---

### 复杂密码验证

**文件**: ResetPasswordModal.vue (119-135行)

```typescript
const newAndConfirmPasswordDifferent = (rule: any, value: any, callback: any) => {
  if (value !== passwordForm.newPassword) {
    callback(
      new Error(t("The repeat password confirmation does not match."))
    )
  }
}

const formRules = reactive<FormRules>({
  newPassword: [
    {
      required: true,
      message: "The new password field is required.",
      trigger: "blur"
    }
  ],
  confirmNewPassword: [
    {
      required: true,
      message: "The repeat password field is required.",
      trigger: "blur"
    },
    { validator: newAndConfirmPasswordDifferent, trigger: "blur" }
  ]
})
```

**表单数据**:
```typescript
const passwordForm = reactive({
  newPassword: "",
  confirmNewPassword: ""
})
```

**注意**: `confirmNewPassword` 必须与 `newPassword` 完全匹配。

---

## 验证示例3: 密码复杂性检查

**文件**: ResetPasswordModal.vue (252-272行)

### 基于正则验证的功能特点

该项目实现了一个**实时密码强度检测**系统，使用多个正则表达式验证不同的规则。

#### 密码验证规则

```typescript
// 在 constants.ts 中定义
// passwordChecks: 密码验证检查列表
export const passwordChecks = [
  { text: "MIN_12_CHARACTERS", isSatisfied: false },
  { text: "AT_LEAST_1_NUM", isSatisfied: false },
  { text: "AT_LEAST_1_UPPERCASE_LETTER", isSatisfied: false },
  { text: "AT_LEAST_1_LOWERCASE_LETTER", isSatisfied: false },
  { text: "AT_LEAST_1_SPECIAL_CHAR", isSatisfied: false }
]

// 用于修改密码时，增加对旧密码的验证
export const resetPasswordChecks = [
  ...密码验证检查列表,
  { text: "OLD_AND_NEW_PASSWORD_DIFFERENT", isSatisfied: false }
]
```

**具体的验证实现如下**: 在 ResetPasswordModal.vue 的 `updatePasswordCheckValues()` 方法中 (252-272行)：

```typescript
function updatePasswordCheckValues() {
  // 1. 检查密码长度
  newPasswordRules[0].isSatisfied = passwordForm.newPassword.length >= 12

  // 2. 检查包含数字
  newPasswordRules[1].isSatisfied = CONTAINS_NUM_REGEX.test(
    passwordForm.newPassword
  )

  // 3. 检查包含大写字母
  newPasswordRules[2].isSatisfied = CONTAINS_UPPERCASE_REGEX.test(
    passwordForm.newPassword
  )

  // 4. 检查包含小写字母
  newPasswordRules[3].isSatisfied = CONTAINS_LOWERCASE_REGEX.test(
    passwordForm.newPassword
  )

  // 5. 检查包含特殊字符
  newPasswordRules[4].isSatisfied = CONTAINS_SPECIAL_CHAR_REGEX.test(
    passwordForm.newPassword
  )

  // 6. 确认密码匹配
  changePasswordRules[newPasswordRules.length].isSatisfied =
    passwordForm.newPassword === passwordForm.confirmNewPassword

  // 7. 所有规则必须满足
  passwordCheckFailed.value = !changePasswordRules.every(
    rule => rule.isSatisfied
  )
}
```

**相关正则常量**: 来自 `regexConstants.ts` (第22-29行)

```typescript
// 检查是否包含数字
export const CONTAINS_NUM_REGEX = /\d/

// 检查是否包含大写字母
export const CONTAINS_UPPERCASE_REGEX = /^.*[A-Z]+.*$/

// 检查是否包含小写字母
export const CONTAINS_LOWERCASE_REGEX = /^.*[a-z]+.*$/

// 检查是否包含特殊字符
export const CONTAINS_SPECIAL_CHAR_REGEX = /[-!@#$%^&*()_+=.?,\\^$|]/
```

**注意**: Element Plus 的密码输入框使用了 `show-password` 属性，可显示密码以检查规则：

```vue
<el-input
  type="password"
  show-password
  v-model="passwordForm.newPassword"
/>
```

---

### 在模板中显示验证状态

**文件**: ResetPasswordModal.vue (27-42行)

```vue
<el-form-item>
  <div class="password-change-popup__validation">
    <div v-for="check in newPasswordRules" :key="check.text">
      <el-txt
        type="body2"
        :class="{
          'password-change-popup__check-color': check.isSatisfied
        }"
        class="password-change-popup__hint-style"
      >
        <i style="margin-right: 5px" class="fa fa-check"></i>
        <span>{{ t(check.text) }}</span>
      </el-txt>
    </div>
  </div>
</el-form-item>
```

**效果**: 用户输入时，满足的条件会变为绿色，实时反馈密码强度。

---

## 验证示例4: 项目创建表单

**文件**: CreateProjectModal.vue (298-338行)

### 多字段验证组合

```typescript
const rules = reactive<FormRules>({
  // 1. 项目名称 - 必填
  name: [
    {
      required: true,
      message: t("PLEASE_PROVIDE_A_PROJECT_NAME"),  // "请输入项目名称"
      trigger: "blur"
    }
  ],

  // 2. 团队选择 - 必填，change触发
  team_id_list: [
    {
      required: true,
      message: t("SELECT_A_TEAM"),  // "请选择团队"
      trigger: "change"
    }
  ],

  // 3. 文件列表 - 自定义验证器
  fileList: [
    {
      required: false,
      validator: fileListChange,
      trigger: "change"
    }
  ],

  // 4. 版本号 - 必填+长度限制
  version: [
    {
      required: true,
      validator: versionChange,
      trigger: "blur"
    },
    {
      min: 1,
      max: 50,
      trigger: "change",
      message: t("ENTER_UP_TO_CHARACTERS", { count: 50 })
      // "最多输入50个字符"
    }
  ],

  // 5. SBOM格式 - 选择时blur/change都触发
  file_type: [
    {
      required: true,
      message: t("PLEASE_SELECT_SBOM_FORMAT"),  // "请选择SBOM格式"
      trigger: ["blur", "change"]
    }
  ]
})
```

**表单数据模型**:

```typescript
const form = reactive({
  name: "",
  provider: "upload",
  version: "",
  file_type: "",
  team_id_list: [],
  compliance_policy_id: "",
  organization_tag_id_list: [],
  description: "",
  fileList: <UploadUserFile[]>[],
})
```

---

### 自定义验证器: 版本号验证

```typescript
const versionChange = (rule: any, value: any, callback: any) => {
  if (form.fileList.length) {
    // 如果有上传文件，版本号为必填
    if (value === "") {
      callback(new Error(t("PLEASE_PROVIDE_A_VERSION_NAME")))
    } else {
      callback()
    }
  } else {
    // 没有上传文件，版本号可选
    callback()
  }
}
```

**特点**: 条件验证，只有上传文件时才要求填写版本号。

---

### 自定义验证器: 文件大小验证

```typescript
const fileListChange = (rule: any, value: any, callback: any) => {
  if (orgSubscriptionUsage.value.upload_size_limit == null) {
    // 没有限制
    callback()
  } else if (
    uploadData.file_size > orgSubscriptionUsage.value.upload_size_limit
  ) {
    // 超出限制
    callback(
      new Error(
        t("FILE_SIZE_ERROR", {
          size: `：${orgSubscriptionUsage.value.upload_size_limit}MB`,
        })
      )
    )
  } else {
    // 在限制内
    callback()
  }
}
```

**验证触发**: `on-change` 事件时触发，文件选择后立即验证。

---

### 文件上传组件事件处理

```typescript
const handleChange = async (uploadFile, formEl: FormInstance | undefined) => {
  const sizeBytes = uploadFile?.size ?? 0

  // 0KB 文件拦截
  if (sizeBytes === 0) {
    ElMessage({
      showClose: true,
      message: t("FILE_EMPTY"),
      type: "error",
    })
    // 清空已选择的文件
    uploadRef.value?.clearFiles()
    form.fileList.splice(0)
    // 重置上传数据
    uploadData.filename = ""
    uploadData.file_size = 0
    uploadData.file_modified = ""
    // 重新验证fileList字段
    if (formEl) {
      await formEl.validateField("fileList")
    }
    return
  }

  // 设置文件数据
  uploadData.filename = uploadFile.name
  uploadData.file_size = sizeBytes / (1024 * 1024)  // 转换为MB
  uploadData.file_modified = now()

  // 验证文件大小
  if (
    orgSubscriptionUsage.value.upload_size_limit != null &&
    uploadData.file_size > orgSubscriptionUsage.value.upload_size_limit
  ) {
    exceedSizeLimit.value = true
  } else {
    exceedSizeLimit.value = false
  }

  // 立即验证fileList字段
  if (formEl) {
    await formEl.validateField("fileList")
  }
}
```

**流程**:
1. 文件选择后触发 `on-change`
2. 检查文件是否为空 (0KB)，是则清空并显示错误
3. 提取文件名、大小、修改时间
4. 验证文件大小是否超出限制
5. 立即触发Element Plus的验证

---

## 验证示例5: 自定义文件验证

**文件**: CreateProjectModal.vue (490-521行)

### 文件名长度验证

```typescript
const exceedFilename = computed(() => {
  if (form.fileList.length == 0) {
    return false
  }
  return uploadData.filename.length >= 200
})
```

### 文件名包含空格验证

```typescript
const fileNameConsistsOfSpace = computed(() => {
  let spaceFound = false
  if (form.fileList.length == 0) {
    return false
  }

  form.fileList.every((file) => {
    if (file.name.indexOf(" ") >= 0) {
      spaceFound = true
    }
    return spaceFound
  })
  return spaceFound
})
```

### 组合验证

```typescript
const exceedFilenameAndConsistSpace = computed(() => {
  if (form.fileList.length == 0) {
    return false
  }
  return exceedFilename.value && fileNameConsistsOfSpace.value
})
```

**警告显示 (Vue模板)**:

```vue
<el-form-item v-if="exceedFilenameAndConsistSpace">
  <WarnTriangleFilled style="width: 16px; color: var(--el-color-danger)" />
  <small style="color: var(--el-color-danger)">
    {{ t("FILE_NAME_CONSIST_SPACE_AND_EXCEED_50") }}
  </small>
</el-form-item>

<el-form-item v-else-if="exceedFilename">
  <WarnTriangleFilled style="width: 16px; color: var(--el-color-danger)" />
  <small style="color: var(--el-color-danger)">
    {{ t("FILE_NAME_EXCEED_200") }}
  </small>
</el-form-item>

<el-form-item v-else-if="fileNameConsistsOfSpace">
  <WarnTriangleFilled style="width: 16px; color: var(--el-color-danger)" />
  <small style="color: var(--el-color-danger)">
    {{ t("FILE_NAME_CONSIST_SPACE") }}
  </small>
</el-form-item>
```

**验证范围**:
- `v-if` / `v-else-if` 链确保只显示一种警告
- 计算属性实时计算，文件变化时立即更新

---

## 验证示例6: 多邮箱验证

**文件**: InviteMemberModel.vue (根据验证报告)

### 多个邮箱地址验证

```typescript
// 验证多个邮箱，以逗号分隔
const validateMultiEmail = (rule: any, value: any, callback: any) => {
  if (!value) {
    callback(new Error("请输入邮箱"))
    return
  }

  // 分割多个邮箱
  const emails = value.split(",").map(email => email.trim())

  // 邮箱正则
  const emailRegex = /^[a-zA-Z0-9._-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/

  // 验证每个邮箱
  for (const email of emails) {
    if (!emailRegex.test(email)) {
      callback(new Error(`"${email}" 不是有效的邮箱地址`))
      return
    }
  }

  callback()
}

const rules = {
  email: [
    { required: true, validator: validateMultiEmail, trigger: "blur" }
  ]
}
```

**支持格式**:
- `"user1@example.com"`
- `"user1@example.com, user2@example.com"`
- `"user1@example.com,user2@example.com,user3@example.com"`

---

## 常用正则表达式

**文件**: `src/utils/regexConstants.ts`

### 1. 邮箱验证

```typescript
// 标准邮箱正则
export const VALID_EMAIL_REGEX =
  /^[a-zA-Z0-9._%+-]+@(?:[a-zA-Z0-9-]+\.)+[a-zA-Z]{2,14}$/

// ForgetPassword中的自定义邮箱正则
/\w[-\w.+]*@([A-Za-z0-9][-A-Za-z0-9]+\.)+[A-Za-z]{2,14}/
```

**区别**: `VALID_EMAIL_REGEX` 支持 `%` `+` 等特殊字符（更宽松）。

---

### 2. 用户名验证

```typescript
export const VALID_USERNAME_REGEX = /^[a-zA-Z0-9@.+\-_]*$/
```

**允许字符**: 字母、数字、@、.、+、-、_

---

### 3. 密码复杂性验证

```typescript
// 包含数字
export const CONTAINS_NUM_REGEX = /\d/

// 包含大写字母
export const CONTAINS_UPPERCASE_REGEX = /^.*[A-Z]+.*$/

// 包含小写字母
export const CONTAINS_LOWERCASE_REGEX = /^.*[a-z]+.*$/

// 包含特殊字符
export const CONTAINS_SPECIAL_CHAR_REGEX = /[-!@#$%^&*()_+=.?,\\^$|]/
```

---

### 4. 电话号码验证

```typescript
// 中国手机号
export const VALID_CN_PHONE_NUMBER = /^1[3456789]\d{9}$/

// 国际电话号码（带区号）
/^[+]?[(]?[0-9]{1,4}[)]?[-\s./0-9]*$/
```

**中国手机号规则**:
- 1开头
- 第2位: 3-9（排除0,1,2）
- 总共11位数字

---

### 5. URL验证

```typescript
export const VALID_URL = /^(https?):\/\/[^\s/$.?#].[^\s]*$/
```

**支持**:
- http:// 或 https://
- 域名或IP地址
- 端口号、路径、查询参数

---

### 6. 项目名称验证

```typescript
export const PROJECT_NAME_REGEX = /^[\u4e00-\u9fffa-zA-Z0-9.,\-,_\s]{1,200}$/
```

**允许**:
- 中文、英文、数字
- 标点: . , - _ ,
- 空格
- 长度: 1-200字符

---

### 7. 标签验证

```typescript
export const TAG_REGEX = /^[a-zA-Z0-9一-\u9FFF_\- ]{2,10}$/
```

**规则**:
- 长度: 2-10字符
- 允许: 中英文、数字、_ - 、空格

---

## 验证最佳实践

### 1. 小组件内验证

**推荐**: 在组件内部使用完整的表单验证流程，不依赖外部状态

```vue
<template>
  <el-form ref="formRef" :model="form" :rules="rules">
    <el-form-item label="邮箱" prop="email">
      <el-input v-model="form.email" />
    </el-form-item>
    <el-button @click="submit">提交</el-button>
  </el-form>
</template>

<script setup>
import { ref } from 'vue'

const formRef = ref()
const form = reactive({
  email: ''
})
const rules = { /* 验证规则 */ }

async function submit() {
  const valid = await formRef.value.validate().catch(() => false)
  if (valid) {
    // 验证通过
    await api.submit(form)
  }
}
</script>
```

---

### 2. 验证失败提示

**推荐**: 使用 Element Plus 的默认错误提示 + 自定义消息

```typescript
const rules = {
  email: [
    {
      required: true,
      message: "请输入有效的邮箱地址",  // 清晰的错误消息
      trigger: "blur"
    }
  ]
}
```

**不推荐**: 只有验证器没有消息
```typescript
// ❌ 错误
const validateEmail = (rule, value, callback) => {
  if (!valid) {
    callback(new Error())  // 没有具体错误消息
  }
}
```

---

### 3. 异步验证

**推荐**: 检查唯一性时使用异步验证

```typescript
const validateUsername = async (rule, value, callback) => {
  if (!value) {
    callback(new Error('请输入用户名'))
    return
  }

  // 调用API检查用户名是否存在
  const exists = await api.checkUsernameExists(value)

  if (exists) {
    callback(new Error('用户名已存在'))
  } else {
    callback()
  }
}
```

---

### 4. 验证触发时机

**推荐**: 根据字段类型选择合适的触发方式

```typescript
const rules = {
  // 输入框: blur
  username: [{ required: true, trigger: "blur" }],

  // 选择器: change
  team_id: [{ required: true, trigger: "change" }],

  // 需要实时验证: blur & change
  password: [{ validator: validatePass, trigger: ["blur", "change"] }]
}
```

---

### 5. 表单清理

**推荐**: 组件卸载或弹窗关闭时清理表单

```vue
<script setup>
import { onUnmounted } from 'vue'

onUnmounted(() => {
  formRef.value?.resetFields()
})

function handleClose() {
  formRef.value?.resetFields()
  emit('update:modelValue', false)
}
</script>
```

---

### 6. 国际化

**推荐**: 所有验证消息使用 i18n

```typescript
import i18n from '@/language/lang'
const { t } = i18n.global

const rules = {
  email: [
    {
      required: true,
      message: t("EMAIL_REQUIRED"),  // 使用翻译Key
      trigger: "blur"
    }
  ]
}
```

**在语言文件中定义**:
```json
{
  "EMAIL_REQUIRED": "Please enter your email",
  "EMAIL_INVALID": "Please enter a valid email address"
}
```

---

### 7. 联动验证

**推荐**: 字段A变化时重新验证字段B

```typescript
const rules = {
  password: [
    { validator: validatePass, trigger: "blur" }
  ],
  confirmPassword: [
    { validator: validateConfirmPass, trigger: "blur" }
  ]
}

// 密码修改时重新验证确认密码
watch(() => form.password, () => {
  formRef.value?.validateField('confirmPassword')
})
```

---

## 常见问题

### Q1: 验证器为什么没触发？

**可能原因**:

1. **prop名称不匹配**
```vue
<!-- ❌ 错误 -->
<el-form-item prop="email">
  <el-input v-model="form.userEmail" />  <!-- prop和v-model不一致 -->
</el-form-item>

<!-- ✅ 正确 -->
<el-form-item Prop="userEmail">
  <el-input v-model="form.userEmail" />
</el-form-item>
```

2. **`trigger`设置错误**
```typescript
// ❌ 输入框用change可能不触发（失焦才触发验证）
{ trigger: "change" }

// ✅ 输入框应该用blur
{ trigger: "blur" }
```

3. **规则未绑定**
```vue
<!-- ❌ 忘记:rules -->
<el-form :model="form">

<!-- ✅ 正确 -->
<el-form :model="form" :rules="rules">
```

---

### Q2: 如何处理多语言错误消息？

**推荐做法**:

```typescript
import i18n from '@/language/lang'
const { t } = i18n.global

const rules = {
  email: [
    {
      required: true,
      message: t("VALIDATION.EMAIL_REQUIRED"),
      trigger: "blur"
    }
  ]
}
```

**在语言文件中**:
```json
{
  "VALIDATION": {
    "EMAIL_REQUIRED": "Please enter your email",
    "EMAIL_INVALID": "Please enter a valid email address"
  }
}
```

---

### Q3: 验证消息不显示怎么办？

**检查步骤**:

1. 确保 `hide-required-asterisk` 不是 `true`（隐藏了必填提示）
```vue
<el-form :hide-required-asterisk="false">
```

2. 检查 `label-width` 是否足够显示标签
```vue
<el-form label-width="120px">
```

3. 确认 `el-form-item` 有 `label`
```vue
<el-form-item label="邮箱" prop="email">
```

4. 验证 `message` 是否有值
```typescript
message: t("EMAIL_REQUIRED")  // 确保翻译Key存在
```

---

### Q4: 如何验证数组选择（多选）？

```typescript
const rules = {
  tags: [
    {
      type: 'array',
      required: true,
      message: '请选择至少一个标签',
      trigger: 'change'
    },
    {
      type: 'array',
      min: 1,
      max: 5,
      message: '选择1-5个标签',
      trigger: 'change'
    }
  ]
}
```

---

### Q5: 如何重置单个字段？

```typescript
// 重置单个字段
formRef.value?.resetFields(['email'])

// 重置所有字段
formRef.value?.resetFields()

// 清理验证状态（不重置值）
formRef.value?.clearValidate(['email'])
```

---

### Q6: 如何自定义错误提示样式？

CSS 样式覆盖:

```scss
/* 错误消息颜色 */
.el-form-item__error {
  color: var(--el-color-danger);
  font-size: 12px;
  line-height: 1.2;
}

/* 验证失败的输入框 */
.el-input.is-error .el-input__inner {
  border-color: var(--el-color-danger);
}

/* 验证成功的输入框 */
.el-input.is-success .el-input__inner {
  border-color: var(--el-color-success);
}
```

---

### Q7: 异步验证如何避免重复请求？

```typescript
let checkUsernameTimer = null

const validateUsername = (rule, value, callback) => {
  if (!value) {
    callback(new Error('请输入用户名'))
    return
  }

  // 清除之前的定时器
  clearTimeout(checkUsernameTimer)

  // 延迟500ms后检查，避免每次输入都请求
  checkUsernameTimer = setTimeout(async () => {
    try {
      const exists = await api.checkUsernameExists(value)
      if (exists) {
        callback(new Error('用户名已存在'))
      } else {
        callback()
      }
    } catch (error) {
      callback(new Error('检查失败，请重试'))
    }
  }, 500)
}
```

---

### Q8: 如何手动显示错误？

**场景**: API返回错误后显示在特定字段

```typescript
try {
  await api.submit(form)
} catch (error) {
  if (error.field === 'email') {
    // 手动设置邮箱字段错误
    formRef.value?.validateField('email', () => {
      const field = formRef.value?.fields.find(f => f.prop === 'email')
      if (field) {
        field.validateMessage = error.message
        field.validateState = 'error'
      }
    })
  }
}
```

---

## 验证性能优化

### 1. 延迟验证

对于输入框，延迟验证提升用户体验:

```typescript
let validateTimer = null

const validateUsername = (rule, value, callback) => {
  clearTimeout(validateTimer)
  validateTimer = setTimeout(() => {
    // 验证逻辑
    callback()
  }, 300)  // 延迟300ms验证
}
```

---

### 2. 避免重复验证

```typescript
let isValidating = false

const validateUnique = async (rule, value, callback) => {
  if (isValidating) return

  isValidating = true
  try {
    const exists = await api.check(value)
    if (exists) callback(new Error('已存在'))
    else callback()
  } finally {
    isValidating = false
  }
}
```

---

### 3. 使用 computed 缓存验证状态

```typescript
const usernameError = computed(() => {
  if (!form.username) return '请输入用户名'
  if (form.username.length < 3) return '用户名至少3个字符'
  if (!/^[a-zA-Z0-9_]+$/.test(form.username)) return '只允许字母、数字、下划线'
  return null
})

// 在模板中使用
<div class="error" v-if="usernameError">{{ usernameError }}</div>
```

---

## 相关文档

- [组件开发规范](./component_guide.md) - Vue组件开发最佳实践
- [正则表达式常量](../../src/utils/regexConstants.ts) - 所有正则表达式定义
- [错误处理指南](./error_handling_guide.md) - 表单提交错误处理
- [Composables指南](./composables_guide.md) - 可复用的验证逻辑

---

## 更新日志

### 2025-11-27
- **创建文档**: v1.0
- **验证文件**: 5个表单组件文件 + regexConstants.ts
- **主要功能**: 7个实际验证示例、15+个正则表达式、密码复杂性检验

---

**文档路径**: `dev_docs/form_validation_examples.md`
**最后更新**: 2025-11-27 20:30
**维护者**: Claude Sonnet 4.5
**代码版本**: v4.10.0

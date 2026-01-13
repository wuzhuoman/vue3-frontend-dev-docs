# [功能] CAVD 关联组件新增备注字段

## 📝 背景与问题描述

- **业务场景**
  - 在漏洞详情页的“关联组件（RELATED_LIBRARY）”区域，用户可以为当前漏洞新增或编辑关联组件（库）。
  - 业务侧希望在组件层面增加一个“备注（Remark）”字段，用于记录与当前漏洞或库相关的补充说明。
- **现状问题**
  - 新增组件弹窗中已有：组件名称、供应商、版本、发布时间、平台、特征文件、组件描述等字段，但缺少专门的备注字段。
  - 组件详情/编辑弹窗中同样只有描述字段，无法查看和编辑备注。
  - 后端已经在库（library）接口的请求体中增加了 `remark` 字段（与 `library_name / vendor / description` 同级），前端需要对齐。
- **影响范围**
  - 新增组件弹窗：`src/views/setting/components/AddNewComponents.vue`
  - 组件详情/编辑弹窗：`src/views/setting/components/CavdComponentDetail.vue`
  - 后端接口（仅做字段对齐，无前端封装改动）：
    - `POST /hub/management/libraries/`
    - `PATCH /hub/management/libraries/{library_id}/`
    - `GET /hub/management/libraries/{library_id}/`
    - `POST /hub/management/library_versions/`（保持仅负责版本创建和漏洞关联）

## 🎯 解决方案

### 技术思路

- 将备注设计为 **库（library）级字段**，与 `library_name / vendor / description` 相同层级：
  - 新增组件时，在创建库接口 `POST /hub/management/libraries/` 中写入 `remark`。
  - 编辑组件时，通过 `PATCH /hub/management/libraries/{library_id}/` 读写 `remark`。
  - 版本接口 `POST /hub/management/library_versions/` 不再携带 `remark`，避免语义混乱。
- 前端表单上，备注字段的交互和校验与“组件描述（description）”保持一致：
  - 使用 `el-input` 的多行文本模式，`maxlength=500`、`show-word-limit`。
  - 备注为**非必填**，仅限制最大长度 500。
- 国际化使用统一 key：`REMARK`，已在 `src/language/cn.js` 和 `src/language/en.js` 中存在：
  - 中文：`REMARK: "备注"`
  - 英文：`REMARK: "Remark"`

## 🔧 实施改动明细

### 1. 新增组件弹窗：`AddNewComponents.vue`

文件：`src/views/setting/components/AddNewComponents.vue`

#### 1.1 UI：新增备注字段

- 在“组件描述”下方新增备注字段表单项：
  - 标签：`t('REMARK')`
  - 绑定字段：`ruleForm.remark`
  - 组件：`el-input` 多行文本，`maxlength="500"`，`show-word-limit`，`type="textarea"`
- 位置：位于描述字段之后，保持与描述类似的布局和样式。

#### 1.2 表单模型和校验

- 扩展表单类型 `IruleForm`：

  ```ts
  interface IruleForm {
    name: String;
    vendor: String;
    version_number: String;
    release_date: any;
    platform: String;
    description: String;
    remark: String;          // ✅ 新增
    fileList: UploadUserFile[];
  }
  ```

- 初始化表单时新增默认值：

  ```ts
  const ruleForm = reactive<IruleForm>({
    name: "",
    vendor: "",
    version_number: "",
    release_date: "",
    platform: "",
    description: "",
    remark: "",             // ✅ 新增
    fileList: [],
  });
  ```

- 在表单校验规则 `rules` 中新增 `remark` 校验：

  ```ts
  remark: [
    {
      max: 500,
      message: t("ENTER_UP_TO_CHARACTERS", { count: 500 }),
      trigger: "blur",
    },
  ],
  ```

  - 说明：备注字段非必填，只做最大长度限制。

#### 1.3 提交时的请求参数调整

- 调用 `orgStore.postLibraries` 时，将 `remark` 一并写入库对象：

  ```ts
  orgStore
    .postLibraries({
      name: ruleForm.name,
      vendor: ruleForm.vendor,
      description: ruleForm.description,
      remark: ruleForm.remark,  // ✅ 新增
      platform:
        ruleForm.platform == "" ? "NOT_SPECIFIED" : ruleForm.platform,
    })
    .then((res) => {
      orgStore
        .postAddLibraryVersions({
          library_id: res.library_id,
          version_number: ruleForm.version_number,
          release_date: ruleForm.release_date,
          security_issue_id_list: [props.securityIssueId],
        })
        .then(async (res) => {
          library_version_id.value = res.lib_version_id;
          await uploadRef.value!.submit();
        });
    })
  ```

- 同时，从 `postAddLibraryVersions` 的 payload 中 **移除了 `remark` 字段**，保持版本接口仅承载版本和漏洞关联信息。

### 2. 组件详情/编辑弹窗：`CavdComponentDetail.vue`

文件：`src/views/setting/components/CavdComponentDetail.vue`

#### 2.1 UI：在详情/编辑页展示并编辑备注

- 在组件信息表单中，描述字段下方新增备注字段：
  - 编辑模式（`isEditor === true`）：
    - `el-input` 多行文本，`v-model="ruleForm.remark"`，`maxlength="500"`，`show-word-limit`。
  - 查看模式：
    - 以只读文本形式展示 `ruleForm.remark`。

#### 2.2 表单模型和校验

- 扩展 `ruleForm`：

  ```ts
  const ruleForm = reactive({
    library_name: "",
    vendor: "",
    platform: "",
    description: "",
    remark: "",     // ✅ 新增
  });
  ```

- 在 `rules` 中为 `remark` 增加长度校验：

  ```ts
  remark: [
    {
      max: 500,
      message: t("ENTER_UP_TO_CHARACTERS", { count: 500 }),
      trigger: "blur",
    },
  ],
  ```

#### 2.3 加载库详情时带出 remark

- `getLibraries` 时，从后台返回的库对象中读取 `remark` 并填充表单：

  ```ts
  const getLibraries = () => {
    return orgStore.getLibraries({ library_id: props.curId }).then((res) => {
      ruleForm.library_name = res.name || "";
      ruleForm.vendor = res.vendor || "";
      ruleForm.platform =
        res.platform == "NOT_SPECIFIED" ? "" : res.platform || "";
      ruleForm.description = res.description || "";
      ruleForm.remark = res.remark || "";   // ✅ 新增

      componentSource.value = res.source || "";
    });
  };
  ```

#### 2.4 保存组件信息时提交 remark

- `SAVE` 分支中，通过 `orgStore.patchLibraries` 将 remark 一并传给后端：

  ```ts
  orgStore
    .patchLibraries({
      library_id: props.curId,
      library_name: ruleForm.library_name,
      vendor: ruleForm.vendor,
      platform:
        ruleForm.platform == "" ? "NOT_SPECIFIED" : ruleForm.platform,
      description: ruleForm.description,
      remark: ruleForm.remark,     // ✅ 新增
    })
    .then(() => {
      ElMessage.success(t("UPDATED_SUCCESSFULLY"));
      emit("update:modelValue", false);
      emit("uploadList");
    });
  ```

### 3. 后端接口对齐说明（供查阅）

- 根据后端 API 文档，`PATCH /hub/management/libraries/{library_id}/` 的请求体示例包含：

  ```json
  {
    "library_name": "string",
    "vendor": "string",
    "language": "string",
    "description": "string",
    "remark": "string"
  }
  ```

- 本次前端改动中：
  - 已对齐字段：`library_name`（前端: `library_name`）、`vendor`、`description`、`remark`。
  - `language` 字段仍未在前端暴露/编辑，保持原有行为不变。如后续业务需要，可在同一组件中追加对应字段和交互。

## 📈 影响评估

- **功能影响**
  - 用户在“新增关联组件”时可以输入备注，备注会随库对象一起创建并持久化。
  - 用户在“组件详情/编辑”弹窗中可以查看并修改备注，保存后会同步更新到库对象。
- **数据模型影响**
  - remark 被明确归属于库（library），而非库版本（library_version）。
  - 版本接口 `POST /hub/management/library_versions/` 的签名不变，仅接收版本号、发布时间及漏洞关联 ID 列表等字段。
- **多品牌兼容性**
  - 本次改动仅涉及管理后台中 CAVD 关联组件的表单字段和请求 payload，不依赖品牌特有配置；
  - 不涉及 `src/config` 下的多品牌配置文件，预期对所有品牌行为一致。
- **风险点**
  - 后端必须在 `GET /hub/management/libraries/{library_id}/` 的返回体中包含 `remark` 字段，否则详情页备注显示为空字符串（不会报错，但数据不可见）。

## ✅ 验证计划

- 新增组件场景：
  1. 在某个漏洞详情页打开“关联组件”区域。
  2. 通过“新增组件”弹窗填写名称、供应商、版本、描述和备注，上传特征文件后保存。
  3. 确认列表中出现新组件。
  4. 点击组件名称进入详情弹窗，确认备注内容与新增时一致。

- 编辑组件场景：
  1. 在组件详情弹窗点击 `UPDATE` 进入编辑模式。
  2. 修改备注字段并保存。
  3. 重新打开详情弹窗，确认备注已更新。

- 回归检查：
  - 不填写备注时，新增和编辑流程应与历史行为一致，不影响表单校验和接口调用。
  - 文件上传、版本历史新增/删除等功能不受本次改动影响。

## 📚 参考资料

- `dev_docs/AI_Coding_Context.md` - 开发流程与方案文档规范
- `dev_docs/plans/README.md` - 方案文档模板与命名规范
- 后端接口文档：`PATCH /hub/management/libraries/{library_id}/` 请求样例（包含 `remark` 字段）
- 前端相关文件：
  - `src/views/setting/components/AddNewComponents.vue`
  - `src/views/setting/components/CavdComponentDetail.vue`
  - `src/api/org.ts`（`getLibraries`、`postLibraries`、`patchLibraries`、`postAddLibraryVersions`）

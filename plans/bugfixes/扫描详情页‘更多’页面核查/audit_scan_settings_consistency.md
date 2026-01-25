# 扫描引擎设置项一致性审计方案 (Audit Plan for Scan Settings Consistency)

## 1. 问题描述 (Problem Description)
目前系统中存在多个扫描引擎（SCA, SAST, CodeTrace, Deep License, Fuzzing, SAST Binary, IaC 等）。每个引擎在发起扫描前（`ScanModal.vue`）都有特定的配置项，这些配置项理应在扫描结果页面的“更多设置”（`ScanAdditionalSettings.vue`）中能够被复查，以确保用户可以追溯扫描时使用的具体参数。

**核心问题**：需要验证 `ScanModal.vue` 中定义的所有表单字段是否都已正确映射并展示在 `ScanAdditionalSettings.vue` 中，确保无遗漏、无错位。

---

## 2. 涉及文件路径 (Involved Files)

### 2.1 配置源 (Source of Truth - Settings Configuration)
- `src/views/project/ScanModal.vue`: 定义了各引擎的表单结构及提交时的 Payload 构建逻辑（`generateProjectPayload` 函数）。

### 2.2 数据转换层 (Data Mapper)
- `src/views/project/components/SAAS/SAASScanDetailHeader.vue`: 包含 `scanAttributes` 计算属性，负责从后端返回的 `scanInfo` 中提取数据并构建传递给 UI 的 `attributes` 对象。

### 2.3 渲染层 (Renderer - UI Display)
- `src/views/scan/components/ScanAdditionalSettings.vue`: 负责最终在 Popover 中渲染这些设置项。

---

## 3. 审计范围与任务清单 (Audit Scope & Task List)

请按照以下引擎逐一核对各字段的 **定义 -> 映射 -> 展示** 链条：

### 3.1 通用设置 (General Settings)
- [ ] **分支/版本 (Branch/Version)**: `branch` 或 `version_id`。
- [ ] **合规策略 (Compliance Policy)**: `compliance_policy_id`。

### 3.2 SCA (Software Composition Analysis)
- [ ] **扫描类型 (Scan Type)**: 源码/二进制 (`scan_type`)。
- [ ] **二进制设置 (SCA Binary Settings)**: GITLEAKS, SYFT 等勾选项 (`sca_binary_settings`)。
- [ ] **SBOM 特殊处理**: 当 provider 为 sbom 时，确认 engine_type 为 `SCA_META` 时的显示逻辑。

### 3.3 SAST & IaC (常规引擎)
- [ ] 核对基础参数（分支、策略）是否在这些引擎的结果页中正确展示。

### 3.4 CodeTrace
- [ ] **语言 (Language)**: 扫描时选择的语言列表。
- [ ] **阈值 (Matching Degree Threshold)**: `lib_version_filter_threshold`。
- [ ] **溯源策略**: `apply_component_tracing_strategy`。
- [ ] **路径设置 (Path Settings)**: 包含/排除模式 (`scanning_file_action`) 及其路径列表 (`scanning_files`)。

### 3.5 Deep License (深度许可证及冲突检测)
- [ ] **强制检查设置**: `DEEPLICENSE_IS_CHECK_ALL` (`perform_deep_license`)。
- [ ] **扫描深度**: `DEEPLICENSE_SCAN_DEEP` (`max_deep`)。
- [ ] **白名单规则**: `DEEPLICENSE_WHITE_LISTING_RULE` (`filename_white_list_regex`)。
- [ ] **项目许可证名称**: `PROJECT_LICENSE_NAME`。
- [ ] **许可证文本**: `LICENSE_TEXT` (及弹窗内容)。
- [ ] **排除路径列表**: `LICENSE_CONFLICT_EXCLUDED_PATHS` (`excluded_paths`)。

### 3.6 SAST Binary
- [ ] **工具/插件选项**: 如 GITLEAKS (`sast_binary_settings`)。

### 3.7 Fuzzing (模糊测试)
- [ ] **执行命令**: `execution_command`。
- [ ] **测试引擎**: `fuzzing_engine`。
- [ ] **持续时间**: `duration` (需确认小时/分钟的换算展示是否正确)。

---

## 4. 实施流程建议 (Execution Workflow)

1. **第一步：Payload 逆推**
   - 深入 `ScanModal.vue` 的 `generateProjectPayload` 函数，确定每个引擎发给后端的完整字段列表。
2. **第二步：一致性映射检查**
   - 在 `SAASScanDetailHeader.vue` 中搜索对应的字段名，确认它们被放入了 `attributes.options` 数组或其他特殊对象中。
   - 在 `ScanAdditionalSettings.vue` 中检查是否有对应的展示逻辑（复杂类型、i18n key、弹窗逻辑）。
3. **第三步：编写审查报告 (重要)**
   - **严禁直接修改代码**。
   - 若发现缺失、显示错误或样式不统一的情况，需详细记录在根目录的 `scan_settings_audit_report.md` 中。
   - 报告需包含：引擎名称、缺失字段、影响文件、建议修复方案。
4. **第四步：等待复审**
   - 完成报告后告知用户，等待用户审核报告，再决定下一步操作。

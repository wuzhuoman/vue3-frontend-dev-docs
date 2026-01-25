# 扫描引擎设置项一致性审计报告 (Scan Settings Consistency Audit Report)

## 1. 审计概述
本次审计通过对比 `ScanModal.vue` (提交层)、`SAASScanDetailHeader.vue` (数据转换层) 以及 `ScanAdditionalSettings.vue` (展现层)，核对了 SCA, SAST, CodeTrace, Deep License, Fuzzing 等引擎的扫描参数一致性。

## 2. 发现的主要问题

### 2.1 CodeTrace 引擎：缺失阈值设置 (Major)
*   **字段**: `lib_version_filter_threshold` (匹配度阈值/Matching Degree Threshold)
*   **现象**: 该字段在 `ScanModal.vue` 中定义并提交，但在结果页面的“更多设置”中完全缺失，用户无法查看扫描时设置的阈值。
*   **影响文件**: 
    1.  `src/views/project/components/SAAS/SAASScanDetailHeader.vue` (未映射该字段)
    2.  `src/views/scan/components/ScanAdditionalSettings.vue` (未渲染该字段)
*   **建议**: 在 `SAASScanDetailHeader.vue` 的 `scanAttributes` 中添加对 `code_trace_summary?.lib_version_filter_threshold` 的映射，并在渲染层展示为百分比。

### 2.2 Deep License 引擎：标签名称不一致 (Minor)
*   **字段**: `perform_deep_license`
*   **现象**: 
    *   在 `ScanModal.vue` 中使用的标签是 `DEEPLICENSE_IS_CHECK_ALL` (“是否检查所有文本文件”)。
    *   在 `SAASScanDetailHeader.vue` 中使用的标签是 `PERFORM_DEEP_LICENSE_AND_COPYRIGHT_SCAN` (“进行深度许可证和版权扫描”)。
*   **建议**: 统一使用 `DEEPLICENSE_IS_CHECK_ALL` 或更准备的术语。

### 2.3 CodeTrace 引擎：溯源策略被隐藏 (Minor)
*   **字段**: `apply_component_tracing_strategy`
*   **现象**: 在 `ScanAdditionalSettings.vue` 中，该字段被设置了 `v-show="false"`。虽然在 `ScanModal.vue` 中它也被隐藏了，但建议确认是否需要保留此逻辑，或者在有数据时应显示。

### 2.4 工具类 Bug：时长转换错误 (Bug)
*   **文件**: `src/utils/dateHelper.ts`
*   **函数**: `convertDurationInSec`
*   **现象**: 当秒数不为 0 时，代码 `formattedDuration = \`${formattedDuration} \${minutes} \${seconds}\`;` 会导致分钟数被重复拼接。
*   **建议**: 修复该逻辑，避免重复显示分钟信息。

## 3. 详细核对清单

| 引擎 | 属性 | payload 字段 | 映射状态 | 展示状态 | 备注 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **通用** | 分支/版本 | `branch`/`version_id` | OK | OK | 在 Header 直观展示 |
| **通用** | 合规策略 | `compliance_policy_id` | OK | OK | |
| **SCA** | 扫描类型 | `scan_type` | OK | OK | |
| **SCA** | 二进制设置 | `sca_binary_settings` | OK | OK | 仅二进制模式显示 |
| **CodeTrace** | 语言 | `languages` | OK | OK | |
| **CodeTrace** | 阈值 | `lib_version_filter_threshold` | **MISSING** | **MISSING** | 关键缺失 |
| **CodeTrace** | 包含/排除路径 | `scanning_files` | OK | OK | |
| **Deep License**| 是否检查所有 | `perform_deep_license` | OK | OK | 标签不一致 |
| **Deep License**| 扫描深度 | `max_deep` | OK | OK | |
| **Deep License**| 白名单 | `filename_white_list_regex` | OK | OK | |
| **Deep License**| 项目许可证名 | `project_license_name` | OK | OK | |
| **Deep License**| 排除路径列表 | `excluded_paths` | OK | OK | 已正确集成 |
| **SAST Binary** | 二进制设置 | `sast_binary_settings` | OK | OK | |
| **Fuzzing** | 执行命令 | `execution_command` | OK | OK | |
| **Fuzzing** | 持续时间 | `duration` | OK | OK | 存在转换 Bug |

## 4. 下一步建议
1.  确认 `lib_version_filter_threshold` 的展示样式。
2.  修复 `dateHelper.ts` 中的拼写错误。
3.  统一 Deep License 相关的 i18n label。

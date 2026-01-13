# 深度许可证扫描状态更新异常修复报告

**日期**: 2025-11-28
**作者**: Antigravity (AI Assistant)
**状态**: 已修复

## 1. 背景：问题起源

本次多引擎扫描更新问题的排查，源于对“深度许可证扫描结果不更新”的修复。

### 1.1 初始问题

用户反馈深度许可证扫描完成后，项目卡片状态不更新。排查发现 `SAASProjects.vue` 的 `watch` 函数中完全缺失对 `deep_license` 类型的处理逻辑。

### 1.2 前置修复

为了解决此问题，我们在 `watch` 函数中补充了 `deep_license` 的处理分支。

### 1.3 引发的新问题

该修复成功解决了 Deep License 的状态更新，但也暴露了深层次的并发问题。由于 Deep License 引擎通常最后完成，它的加入使得原本就存在的“防抖导致消息丢失”问题（SCA 与 CodeTrace 竞争）变得更加明显和不可接受，从而引发了本次的全面排查与修复。

## 2. 问题描述

在执行“深度许可证扫描”时，系统会同时启动三个扫描引擎（SCA、CodeTrace、Deep License）。用户反馈在扫描完成后，只有 Deep License 的状态正确更新为“已完成”并显示结果，而 SCA 和 CodeTrace 的进度条一直卡在 95% 或 99%，且不显示最终结果。手动刷新页面后状态恢复正常。

## 3. 根本原因分析

经过深入排查和用户反馈验证，确认导致该问题的**唯一根本原因**是：

### 3.1 防抖逻辑导致消息丢失 (Race Condition)

- **现象**: SCA 和 CodeTrace 的扫描通常在极短的时间间隔内（约 0.6秒）相继完成。
- **机制**: `SAASProjects.vue` 中的 `watch` 函数使用了 500ms 的 `setTimeout` 进行防抖处理。
- **后果**: 当 SCA 的完成消息到达触发 `watch` 后，计时器启动。紧接着（<500ms）CodeTrace 的完成消息到达，触发 `watch` 并清除了前一个计时器。导致 SCA 的更新任务被取消（丢弃），只有最后到达的消息（CodeTrace 或 Deep License）被处理。

> **修正说明**: 最初推测“Scan ID 不匹配”也是原因之一。但经确认，扫描过程中进度条能正常更新（Running 状态），说明 Scan ID 在大部分时间内是匹配的。因此，“Scan ID 不匹配”不是导致状态卡在 95% 的直接原因，防抖导致的竞争条件才是核心主因。

## 4. 修复方案

针对上述原因，实施了以下修复措施：

### 4.1 核心修复：移除关键状态的防抖

为了解决消息丢失问题，重构了 `watch` 逻辑：

- **Finished 状态**: 当收到 `scan_status == "finished"` 的消息时，**立即执行**更新逻辑，不进行任何防抖等待。
- **Running 状态**: 保留防抖逻辑，以避免进度条频繁刷新带来的性能开销。

### 4.2 增强健壮性：引入特征匹配机制 (Feature Matching)

虽然 Scan ID 匹配正常，但为了防止未来可能出现的 ID 同步延迟问题，我们作为防御性措施引入了特征匹配：

- 在 `finished` 状态下，如果 ID 匹配失败，尝试通过 WebSocket 消息中的 `summary` 特征字段（如 `code_trace_issue_summary`）来识别引擎并强制更新。

## 5. 涉及文件与改动

**文件路径**: `\vue3-frontend\src\views\project\SAASProjects.vue`

**主要改动代码片段**:

```javascript
// 1. 提取更新逻辑为独立函数
const handleProjectStatusUpdate = () => {
  // ... (包含特征匹配和状态更新的核心逻辑)
};

// 2. 重构 watch 函数
let timer = null;
watch(
  () => projectStore.projectSaasProgressItem,
  () => {
    // 无论什么状态，先清除之前的 timer，避免状态回滚或冲突
    if (timer) {
      clearTimeout(timer);
      timer = null;
    }

    const msg = projectStore.projectSaasProgressItem;
    // 关键修复：如果是 finished 状态，立即执行，不防抖
    if (msg?.scan_status === "finished") {
      handleProjectStatusUpdate();
    } else {
      // 其他状态（如 running）保留防抖
      timer = setTimeout(handleProjectStatusUpdate, 500);
    }
  },
  { deep: true },
);
```

## 6. 排查与修复过程记录

为了给今后的类似问题提供参考，以下记录了本次修复的完整排查路径、失败尝试及最终突破口。

### 6.1 初始分析阶段

- **数据收集**: 获取了 WebSocket 通讯记录。
- **关键发现**: 所有三个引擎（SCA, CodeTrace, Deep License）推送的 `scan_type` 均为 `"source_code"`。
- **初步结论**: 原有的基于 `eng.name` 的判断逻辑在多引擎场景下失效，因为无法仅凭 `scan_type` 区分引擎。

### 6.2 第一次尝试 (失败)

- **方案**: 实施“特征匹配”策略。即在 `scan_id` 不匹配时，通过检查 `summary` 中的特定字段（如 `code_trace_issue_summary`）来识别引擎并强制更新。
- **结果**: 用户测试反馈，Deep License 状态更新成功，但 SCA 和 CodeTrace 仍然卡在 95% 进度。
- **反思**: 特征匹配解决了 Deep License 的问题（它是最后完成的），但未能解决前两个引擎的问题。这暗示存在其他机制导致消息丢失。

### 6.3 第二次分析 (突破口)

- **深入排查**: 重新审视 WebSocket 日志的时间戳和前端代码逻辑。
- **关键发现**:
  - SCA 完成时间: `12:28:10`
  - CodeTrace 完成时间: `12:28:10` (仅相差约 0.6秒)
  - 前端代码: `watch` 函数中包含 `setTimeout(..., 500)` 的防抖逻辑。
- **根本原因确认**: **竞争条件 (Race Condition)**。由于 SCA 和 CodeTrace 几乎同时完成，SCA 的处理逻辑还在防抖等待期内，就被紧随其后的 CodeTrace 消息触发的 `clearTimeout` 取消了。因此，SCA 的状态永远无法更新。

### 6.4 最终修复

- **策略调整**: 在保留“特征匹配”以解决 ID 问题的前提下，**移除了 `finished` 状态的防抖逻辑**。
- **逻辑**: 只要收到“完成”信号，立即执行更新，确保每个引擎的完成事件都能被独立、完整地处理。
- **验证**: 用户测试通过，所有三个引擎状态均能正确更新。

## 7. 项目设计洞察与建议

在排查过程中，对项目在扫描状态管理方面的设计有以下发现：

### 7.1 WebSocket 消息处理机制

- **现状**: `DashboardLayout.vue` 接收 WebSocket 消息并直接更新 Pinia Store (`projectStore.projectSaasProgressItem`)。
- **风险**: 这种单对象覆盖模式在高并发推送下容易丢失中间状态。虽然 Vue 的响应式系统能处理大部分情况，但配合防抖逻辑时极易出错。
- **建议**: 未来考虑在 Store 中维护一个消息队列，或者使用事件总线（Event Bus）分发关键事件，确保所有 `finished` 消息都能被消费。

### 7.2 多引擎数据结构

- **现状**: 所有源代码扫描（SCA, SAST, CodeTrace, Deep License）的 `scan_type` 均为 `source_code`，缺乏明确的引擎标识字段。
- **风险**: 前端必须依赖 `summary` 内部结构的差异来区分引擎，这增加了耦合度和维护成本。
- **建议**: 建议后端在 WebSocket 消息中增加明确的 `engine_type` 字段（如 `SCA`, `CODETRACE`, `DEEP_LICENSE`），从而简化前端的匹配逻辑。

### 7.3 前端状态同步

- **现状**: 前端列表的 `scan_id` 更新依赖于 WebSocket 消息或手动刷新，存在滞后性。
- **建议**: 在启动扫描的 API 响应中，后端应尽可能返回新生成的 `scan_id` 列表，前端立即更新本地状态，而不是等待 WebSocket 推送。

## 8. 技术附录

为了便于后续维护和排查，以下记录了相关的技术细节和数据结构。

### 8.1 WebSocket 通讯数据结构

根据 `webscoket通讯记录.txt`，多引擎扫描的 WebSocket 消息具有以下特征：

- **通用字段**: 所有引擎的 `scan_type` 均为 `"source_code"`，无法直接区分。
- **Deep License 数据**: 包含在 `summary.deep_license_summary` 中。
- **CodeTrace 数据**: 包含在 `summary.code_trace_issue_summary` 中。
- **SCA 数据**: 主要体现为 `summary.lib_ver_count`。

**Deep License 消息示例**:

```json
{
  "scan_id": 87574,
  "scan_status": "finished",
  "scan_type": "source_code",
  "summary": {
    "deep_license_summary": {
      "license_count": 10,
      "lib_license_count": 18,
      "copyright_count": 23,
      "license_conflict_count": 23,
      "perform_deep_license": false
    }
  }
}
```

### 8.2 关键数据流

扫描状态的更新经过了以下完整的链路：

1.  **WebSocket 接收**: `src/views/layouts/DashboardLayout.vue` 监听 `scan_status_update` 事件。
2.  **Store 更新**: 调用 `projectStore.updateSAASProjectList(data)`。
3.  **State 存储**: 数据被存储在 `src/stores/project/index.ts` 的 `state.projectSaasProgressItem` 中。
4.  **UI 响应**: `src/views/project/SAASProjects.vue` 通过 `watch` 监听 `projectStore.projectSaasProgressItem` 的变化并更新 UI。

### 8.3 深度许可证初始修复代码

在解决“深度许可证状态不更新”的初始阶段（即引发本次并发问题的修复），我们在 `SAASProjects.vue` 中添加了如下逻辑：

```javascript
} else if (eng.name.toLowerCase().includes("deep_license")) {
  // 从 summary.deep_license_summary 中获取统计数据
  const deepLicenseSummary = projectStore.projectSaasProgressItem?.summary?.deep_license_summary || {};

  eng.license_count = deepLicenseSummary.license_count || 0;
  eng.lib_license_count = deepLicenseSummary.lib_license_count || 0;
  eng.copyright_count = deepLicenseSummary.copyright_count || 0;
  eng.license_conflict_count = deepLicenseSummary.license_conflict_count || 0;
}
```

正是这段逻辑的加入，使得 Deep License 引擎参与到了状态更新的竞争中，从而暴露了原有的防抖缺陷。
